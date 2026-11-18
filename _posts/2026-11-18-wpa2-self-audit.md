---
layout: post
title: "Auditing my own WPA2 network with a Flipper Zero, Marauder and hashcat"
date: 2026-11-18 14:00:00 -0600
tags: [wifi, wpa2, hashcat, flipper-zero, blue-team]
---

**Summary.** I wanted to understand the WPA2-PSK handshake attack well enough to check an automated version of it later, so in August 2026 I ran the whole thing by hand against my own home network. I flashed ESP32 Marauder onto a Flipper Zero WiFi dev board, captured my network's 4-way handshake with a targeted deauthentication, converted the capture with hcxpcapngtool, and ran it through hashcat mode 22000 on a laptop iGPU. rockyou.txt ran to exhaustion without cracking it. This post covers how each step works, why the attack failed, and what a defender should take from it.

## Scope and legality

Everything here was done against a network I own, using devices I own, with my own authorization. I didn't capture, deauthenticate or test anyone else's network or clients. Capturing handshakes from networks you don't control, or knocking other people's devices offline, is illegal in many places, so don't point any of this at something that isn't yours.

I've also scrubbed everything that identifies my network. There's no SSID, BSSID, MAC address, internal address or router model in this post, and I'm not publishing the capture or the hash file. The cracking section explains why the hash file matters.

## Setup

- Flipper Zero running RogueMaster firmware
- Flipper Zero WiFi dev board (ESP32-S2, 2.4 GHz only)
- ESP32 Marauder v1.15.0, flashed from justcallmekoko's official installer (I checked the version after flashing)
- The Marauder companion app on the Flipper to drive it
- ASUS ROG Flow Z13: Kali in WSL2 for hcxpcapngtool, and hashcat on the Radeon 8060S iGPU through OpenCL
- rockyou.txt as the wordlist

The flash didn't go cleanly at first. After flashing, the board kept failing to boot, and I spent a while assuming the hardware was bad. It wasn't. A full erase-flash cycle, wiping the chip completely and then installing Marauder fresh, fixed it. I did the erase through the installer; esptool does the same job from a terminal if you prefer:

```bash
esptool.py --chip esp32s2 --port <port> erase_flash
```

Because the ESP32-S2 only does 2.4 GHz, that's the band I tested.

## Capture

The capture had two parts: a passive scan, then a targeted deauthentication to force a fresh handshake.

The passive scan just listens for beacon frames and builds a list of access points in range, along with their channels. Nothing gets transmitted at anyone. I used it to find my own AP and its channel. A scan like this also picks up the neighbors' networks; I only noted my own, and none of that list appears here.

Next, the handshake. When a client joins a WPA2-PSK network, it and the AP run the 4-way handshake. Messages 1 and 2 carry the AP's nonce (ANonce), the client's nonce (SNonce), and a message integrity code (MIC) computed with a key derived from the passphrase. That's everything an attacker needs to start guessing offline. Nobody has to send the passphrase over the air for this to work.

You can wait for a client to reconnect on its own, but that's slow. The standard shortcut is to send deauthentication frames spoofed from the AP. The client drops, reconnects within seconds, and the capture catches the new handshake. This works because WPA2 without Protected Management Frames doesn't authenticate management frames, so the client has no way to tell a spoofed deauth from a real one.

On the Flipper I drove Marauder through its companion app: scan for access points, select my own, then sniff for the handshake while sending deauths against it. The equivalent Marauder CLI commands look roughly like this:

```text
scanap
stopscan
list -a
select -a <index>
sniffpmkid -c <channel> -d -l
```

The capture was saved to the Flipper's SD card as a pcap file.

## Cracking

First, convert the capture into hashcat's format:

```bash
# Z13 — WSL2 (Kali)
hcxpcapngtool -o <hashfile>.hc22000 <capture.pcap>
```

hcxpcapngtool prints a summary of what it found, including how many usable EAPOL message pairs it could pull out. One good pair is all hashcat needs.

The output is one line per handshake, in this shape:

```text
WPA*02*<MIC>*<MAC_AP>*<MAC_CLIENT>*<ESSID>*<ANONCE>*<EAPOL>*<MESSAGEPAIR>
```

That line is why I'm not publishing the hash file, even though it isn't the password. It contains the AP's MAC address, the client's MAC address, and the network name (in hex). Posting it would identify the network, and it would hand anyone a free shot at cracking it.

Then hashcat:

```bash
hashcat -I                                   # list OpenCL devices, confirm the iGPU shows up
hashcat -m 22000 -a 0 -D 2 <hashfile>.hc22000 rockyou.txt
```

`-m 22000` is the WPA-PBKDF2-PMKID+EAPOL mode, `-a 0` is a straight dictionary attack, and `-D 2` restricts hashcat to GPU devices.

For every candidate password, hashcat runs PBKDF2-HMAC-SHA1 with 4,096 iterations, using the SSID as the salt, to get the pairwise master key (PMK). From the PMK, the two nonces and the two MAC addresses, it derives the pairwise transient key (PTK), computes the MIC, and compares it with the captured one. The 4,096 iterations make each guess deliberately expensive, and salting with the SSID means precomputed tables only help against common network names.

## Result

hashcat worked through all of rockyou.txt, about 14.3 million candidates, and reported the run as Exhausted. No crack. My passphrase isn't a common dictionary word, so a list of leaked passwords had nothing to match.

To be precise about what that proves: it shows my passphrase isn't in rockyou.txt. It doesn't show it can't be cracked. Rules, bigger lists and masks are the next steps, and that's what the follow-up is for.

## Defender takeaways

1. **For WPA2-PSK, the passphrase is everything.** Anyone in radio range can collect the handshake, and after that the attack runs offline where you can't see it. A long, random passphrase that isn't built from dictionary words is the control that actually decides the outcome.
2. **Use WPA3-SAE where your devices support it.** SAE (Simultaneous Authentication of Equals) doesn't expose anything that can be guessed offline; each guess has to go through a live exchange with the AP. Be careful with WPA2/WPA3 transition mode, though. It still accepts WPA2 clients, so the attack above still works against them.
3. **Turn on Protected Management Frames (802.11w) if you can.** PMF authenticates deauthentication and disassociation frames, so clients that support it ignore spoofed deauths. It's mandatory in WPA3 and optional in WPA2. PMF removes the forced reconnect, not the attack: an attacker can still wait for a natural reconnect, or go after the PMKID, which the same mode 22000 format covers and which needs no client at all. PMF is not a substitute for a strong passphrase.
4. **Deauth attacks are noisy.** A burst of deauth frames claiming to come from your AP, followed by several clients dropping and reconnecting at once, is a clear signal. Wireless intrusion detection tools such as Kismet can alert on deauth floods. On a home network without monitoring, the symptom is devices dropping off WiFi for no obvious reason.

## What's next

Doing this by hand was the point. Next I'm writing an escalating self-audit script that runs a dictionary first, then rules, then larger wordlists, then bounded masks, with the cracking moved to my desktop's RTX 4070 Super. Having done every step manually, I know what each stage should produce, which means I can check the script's output instead of trusting it.

## Lessons learned

- **Erase before blaming hardware.** The boot failure that looked like a dead board was fixed by a full flash erase.
- **The hash file is sensitive.** It isn't the password, but it carries the network name and MAC addresses, and anyone who has it can attack the passphrase offline.
- **"Not cracked" is a narrow result.** A clean rockyou.txt run tells you one thing about the passphrase, so it's worth saying exactly what was tested.
- **Manual first, automation second.** I'd rather understand each step than trust a tool that hides them.

