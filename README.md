# sprinklr-firmware

Compiled firmware binaries for **[Sprinklr](https://github.com/dunlapbs/sprinklr)**
sprinkler controllers. This repository is **public** on purpose — its
only job is to host `.bin` files at stable URLs that the devices themselves
can fetch for remote OTA updates.

The source code lives in the private [`dunlapbs/sprinklr`](https://github.com/dunlapbs/sprinklr)
repository; this repo just carries the build output so it can be linked to
without exposing the source.

## Layout

```
.
├── esp32-wroom-32e/        LC-Tech ESP32 Relay X8 board (current hardware)
│   ├── firmware.bin        App-partition image. Load at 0x10000 for OTA
│   │                         updates or over-the-air via /api/firmware-pull.
│   ├── firmware-full.bin   Merged bootloader + partitions + app at 0x0.
│   │                         Use for USB flashing a blank chip.
│   └── VERSION             Build-info: version string, source commit,
│                             publish timestamp, sha256 of each bin.
└── (future boards go in their own sibling folder, same shape)
```

## Using these binaries

### Remote OTA from the portal

The device's `POST /api/firmware-pull` endpoint accepts a URL and streams
the firmware in via `HTTPUpdate`. Point it at GitHub's raw-content URL:

```
curl -X POST -H 'Content-Type: application/json' \
     -d '{"url":"https://raw.githubusercontent.com/dunlapbs/sprinklr-firmware/main/esp32-wroom-32e/firmware.bin"}' \
     http://<device-ip>/api/firmware-pull
```

The device downloads, writes to the inactive OTA partition, and reboots
into the new image (~6–10 s total). `firmware.bin` is what the pull path
wants — **not** `firmware-full.bin`.

### USB flashing a fresh chip

Use `firmware-full.bin` with `esptool` at offset `0x0`. It bundles the
bootloader, partition table, and app:

```
esptool.py --chip esp32 --port <COM> --baud 460800 erase_flash
esptool.py --chip esp32 --port <COM> --baud 460800 \
           write_flash --flash_mode dio --flash_freq 40m --flash_size 4MB \
           0x0 firmware-full.bin
```

On Windows, the ARM-native way is `python -m esptool …`. On macOS/Linux,
`esptool.py` works the same.

## Verifying a download

`VERSION` in each device folder lists the sha256 of both binaries. Compare
with `sha256sum firmware.bin` on your machine, or check the version string
reported by the device at `GET /api/status` after the flash completes —
it should match the `version:` line in `VERSION`.

## Publishing new builds

This repo is regenerated from the source tree. Nothing here should be
hand-edited. The source repo's Makefile has a target that rebuilds,
copies the fresh binaries here, updates `VERSION`, commits, and pushes.
