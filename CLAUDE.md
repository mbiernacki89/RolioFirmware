# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A personal ZMK user-config repo (fork of MickiusMousius/RolioFirmware) for the Rolio46 split wireless keyboard: 46 keys, two EC11 encoders, PWM backlight, Vista508 memory-in-pixel display, nice!nano v2 controllers. The keymap is the owner's custom macOS-oriented layout; the upstream README's Windows/macOS per-profile switching and default-layout diagram no longer describe this keymap.

## Building

There is no local build setup in the repo. Firmware is built by GitHub Actions (`.github/workflows/build.yml`), which calls ZMK's reusable `build-user-config.yml@v0.3` on every push/PR. The build matrix is `build.yaml`:
- `rolio_left vista508` (central, with the `studio-rpc-usb-uart` snippet for ZMK Studio)
- `rolio_right vista508` (peripheral)
- `settings_reset`

Dependencies are pinned in `config/west.yml`: ZMK and `zmk-userspace` are pinned to specific commits so the build keeps working; `zmk-ls0xxvcom-driver` (Vista508 display driver) tracks `main`. Changing the ZMK revision or the workflow tag can break the build — upstream deliberately locked these.

To build locally you need a standard ZMK west workspace (`west init -l config && west update`), then e.g.:
```
west build -s zmk/app -b nice_nano_v2 -- -DSHIELD="rolio_left vista508" -DZMK_CONFIG=$PWD/config -DZMK_EXTRA_MODULES=$PWD
```
The repo root is itself a Zephyr module (`zephyr/module.yml`, `board_root: .`), which is how `boards/shields/*` get picked up.

## Layout

- `boards/shields/rolio/` — the keyboard shield.
  - `rolio.dtsi`: kscan matrix (4 rows × 14 cols), matrix transform, encoders, backlight; `rolio46-layout.dtsi` is the physical layout for Studio. `rolio_left.overlay`/`rolio_right.overlay` set column GPIOs, enable the per-side encoder, and set up soft-off wakeup; the right side uses `col-offset = <6>`.
  - `Kconfig.defconfig`: left half is the split central.
  - `boards/nice_nano_v2.overlay`, `boards/mikoto.overlay`: controller-specific pinctrl for backlight PWM and the display SPI bus.
  - `rolio.keymap`: layers QWR/NAV/SYM/FUN/HAX/BLE. The `#define` indices at the top must match the order of layer nodes. Each layer has `sensor-bindings` for the two encoders. Bindings use the ASCII box-drawing comments as a visual grid — keep the column alignment when editing.
  - `keymap_aliases.dtsi` (`__` = `&trans`, `xx` = `&none`, mod/hold-tap helper macros), `keymap_behaviors.dtsi` (hold-taps, mod-morphs, sensor-rotate behaviors), `keymap_macros.dtsi` (app-launch / desktop / screenshot macros, mostly macOS shortcuts). These are `#include`d by `rolio.keymap`.
- `boards/shields/vista508/` — display shield with a custom LVGL status screen (adapted from ZMK's nice!view; keep the MIT attribution). `CMakeLists.txt` compiles `widgets/status.c` on the central and `widgets/art.c` + `peripheral_status.c` on the peripheral. `CONFIG_ZMK_DISPLAY_STATUS_SCREEN_BUILT_IN=n` in `config/rolio.conf` selects this custom screen.
- `config/rolio.conf` — Kconfig for both halves (keyboard name, encoders, pointing/scroll, soft-off, backlight, BLE TX power, split battery, Studio).
- `config/rolio.json` — layout definition for the web Keymap Editor; not used by the build.
- `image_converter/` — Python tool that turns a folder of images into `art.c` for the peripheral display (`./setup.sh`, `source venv/bin/activate`, `python3 converter/main.py --inDir <dir> --outDir <out>`). Copy the result to `boards/shields/vista508/widgets/art.c`. Images are scaled to 144×147; keep the count low (≤ ~25) or the firmware may not fit.
