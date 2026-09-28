# Termux + termux-usb (non-root OTG)

Product path for BootROM → SPRD3 → FDL1 on an unrooted Android **host** phone.
Uses Termux + Termux:API (`termux-usb`) to hand an open usbfs FD to `spd_dump`.

**Out of scope for v1:** FDL2 SEND timeout / full flash success. Stop at FDL1 OK.

## Pin

This branch is based on TomKing `f2fc779210d9e4b5ca1904c79a49cc5e114b58f3`
(`put FindPort in loop`), plus Termux FD fixes:

- `--usb-fd N` as the **next** argument (normal parse)
- Legacy last-argv fallback for `termux-usb -e "./spd_dump --usb-fd"`
- If no `--usb-fd`, adopt `TERMUX_USB_FD` or `SPD_USB_FD` from the environment
  (what `termux-usb -E` exports)
- `libusb_wrap_sys_device` + `LIBUSB_OPTION_NO_DEVICE_DISCOVERY` (already at pin)

## Build on Termux

```bash
# Termux + Termux:API from F-Droid
pkg install termux-api clang make libusb git
git clone https://github.com/Seuj09/spreadtrum_flash.git
cd spreadtrum_flash
git checkout feature/termux-usb
make
```

## FDL assets (ums9230 / Infinix)

Do **not** expect huge FDL blobs in this repo. Copy them from the Seuj09
Termux companion release / tree:

```bash
# Example: from Spd_dump_termux no-root package
mkdir -p fdl/ums9230/infinix
cp /path/to/Spd_dump_termux/no-root/fdl/ums9230/infinix/fdl1-dl.bin fdl/ums9230/infinix/
cp /path/to/Spd_dump_termux/no-root/fdl/ums9230/infinix/fdl2-dl.bin fdl/ums9230/infinix/
```

Upstream: [Seuj09/Spd_dump_termux](https://github.com/Seuj09/Spd_dump_termux) `no-root/fdl/`.

## Smoke (BootROM → FDL1)

Cold-plug target in download mode (BootROM `1782:4d00`). Grant the USB
permission dialog immediately.

```bash
cd ~/spreadtrum_flash   # package root with ./spd_dump + scripts/

./scripts/spd_dump-usb --verbose 1 \
  exec_addr 0x65015f08 \
  fdl ./fdl/ums9230/infinix/fdl1-dl.bin 0x65000800 \
  fdl ./fdl/ums9230/infinix/fdl2-dl.bin 0x9efffe00 \
  exec
```

Expect: permission grant → CHECK_BAUD / SPRD3 handshake → FDL1 load/exec.
FDL2 SEND may still time out — that is **out of scope** for this gate.

## Raw termux-usb (advanced)

`spd_dump-usb` is a thin poll + `termux-usb -r -E -e` wrapper. Equivalent
one-shot once you know the bus path:

```bash
# -r request permission; -E export TERMUX_USB_FD into the child
termux-usb -r -E -e ./spd_dump /dev/bus/usb/N/M --verbose 1 \
  exec_addr 0x65015f08 \
  fdl ./fdl/ums9230/infinix/fdl1-dl.bin 0x65000800 \
  fdl ./fdl/ums9230/infinix/fdl2-dl.bin 0x9efffe00 \
  exec
```

Legacy append style (still supported):

```bash
termux-usb -e "./spd_dump --usb-fd" /dev/bus/usb/N/M
# FD is appended as the last argv; no further commands on that one-shot form
```

Or pass the FD explicitly:

```bash
./spd_dump --usb-fd "$TERMUX_USB_FD" --verbose 1 exec_addr 0x65015f08 ...
```

## Related

- Seuj09 spdhost (protocol/hello hardening): [Spd_dump_termux no-root](https://github.com/Seuj09/Spd_dump_termux/tree/main/no-root) — uses the same `termux-usb -E` contract.
- Android APK host (UsbManager FD, secondary): [spd-dump-host](https://github.com/Seuj09/spd-dump-host) — deferred vs Termux path.
