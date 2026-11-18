---
layout: post
title: "Unlocking drives from my old home recording system, part 1: what's actually locking them"
date: 2026-11-18 08:30:00 -0600
tags: [hardware, reverse-engineering, ata-security, uart, iot, lab-notes]
---

**Summary.** I have two 2TB hard drives from my own old home security recording system, and neither one will give up its data in a normal PC. They aren't encrypted in the usual sense. The recording board locked them with ATA Security, a password feature built into the drive's firmware. This is part 1 of an in-progress project: how I figured out what kind of lock it is, why the usual shortcut probably won't work, and the plan to get the password out of the board itself over a serial console. The drives are still locked. There are no results yet, and this post doesn't pretend otherwise.

## Scope and ownership

The drives and the board came out of the home security recording system that was installed in my own house. I took it apart and kept the board and both drives. Nothing here touches anyone else's hardware, anyone's cloud service or account. I'm not publishing firmware dumps or anything that could be used against the vendor's service. Since I took it apart, the board has only ever been on bench power: no network cable, no Wi-Fi, nothing that lets it reach the internet.

## The hardware

- Two WD Purple drives, model WD20EJRX, 2TB each. WD Purple is Western Digital's line for surveillance recorders.
- The recording board they came from: a Space Monkey v2 (SM2) board, from a Vivint home system. Its main chip is an NXP i.MX6 DualLite, an ARM system-on-chip common in embedded Linux devices.

## What kind of lock is this?

The first question was whether I was dealing with encrypted video or a locked drive. They need completely different approaches. Encrypted video means the bytes on the platter are scrambled and a key lives somewhere else. A locked drive means the drive's own firmware refuses to read or write until it gets a password.

This is the second kind. The drives report that ATA Security is enabled and locked. ATA Security is part of the ATA command set that SATA drives implement. A host sets a user password with `SECURITY SET PASSWORD`, and after a power cycle the drive rejects data access until the host sends `SECURITY UNLOCK` with the right password. The board sets this password itself and unlocks the drives at every boot. It isn't a password anyone in the household chose.

I found this by accident in October 2026. With the drives connected to my desktop, the BIOS stopped at boot and asked for a drive password. I then read each drive's security state from Windows with smartmontools (`smartctl -i -g security`). Both drives reported security enabled and locked, at security level MAX.

## Why the easy route probably won't work

ATA Security has two passwords. The user password is the one the board set. There's also a master password, which manufacturers ship with a factory default so a locked drive can at least be erased and reused.

Drives report a **master password identifier** (sometimes called the revision code) in their IDENTIFY data. A factory default value usually means the master password was never changed. These drives report `0x0001`, which isn't the default. That suggests whoever set up the drives also changed the master password, so a published factory master password most likely won't work, not even for an erase.

Two more points that shape the plan:

- **An erase isn't an unlock.** Even when a master password works, in the drive's maximum security level it only permits `SECURITY ERASE UNIT`, which wipes the drive. That gets the hardware back, but not the data. In my case that's fine: my goal is to reuse the drives, not to recover the old footage. But an erase still needs a master password that works, and the `0x0001` identifier suggests I don't have one.
- **Guessing is a dead end.** The drive limits failed unlock attempts. After a handful of wrong passwords it refuses more until it's power cycled, so brute forcing over SATA is too slow to matter.

## The real lead: the board already knows the password

The recording board unlocks these drives every time it boots. So the password, or the way to derive it, has to be on the board: stored in flash, or computed from something like the drive's serial number. If I can watch the board boot, I can learn how it handles the drives.

That's where a serial console comes in. Embedded Linux boards often expose a UART (universal asynchronous receiver-transmitter): a TX, RX and ground connection that prints boot logs and sometimes drops to a shell. I'm working toward that now:

1. **Power and bench setup.** The board powers on and runs on the bench.
2. **Rule out the obvious.** The board has a micro USB port, and it would have been convenient if it were a console. It isn't. On this chip it's most likely the USB OTG and serial download port. I compared Windows' list of devices before and after plugging the running board in, and nothing new appeared.
3. **Find the UART.** There are two candidate headers. On the six-pin header, a continuity check against a known ground point found a ground pin, and two other pins sit at about 3.2V, which points to 3.3V logic. That rules out needing a 1.8V level shifter. On the other header I couldn't find a ground at all, so I set it aside.
4. **Listen before talking.** My first try at a bridge was a Flipper Zero in USB-UART mode, wired listen-only: ground plus the Flipper's RX, with its TX left disconnected so nothing could drive the board's pins. That stalled on the tool, not the board. The custom firmware on my Flipper wouldn't actually switch the USB channel to the bridge, so all I ever saw was the Flipper's own command line. The next try is an ESP32 dev board running a small serial passthrough sketch, or a cheap 3.3V USB-to-serial adapter.
5. **Find the baud rate and read the boot log.** 115200 is the first guess.

Steps 4 and 5 are where I am now. I'll report what the boot log shows in part 2.

## What this means for defenders

1. **Old recorders keep footage.** A drive from a home security system can hold months of video of the inside and outside of a house. Before you sell, recycle or throw out a recorder or its drives, wipe them, or physically destroy them if you can't.
2. **ATA Security protects against casual reuse, not determined people.** It stops someone from plugging the drive into a PC and browsing it. It doesn't stop someone who has the board that unlocks it, or a vendor tool. If you build devices, the security of drive locking comes down to where the password lives and whether it's unique per device.
3. **Debug interfaces are attack surface.** If a UART on a shipped board gives a root shell, anyone with physical access and a cheap adapter has the same access I'm trying to get. Manufacturers should disable or authenticate debug consoles on production hardware.

## Lessons learned

- **Name the lock before picking it.** Encryption and ATA Security look the same from the outside ("I can't read the drive"), but the fixes have nothing in common.
- **Read the identifier before trying defaults.** The master password identifier told me the common shortcut was likely closed before I spent time on it.
- **The device that locks the data is usually the best way back in.**
