---
title: "Getting the Goodix HTK32 Fingerprint Reader Working on Omarchy"
date: 2026-05-03
slug: goodix-htk32-omarchy
description: "Five stacked failure modes between a fresh Omarchy install and a working fingerprint reader on a Dell XPS 9370."
tags:
  - linux
  - arch
  - omarchy
  - fprintd
  - libfprint
  - goodix
  - dell-xps
draft: false
---

## TL;DR

Dell XPS 9370 with the Goodix HTK32 (USB `27c6:5385`) on Omarchy:

```bash
sudo pacman -S glib2-devel
yay -S libfprint-goodix53x5
# Answer 'y' to remove the stock libfprint when prompted.
# If yay was wrapped with --noconfirm and bailed, install the cached build directly:
sudo pacman -U ~/.cache/yay/libfprint-goodix53x5/libfprint-goodix53x5-*.pkg.tar.zst

sudo systemctl restart fprintd
fprintd-enroll   # retry once if you hit a GTLS error on the first run
fprintd-verify
```

The package ships a udev rule that unbinds `cdc_acm` from the sensor on boot, so the fix survives reboots. If the sensor was previously paired with Windows Hello, expect more pain — see Stage 4.

---

## The hardware

```
$ lsusb | grep 27c6
Bus 001 Device 004: ID 27c6:5385 Shenzhen Goodix Technology Co.,Ltd. Fingerprint Reader
```

The `5385` / `5395` USB IDs aren't supported by mainline libfprint. There's an [open merge request](https://gitlab.freedesktop.org/libfprint/libfprint/-/merge_requests/43) and a downstream driver at [AndyHazz/goodix53x5-libfprint](https://github.com/AndyHazz/goodix53x5-libfprint) that the AUR package `libfprint-goodix53x5` wraps. Five things went wrong between "yay -S" and a working `fprintd-verify`. Each is solvable; in aggregate they ate an afternoon.

## Stage 1 — meson fails on `glib-mkenums`

The build aborted at meson configure:

```
Program /usr/bin/glib-mkenums found: NO
libfprint/meson.build:185:17: ERROR: Dependency 'glib-2.0' tool variable
'glib_mkenums' contains erroneous value: '/usr/bin/glib-mkenums'
```

Arch [split](https://gitlab.archlinux.org/archlinux/packaging/packages/glib2/-/commit/02a3a726) the GLib developer tools (`glib-mkenums`, `glib-genmarshal`, `gdbus-codegen`) out of `glib2` into a separate `glib2-devel` package in May 2024. The `glib-2.0.pc` file still advertises `/usr/bin/glib-mkenums` as a tool variable, but the binary lives in the devel package. AUR PKGBUILDs that haven't been refreshed since then miss this in their `makedepends`.

```bash
sudo pacman -S glib2-devel
```

Sanity check:

```bash
pacman -Qo /usr/bin/glib-mkenums   # should print: glib2-devel
```

## Stage 2 — conflict with stock libfprint

```
:: libfprint-goodix53x5-1.94.10-7 and libfprint-1.94.10-1 are in conflict.
   Remove libfprint? [y/N]
error: unresolvable package conflicts detected
```

`libfprint-goodix53x5` is a drop-in replacement for `libfprint`, not a parallel install — same `libfprint-2.so.2`, with Goodix patches on top. The PKGBUILD declares `conflicts=('libfprint')` and `provides=('libfprint')`, so anything that depends on `libfprint` keeps working after the swap.

The `omarchy-pkg-aur-install` wrapper passes `--noconfirm` to yay, which means pacman's default answer to the conflict prompt (N) wins and the install bails. Two ways out:

```bash
# Drop the wrapper, run yay interactively:
yay -S libfprint-goodix53x5

# Or install the already-built package from the cache:
sudo pacman -U ~/.cache/yay/libfprint-goodix53x5/libfprint-goodix53x5-*.pkg.tar.zst
```

Heads up: while this is installed, official `libfprint` updates from `extra` are held back. You'll need to rebuild the AUR package whenever upstream version-bumps. `yay -Sua` picks this up.

## Stage 3 — `Device 27c6:5385 is already open`

First failure mode:

```
failed to claim device: GDBus.Error:net.reactivated.Fprint.Error.Internal:
Open failed with error: Device 27c6:5385 is already open
```

After a restart, the more useful error:

```
USB error on device 27c6:5385 : Resource busy [-6]
```

`EBUSY` means a kernel driver is holding the interface, not a stale userspace process. My first guess was `usbhid` — that's the more common culprit for fingerprint readers — but `lsusb -t` showed otherwise:

```
|__ Port 010: Dev 004, If 0, Class=Communications, Driver=cdc_acm, 12M
|__ Port 010: Dev 004, If 1, Class=CDC Data,       Driver=cdc_acm, 12M
```

The Goodix HTK32 advertises a USB Communications Device Class descriptor, so the kernel's `cdc_acm` module claims it as a serial modem on every boot. libfprint's libusb access then can't take it back.

The AUR package ships `91-goodix-fingerprint.rules`, which tells udev to unbind `cdc_acm` whenever it attaches to `27c6:5385`. Once installed it's automatic, but for a one-shot manual unbind during debugging:

```bash
echo -n "1-10:1.0" | sudo tee /sys/bus/usb/drivers/cdc_acm/unbind
```

(Bus 1, port 10. The companion interface `.1` releases automatically when its sibling unbinds.)

Lesson: always check `lsusb -t` before assuming which kernel module is hijacking a USB device. Same shape of problem, different driver.

## Stage 4 — `GTLS identity verification failed`

```
Open failed with error: GTLS identity verification failed
```

This one cost the most time. The Goodix sensors store a PSK in flash and do a TLS-PSK handshake with the host. The PSK [cannot be read back](https://github.com/fwupd/fwupd/discussions/3637) — once written, only re-flashing firmware clears it. So if Windows Hello has ever paired the sensor, its PSK is stuck in there and Linux can't authenticate against it.

I went down the rabbit hole: [goodix-fp-dump](https://github.com/goodix-fp-linux-dev/goodix-fp-dump), the [Neodyme reverse-engineering writeup](https://neodyme.io/en/blog/fingerprint_reversing/), the firmware-flashing discussion in fwupd, the Trust-On-First-Use whitebox-PSK dance Windows does. All interesting, none of it useful here.

What actually fixed it: retrying. With verbose logging in case the next attempt also failed:

```bash
sudo systemctl stop fprintd
sudo G_MESSAGES_DEBUG=all FP_DEBUG=3 LIBUSB_DEBUG=3 /usr/lib/fprintd/fprintd &
# In another terminal:
fprintd-enroll
```

It worked. Not because of the debug flags — because the driver's TOFU PSK negotiation needed a couple of attempts to settle. The first failed run completed the pairing; subsequent runs sailed through.

**Takeaway**: on a sensor that's never seen Windows, the GTLS error on first enrollment is plausibly transient. Retry once or twice before going anywhere near the firmware-flashing tools.

If you do have a Windows-paired sensor, the realistic path is the [goodix-fp-linux-dev](https://github.com/goodix-fp-linux-dev) project — they have a Discord and `flash_5385.py` scripts. Their own README warns the tooling is unstable, and the Goodix firmware blobs aren't legally redistributable, so you're sourcing them yourself. Most people in this situation are better off without fingerprint auth until upstream matures.

## Stage 5 — enrolled, but `verify-no-match`

```
Verify started!
Verifying: right-index-finger
Verify result: verify-no-match (done)
```

Enrollment completed with all 9 stages but verification refused to match. The driver uses [SIGFM](https://github.com/goodix-fp-linux-dev/sigfm) — SIFT keypoint matching with CLAHE contrast preprocessing — on a tiny 108×88 px capacitive image. It's sensitive to finger placement and pressure in a way the more common minutiae-based matchers aren't.

Re-enroll, paying attention to the things that matter to a SIFT matcher:

```bash
fprintd-delete $USER
fprintd-enroll
```

- **Cover the sensor area.** Across the 9 captures, shift the finger a few millimeters between presses (left, right, up, down, slightly rotated) so the stored samples cover the region you'll naturally land on during verification.
- **Press flat with the pad of the finger.** Not the tip.
- **Match the pressure you'll use later.** Mashing during enroll then tapping during verify gives meaningfully different capacitive images.
- **Skin condition matters.** Capacitive sensors hate dry skin. If your hands are freshly washed or it's a low-humidity day, rub your finger against your nose for natural oils. Sounds silly, works.

After a careful re-enroll, `fprintd-verify` matched cleanly.

## Final state

```bash
$ fprintd-verify
Using device /net/reactivated/Fprint/Device/0
Listing enrolled fingers:
 - #0: right-index-finger
Verify started!
Verifying: right-index-finger
Verify result: verify-match (done)
```

A reboot confirmed the udev rule survives, so the device is properly free at boot.

For PAM integration (sudo, hyprlock, sddm), drop:

```
auth   sufficient   pam_fprintd.so
```

above the `pam_unix.so` line in `/etc/pam.d/sudo` and your screen locker's PAM file. Test sudo in a *new* terminal before closing your existing root session — typos in PAM config can lock you out of authentication entirely.

## Caveats

If you ever boot Windows on this machine, Windows Hello will re-pair the sensor with its own PSK and you'll be back to GTLS errors on Linux until you flash it back. With this driver in its current state, it's effectively one-OS-at-a-time.

The AUR package will need rebuilding whenever upstream `libfprint` version-bumps in `extra`, since the goodix53x5-patched package holds back the stock one. If/when the patches land in upstream libfprint, switching back is a simple `pacman -S libfprint`.

---

Five stacked failure modes, none individually a showstopper, none documented in one place. Hope this saves someone else the afternoon.
