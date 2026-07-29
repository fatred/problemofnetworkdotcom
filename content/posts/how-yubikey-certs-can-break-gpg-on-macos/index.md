---
title: "How Yubikey resident certs can break Yubi resident GPG on macOS"
date: 2026-07-29
tags: ["yubikey", "gpg", "macos", "piv", "smartcard"]
summary: "gpg --card-status started failing with 'Operation not supported by device' after a macOS upgrade. The cause wasn't the upgrade — it was the OpenSC CryptoTokenKit extension quietly claiming the reader."
---

_AI Content Warning: This post was written after diagnosing this problem using Claude Opus 5. The post was produced by claude and edited by me before publishing_

## TL;DR

`gpg --card-status` failed with `Operation not supported by device` on macOS. The
YubiKey was fine, PC/SC was fine, and nothing was paired for smartcard login. The
culprit was the **OpenSC CryptoTokenKit token extension**, which had registered
itself as a CTK token driver and was holding the PC/SC connection to the card's
CCID applet. `scdaemon` asked for the same reader and was refused.

One command fixed it:

```bash
pluginkit -e ignore -i org.opensc-project.mac.opensctoken.OpenSCTokenApp.OpenSCToken
```

The rest of this post is the diagnostic path, because the error message points
in entirely the wrong direction and the obvious suspects are all innocent.

## The symptom

```
$ gpg --card-status
gpg: selecting card failed: Operation not supported by device
gpg: OpenPGP card not available: Operation not supported by device
```

This appeared the morning after a macOS upgrade, which sent me straight down a
blind alley. The upgrade was a red herring — it just happened to be the reboot
that re-registered the extension that was actually at fault. What I'd *really*
done was, a few days earlier, load a certificate into PIV slot 9a for an
unrelated Kubernetes mTLS project.

## Step 1: figure out which layer is broken

`Operation not supported by device` reads like a USB or driver problem. It isn't.
It means `scdaemon` found the card and failed to **claim** it. So the first job
is establishing how far up the stack things are healthy.

```bash
$ ykman list
YubiKey 5C Nano (5.7.1) [OTP+FIDO+CCID] Serial: 12345678
```

`ykman` talks to the card over PC/SC. If it works, the reader, the CCID applet,
and the PC/SC layer are all fine, and the problem is specifically that something
is preventing *gpg* from getting a connection.

(`system_profiler SPUSBDataType | grep -i yubi` returned nothing for me. Ignore
that — it's unreliable on recent macOS and `ykman` is the authoritative answer.)

## Step 2: rule out other card consumers

Anything holding an exclusive connection to the card will produce this. The usual
suspects are the Yubico Authenticator GUI, a stray `ykman` process, a Homebrew
`pcscd` competing with the system PC/SC service, or — in my case very plausibly —
my own tooling, since the PIV project I'd been working on opens the card via
`piv-go` and holds it for the life of the process.

```bash
$ gpgconf --kill all
$ pgrep -fl 'yubikey|scdaemon|pcscd|ykman|Yubico'
3145 /System/Library/Frameworks/PCSC.framework/.../com.apple.ctkpcscd
3490 /System/Library/Frameworks/PCSC.framework/.../com.apple.ctkpcscd
```

Only Apple's own `ctkpcscd` — the CryptoTokenKit PC/SC daemon. That's expected
and always running. No third-party process was holding the card.

## Step 3: the smartcard-login red herring

Since rebooting, macOS had started prompting me about using the 9a certificate
for login. Reasonable hypothesis: CryptoTokenKit had decided the token was a
login identity and grabbed it accordingly.

```bash
$ sc_auth identities
SmartCard: org.opensc-project.mac.opensctoken.OpenSCTokenApp.OpenSCToken:a1b1...
Unpaired identities:
7CF7...C99A	Certificate for PIV Authentication (alice.anderson)

$ sc_auth list
(empty)
```

`sc_auth list` being empty is the important bit: **nothing was paired**. The
popups were macOS *offering* to pair, not evidence that it had. So login pairing
was not the blocker, and disabling it would not have fixed anything.

It's still worth killing the prompts, since they're noise:

```bash
sudo sc_auth pairing_ui -s disable
```

I'd avoid the bigger hammer here. Setting `allowSmartCard -bool false` in
`/Library/Preferences/com.apple.security.smartcard` will stop the prompts too,
but on some releases it also disables the CryptoTokenKit PIV token driver that
PC/SC-based PIV tooling depends on. You'd trade one broken thing for another.

## Step 4: the actual cause

The real signal was hiding in plain sight on the first line of `sc_auth
identities` — an OpenSC token identity. Which means the OpenSC **CryptoTokenKit
token extension** is installed and active:

```bash
$ pluginkit -m -p com.apple.ctk-tokens -vvv
...
     org.opensc-project.mac.opensctoken.OpenSCTokenApp.OpenSCToken(1.1.1)
	            Path = /Applications/Utilities/OpenSCTokenApp.app/Contents/PlugIns/OpenSCToken.appex
	     Parent Bundle = /Applications/Utilities/OpenSCTokenApp.app
	    Display Name = OpenSC token driver

$ security list-smartcards
com.apple.pivtoken:A1B1F8968C244BC0B89E0BD5702A9BBB
org.opensc-project.mac.opensctoken.OpenSCTokenApp.OpenSCToken:a1b1f8968c244bc0b89e0bd5702a9bbb
```

Two drivers, same card. OpenSCToken registers as a CTK token driver, and when
the card is inserted it opens a PC/SC connection to the CCID applet and holds
it. `scdaemon` then asks for the same reader and gets refused.

Why now? OpenSC tends to arrive as a dependency of PIV tooling, so it had almost
certainly been installed during the Kubernetes work. The macOS upgrade
re-registered the extension on first boot, which is why the timing pointed at
the OS.

## The fix

```bash
pluginkit -e ignore -i org.opensc-project.mac.opensctoken.OpenSCTokenApp.OpenSCToken
gpgconf --kill all
# unplug and replug the key
gpg --card-status
```

```
Reader ...........: Yubico YubiKey OTP FIDO CCID
Application ID ...: D2760001240100000006123456780000
Application type .: OpenPGP
Version ..........: 3.4
Manufacturer .....: Yubico
Serial number ....: 12345678
Name of cardholder: Alice Anderson
Login data .......: alice.anderson@problemofnetwork.com
Key attributes ...: rsa4096 rsa4096 rsa4096
PIN retry counter : 5 5 5
Signature counter : 1514
Signature key ....: 3F2A 9C41 7BD8 05E6 1A93  C7F0 48BE 2D15 6A0C 93E7
Encryption key....: 8D14 6B2E 9F03 A7C5 20D8  4E61 B39A 7C42 15FF 8B60
Authentication key: C60B 41D7 2E95 8A03 F71C  6B24 90E5 3D18 47AC 22F9
```

Back in business.

## Making it stick

`pluginkit -e ignore` is per-user state and gets re-evaluated when the parent
bundle's timestamp changes. An OpenSC upgrade or another macOS update can
re-register the extension and put you right back here. If you don't need the CTK
token driver, remove the bundle:

```bash
brew list --cask opensc 2>/dev/null || brew list opensc 2>/dev/null
sudo rm -rf /Applications/Utilities/OpenSCTokenApp.app
```

Removing the app bundle kills the token extension while leaving
`opensc-pkcs11.so` intact if it's installed elsewhere — the PKCS#11 module and
the CTK extension are separate components.

## The scdaemon.conf worth having

I found this already in `~/.gnupg/scdaemon.conf`, presumably from the last time I
fought this:

```
disable-ccid
reader-port Yubico YubiKey OTP+FIDO+CCID
```

`disable-ccid` is correct and worth keeping. It stops gpg using its built-in
libusb CCID driver — which claims the USB interface exclusively — and puts it on
PC/SC instead, so it can coexist with other PC/SC consumers.

`reader-port` is a trap. Compare the pinned string to the reader name gpg
actually reported above: `OTP+FIDO+CCID` versus `OTP FIDO CCID`. It doesn't
match. It only worked because `scdaemon` falls back to the sole available reader
when the pinned one isn't found. The PC/SC reader name is derived from which
interfaces are enabled on the key, so a `ykman config usb` change rewrites it,
and macOS sometimes appends a slot suffix. Pin it and it will fail confusingly
at the worst moment.

What I settled on:

```
disable-ccid
pcsc-shared
```

`pcsc-shared` lets gpg hold a shared connection so it can coexist with
PC/SC-based PIV tooling on the same key. One caveat: it's shared *access*, not
shared *transactions*. Fire a gpg operation at the exact moment another consumer
is mid-operation and you can still get a transient conflict. Rare, fails cleanly,
but that's the mechanism if you ever see one.

I'd also skip the explicit `pcsc-driver` path that a lot of older guides
recommend. Current gnupg finds the PCSC framework on its own, and a hardcoded
system path is one more thing to break on the next OS bump.

## Takeaways

1. `Operation not supported by device` means **claim failed**, not **card not
   found**. `ykman list` tells you which.
2. On macOS, check `security list-smartcards` early. Two drivers registered for
   one card is the whole problem, visible in one line.
3. `sc_auth identities` showing something is not the same as `sc_auth list`
   showing something. Unpaired identities are offers, not pairings.
4. Loading a PIV cert onto a key you also use for OpenPGP changes which macOS
   subsystems take an interest in it. The failure surfaced days later, at a
   reboot, which made an innocent OS upgrade look guilty.