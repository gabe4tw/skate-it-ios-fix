# skate-it-ios-fix

Get **Skate It** (EA, 2010, `com.ea.skateit.inc`, v1.0.70, 32-bit armv6)
running on modern iPhones.

This is a copy of [Applesauce](https://github.com/johnny901901901/Applesauce),
the iOS port of [touchHLE](https://github.com/touchHLE/touchHLE) /
[HyperHLE](https://github.com/HyperHLE/HyperHLE), with fixes the game needs and
a GitHub Actions workflow that builds the iPhone app without a Mac. The
Applesauce README is kept as [APPLESAUCE_README.md](APPLESAUCE_README.md).

Based on Applesauce `ios-host` at `c75278b`. This project isn't affiliated with
or endorsed by EA, Applesauce, HyperHLE or touchHLE, and includes no games.

## What's fixed

| Symptom | Cause | Fix |
| --- | --- | --- |
| Garbled text (`ÿÿÿlf`), skater spawns half inside the ground and bails, board renders as a black blob, Career/Options can't be clicked | The main thread's stack started right against `0xFFFFFFFF`. The game's 0x400-byte stack buffers ran past the end of the address space, and the emulator silently threw those writes away. | Leave 4 KiB of headroom above the initial stack (`src/stack.rs`, `src/mem.rs`) |
| Save file header can't be read (Windows builds only; harmless on iOS) | `read()` passed guest memory straight to `ReadFile`, which fails with error 998 on pages Windows hasn't committed yet | Read into a host buffer, then copy (`src/libc/posix_io.rs`) |

The same fixes are proposed upstream to HyperHLE from
[gabe4tw/HyperHLE-Headroom-Fix](https://github.com/gabe4tw/HyperHLE-Headroom-Fix)
(branch `stack-top-headroom`).

## Install on iPhone

Requires iOS 17.4 or newer (for older versions, see the JIT table in
[platform/ios/README.md](platform/ios/README.md)).

1. Download `Applesauce-iOS-unsigned.ipa` from the latest
   [release](../../releases). For the newest build of `main`, open the latest
   successful run of [Build iOS IPA](../../actions/workflows/build-ios-ipa.yml)
   while logged in to GitHub instead, download the `Applesauce-iOS-unsigned-…`
   artifact and unzip it.
2. Sideload it with AltStore, SideStore or Sideloadly. If the official
   Applesauce is installed, back up its saves first.
3. Set up [StikDebug](https://github.com/StephenDev0/StikDebug) with its pairing
   file and LocalDevVPN (one-time; see
   [platform/ios/README.md](platform/ios/README.md#stikdebug-and-localdevvpn)).
4. Supply your own copy of Skate It. This app doesn't include the game. Either
   tap **Import Game**, or put the decrypted `.ipa` in
   **Files › On My iPhone › Applesauce › touchHLE_apps**.

## Starting the game

The app starts Skate It on its own instead of waiting in the library:

1. Open the app. If JIT is off, it asks StikDebug to enable it (StikDebug
   briefly comes to the front).
2. Once JIT is on, Skate It starts. Exiting the game returns to the library;
   reopen the app to auto-start again.

If StikDebug isn't installed or JIT still isn't on when you come back, the usual
"JIT Isn't Enabled" prompt appears.

**Why JIT isn't simply always on:** on iOS 17.4+ a sideloaded app can't enable
JIT for itself. A debugger has to attach to it each time it starts, and that's
what StikDebug does. Permanent JIT only exists for TrollStore installs on A11
and older devices, which doesn't cover an iPhone 15.

The game to auto-start is set by `ApplesauceAutoLaunchBundleIdentifier` in
[platform/ios/Config/Info.plist](platform/ios/Config/Info.plist). Remove the
key to get Applesauce's normal library behaviour.

## Building

Every push to `main` builds the unsigned IPA on a GitHub macOS runner
([.github/workflows/build-ios-ipa.yml](.github/workflows/build-ios-ipa.yml)).
Publishing a release builds it too and attaches the IPA and its SHA-256 to the
release. Applesauce's own release and AltStore-feed workflows are removed, since
they publish for the Applesauce project.
To build on a Mac yourself, follow "Build From Source" in
[platform/ios/README.md](platform/ios/README.md).

## License

- The emulator and app source (everything inherited from Applesauce, HyperHLE
  and touchHLE, **including the modified files above**) is under the
  [Mozilla Public License 2.0](LICENSE), like upstream.
- The auto-start change modifies upstream's MPL-2.0 files
  (`platform/ios/Sources/NativeHost.swift`, `platform/ios/Config/Info.plist`) and
  is MPL-2.0 as well.
- Files added by this project that aren't derived from upstream (this README and
  `.github/workflows/build-ios-ipa.yml`) are under the
  [GNU GPL 3.0](LICENSE-GPL-3.0).
