# EpsonPerfection3170-PHOTO-Ubuntu2604LTS
DEB for thoose who need a driver. 

Epson Perfection 3170 (GT-9400) driver for 64-bit Ubuntu

A `.deb` package that makes the **Epson Perfection 3170 Photo** scanner
(USB ID `04b8:0116`, internal name **GT-9400**) work on modern 64-bit Ubuntu
(built and verified on Ubuntu 26.04 "resolute").

## Why this is non-trivial

The scanner speaks a proprietary Epson protocol handled by SANE's **epkowa**
backend. For this model, epkowa dlopen()s a 32-bit-only Epson *interpreter*
(`libesint*.so`) which in turn needs a firmware blob (`esfw32.bin`) uploaded to
the scanner. So a pure 64-bit SANE cannot drive it directly — you need a
**32-bit SANE stack** for the backend, while your scanning app is 64-bit.

This package bridges the two with a classic trick (from Alexander Trufanov's
`old-epson-drivers-epkowa-for-x64` project):

```
64-bit app (XSane/Skanlite/scanimage)
   -> SANE "net" backend (64-bit, system libsane 1.4)
      -> localhost:6566
         -> saned32 (32-bit daemon, systemd socket-activated)
            -> epkowa backend (32-bit)
               -> libesint + esfw32.bin firmware
                  -> the scanner over USB
```

## What modern Ubuntu broke (and how the package fixes it)

| Problem | Fix in this package |
| --- | --- |
| `saned32` needs `sanei_w_authorization_req`, **removed** from libsane 1.4 | Bundles an old `libsane.so.1.0.27` (i386) in `/usr/lib/epson-3170/i386/` and points `saned32` at it via `LD_LIBRARY_PATH` in the systemd unit |
| `libnsl.so.1` renamed (now `libnsl2` / `libnsl.so.2`) | `libc6:i386` (≥2.31) still ships a `libnsl.so.1` compat stub — pulled in as a normal dependency |
| Firmware `esfw32.bin` is **not** in the old bundle | Bundled, extracted from Fedora's redistributable `iscan-firmware` package → `/usr/share/iscan/esfw32.bin` |
| `libieee1284-3` renamed `libieee1284-3t64` | Depends on `libieee1284-3t64:i386 \\| libieee1284-3:i386` |

## Install

```sh
sudo apt install ./epson-perfection-3170-driver_2.10.0.1+ubuntu26.04-1_amd64.deb
```

The postinst: enables `i386` deps, creates the `saned` user, registers and
starts the `saned32.socket`, and adds `localhost` + `net` to your 64-bit
`/etc/sane.d/` config so frontends find the scanner automatically.

## Verify

```sh
scanimage -L
# -> device `net:localhost:epkowa:libusb:003:014' is a Epson Perfection 3170 flatbed scanner

scanimage -d "net:localhost:epkowa:libusb:003:014" --mode Color --resolution 100 --format=png -o scan.png
```

Then use **XSane**, **Skanlite** or **simple-scan** — the 3170 appears as a
`net:localhost:...` device.

## Files installed

- `usr/sbin/saned32`, `usr/sbin/saned32-run` — 32-bit daemon (+ wrapper)
- `usr/lib/i386-linux-gnu/sane/libsane-epkowa.so.1.0.15` — epkowa backend
- `usr/lib/i386-linux-gnu/libesmod.so.1.1.0` — Epson module lib
- `usr/lib/iscan/libesint*.so.2.0.0` — 32-bit interpreters for many models
- `usr/share/iscan/esfw32.bin` — GT-9400 / Perfection 3170 firmware
- `usr/lib/epson-3170/i386/libsane.so.1.0.27` — old libsane for saned32
- `lib/systemd/system/saned32.socket` + `saned32@.service`
- `etc/sane.d/epkowa.conf`, `etc/default/saned32`


Made using Kimi3 and Hermes Agent 
