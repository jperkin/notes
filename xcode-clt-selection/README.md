## Xcode vs CLT selection

Due to Apple, what should be an absolutely trivial operation of installing a
fricking compiler is of course a huge pain in the arse where you hit problems
at every single turn.

Do *NOT* attempt to do this in any reasonable way, you know, like via the App
Store, or downloading a specific release from <https://developer.apple.com>,
because every single path is littered with landmines.

For example:

* You can't just install an older Xcode on a newer OS, **even though it works
  absolutely fine**, because the installer just refuses to proceed.

* You can't use an older SDK with a newer compiler, **even though it works
  absolutely fine**, because `xcrun` just refuses to consider it:

```
$ xcrun --sdk macosx12.3 --show-sdk-path
2024-05-20 12:43:39.994 xcodebuild[94017:249847552] [MT] DVTSDK: Skipped SDK /Applications/Xcode-15.4.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX12.3.sdk; its version (12.3) is below required minimum (13.0) for the macosx platform.
xcodebuild: error: SDK "macosx12.3" cannot be located.
xcrun: error: unable to lookup item 'Path' in SDK 'macosx12.3'
```

* Note that if you use the latest Command Line Tools with SDK 12.3 then `xcrun`
  works fine!  Head, meet desk.

* You can't use (without workarounds) Command Line Tools 15.3+ on its own,
  because Apple broke m4 and yacc which are only shipped in the full Xcode
  build.

There are many other problems once you have the tools installed (like
intermittent SIGBUS when trying to use tools inside a chroot), but to keep this
document short it only considers the actual installation.

### Solution

First, go to <https://xcodereleases.com> and find the latest version of the
Xcode release series where the default SDK matches the one you want to target.

**HOWEVER**, note that during the initial run-up to a new version, the SDK will
still be on the older release for the first few revisions.  **DO NOT** install
them.

For example, if you are targetting SDK 12.3, the newest version is 13.4.1, and
not 14.0.1, as even though 14.0.1 targets SDK 12.3, it is one of the earliest
releases in the 14.x series and has major bugs (like `ld(1)` crashing) that are
fixed in later versions, but those have been switched to target a newer SDK.

Head over to <https://github.com/XcodesOrg/xcodes/releases>, unzip the latest
release, and use the `xcodes` CLI to install the version you want.

```
$ ./xcodes install 13.4.1
```

Do not install Command Line Tools.
