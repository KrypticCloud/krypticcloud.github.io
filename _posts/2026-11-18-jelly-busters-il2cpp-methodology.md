---
layout: post
title: "How I approach static analysis of a Unity IL2CPP Android game"
date: 2026-11-18 08:20:00 -0600
tags: [reverse-engineering, android, unity, il2cpp, mobile-security, methodology]
---


**Summary.** I did a read-only static analysis of a mobile game I'd installed normally, to learn how Unity games are put together on Android. I did this in October 2026. It turned out to be a Unity 2022.3 build compiled with IL2CPP, shipped for arm64 only. This post is about method, not about the game: how to recognize an IL2CPP build, what's recoverable from one and what isn't, and why runtime observation of an arm64-only app falls apart in the Android emulator on an x86_64 PC. There's no dumped code, no class or method names, no offsets, no assets and no server endpoints here, and nothing that would help anyone cheat.

## Scope and rules I set myself

- I installed the game the normal way, on my own device.
- Read-only. I didn't modify the game, run a patched client, or make in-app purchase or leaderboard requests of any kind.
- No requests to the game's servers beyond what normal play sends.
- Nothing extracted gets published. My structure notes stay on my own laptop.

## Step 1: get the APK

I pulled the installed APK off my phone over ADB (Android Debug Bridge), using wireless debugging from PowerShell on my Windows laptop. No root needed:

```powershell
adb connect <phone-ip>:<port>
adb shell dumpsys package <package.name>   # version, ABI, installer, split list
adb shell pm path <package.name>           # one path per split
adb pull <each-path-from-above>
```

The game shipped as two split APKs: a base APK and an arm64 config split that holds the native libraries. An APK is a ZIP file, so the first look is just listing what's inside. Windows' built-in `tar` reads ZIPs, which saves installing anything:

```powershell
Get-ChildItem *.apk | ForEach-Object { tar -tf $_.FullName } |
  Select-String 'lib/|global-metadata|bin/Data|libunity|libil2cpp'
```

After that I copied the APKs into WSL2 (on its own ext4 filesystem, not the Windows mount, which is painfully slow for thousands of small files) and recorded SHA-256 hashes before extracting anything.

## Step 2: identify the engine and the build type

Two things in the APK give away a Unity IL2CPP build:

- `lib/arm64-v8a/libil2cpp.so`: the game's C# code, compiled to native ARM64 code
- `assets/bin/Data/Managed/Metadata/global-metadata.dat`: a metadata file describing the game's types and methods

Unity can build a game two ways. With Mono, the C# ships as .NET assemblies (DLLs), which decompile back to readable C# almost directly. With IL2CPP, Unity converts the compiled C# to C++ and compiles that to native code. That's much harder to read, and it's a common choice for mobile builds.

The `lib/` folder held only `arm64-v8a`, with no 32-bit ARM or x86 libraries. That detail matters in step 4. I read the Unity version (2022.3) from the header of the packed asset archive, `assets/bin/Data/data.unity3d`, and cross-checked it against strings in `libunity.so`. The cross-check mattered: the library also held a much older version string baked in from the engine's own build, so grabbing the first version-looking string would have been wrong.

One more check before going further: the first bytes of `global-metadata.dat`. An unprotected file starts with the magic bytes `af 1b b1 fa` followed by a version number. This one was in the clear (metadata version 31), so no runtime memory dump was needed to recover it.

## Step 3: recover structure from the metadata

The native library alone is a large stripped binary. The metadata file is what makes it navigable. It holds the names and signatures of types, fields and methods, because the Unity runtime needs them for things like reflection. A metadata dumper reads `libil2cpp.so` and `global-metadata.dat` together and rebuilds the game's type structure: class names, method signatures, and where each method's compiled code lives in the library.

I used Il2CppInspectorRedux, a free and open-source dumper that runs on the .NET 8 SDK. It produced a C# type dump, a C header of the game's types, a Ghidra script that labels functions in the disassembler, and a JSON export of the metadata.

What you get back is structure, not source. You see that a method exists, what it takes and returns, and where it is. You don't get its body as C#. To learn what a method does, you go to its address in a disassembler and read ARM64. The dump is a map that tells you where to look.

One trap: the build's list of assemblies (`ScriptingAssemblies.json`) names everything the project *could* contain. Several assemblies with interesting-sounding names turned out to hold zero types in the dump, because Unity's code stripping removed everything nothing referenced. The type dump tells you what's actually there; the assembly list doesn't.

I wrote my findings up as structure notes. In general terms: a dependency-injection container wires the app together, game state lives in an entity-component-system design with a small set of data components (about 30), and around that sit level loading and difficulty logic, the economy and live-ops features, a remote-config bootstrap and the network layer. Those notes stay private.

## Step 4: the emulator snag for runtime observation

Static analysis tells you what code exists. Watching the app run tells you which code matters. For that I wanted the game in an emulator, and hit a practical problem.

Current Android Emulator builds no longer offer arm64 system images on x86_64 host machines. Since this game ships arm64-only, there's no matching image to run it natively. The workaround is an x86_64 Android Virtual Device (AVD) with ARM translation, which translates the app's ARM64 instructions to x86_64 as it runs. It works for many apps, but it's slower, and it means the emulator isn't running exactly the same instructions as a real phone.

I used an Android 14 (API 34) Google APIs x86_64 image. The system property `ro.dalvik.vm.isa.arm64=x86_64` confirms translation is active, and the device advertises both `x86_64` and `arm64-v8a`. The game installs and launches there, so for watching the app's own network traffic this setup is fine.

Hooking the game's code is where it breaks. I attached Frida (17.24.3, frida-server on the emulator) to the running game. It could see the translator's own x86_64 libraries but not the game's arm64 ones: those are loaded by the translator's own loader, and none of `libil2cpp.so`'s memory mappings are executable as native x86_64 code. So function hooks can't work under translation. Reading process memory still does, since reading bytes doesn't mean executing ARM code. The game's anti-cheat didn't react to the attach. Hooking properly needs real arm64: a rooted phone, or arm64 Android running natively on ARM hardware.

That's where the project stands.

## What this means for defenders and developers

1. **IL2CPP is not protection.** It makes reading code harder than with Mono, but the metadata file hands over a map of the whole codebase. Treat anything in the client as readable.
2. **Never trust the client.** Purchases, scores, currency and progress must be validated on the server. Any check that only happens in the app can be studied and eventually bypassed.
3. **Metadata protection raises the cost, not the ceiling.** Encrypting or obfuscating `global-metadata.dat` slows a dumper down, but the runtime has to decrypt it to run, so a determined analyst can still get it. It's worth doing as a speed bump, not as a security boundary.
4. **Don't ship secrets in the app.** API keys, signing secrets and internal endpoints compiled into a mobile build should be treated as already public. This build's dump and assets held ad-SDK app keys and cloud-config values in plain text, which is why my notes record key names and lengths only, never the values.

## Lessons learned

- **Identify before you analyze.** Two file paths in the APK told me which tools and approach would work.
- **Know what each technique gives you.** A metadata dump gives you structure, not behavior.
- **Check where your code actually runs.** Under ARM translation, the game's code never runs as native code, so tools built to hook native code have nothing to attach to.
- **Set your rules first.** Writing down read-only, no modified client and no publishing extracted material before starting kept the project clearly on the right side of the line.
