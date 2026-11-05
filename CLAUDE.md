# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A personal ZMK user-config repo (fork of MickiusMousius/RolioFirmware) for the Rolio46 split wireless keyboard: 46 keys, two EC11 encoders, PWM backlight, Vista508 memory-in-pixel display, nice!nano v2 controllers. The keymap is the owner's custom macOS-oriented layout; the upstream README's Windows/macOS per-profile switching and default-layout diagram no longer describe this keymap.

## Building

Firmware is built by default on GitHub Actions (`.github/workflows/build.yml`), which calls ZMK's reusable `build-user-config.yml@v0.3` on every push/PR, and local builds are possible through the upstream Docker devcontainer described below. The CI build matrix is `build.yaml`:
- `rolio_left vista508` (central, with the `studio-rpc-usb-uart` snippet for ZMK Studio)
- `rolio_right vista508` (peripheral)
- `settings_reset`

Dependencies are in `config/west.yml`: ZMK is pinned to commit f9be3ed so the build keeps working; the MickiusMousius modules `zmk-ls0xxvcom-driver` (Vista508 display driver), `zmk-patch-batterylevel`, `zmk-patch-ec11`, `zmk-smart-sleep`, `zmk-defaultlayer` and `zmk-ble-passkey` track `main` (clone-depth 1). Changing the ZMK revision or the workflow tag can break the build — upstream deliberately locked these.

Local builds use upstream's setup: `doc/tools_setup.md` (Docker + VS Code devcontainer), `doc/building.md` (manual commands), and `tools/build_rolio.sh [left|right|all]` which builds inside the container (paths under /workspaces). `tools/readme.md` covers the other tools (keymap formatter/drawer, `clean_code.sh`, etc.).

The repo root is itself a Zephyr module (`zephyr/module.yml`, `board_root: .` and `dts_root: .`), which is how `boards/shields/*` and the custom behavior bindings in `dts/bindings/behaviors/` get picked up.

## Layout

- `boards/shields/rolio/` — the keyboard shield.
  - `rolio.dtsi`: kscan matrix (4 rows × 14 cols), matrix transform, encoders, backlight; `rolio46-layout.dtsi` is the physical layout for Studio. `rolio_left.overlay`/`rolio_right.overlay` set column GPIOs, enable the per-side encoder, and set up soft-off wakeup; the right side uses `col-offset = <6>`.
  - `Kconfig.defconfig`: left half is the split central.
  - `boards/nice_nano_v2.overlay`, `boards/mikoto.overlay`: controller-specific pinctrl for backlight PWM and the display SPI bus.
  - `CMakeLists.txt` compiles `behavior_wpm_tog.c` and `behavior_mod_tog.c`: `&wpm_tog` cycles the left display's info area (typing animation / WPM graph / modifiers), `&mod_tog` toggles auto-showing held modifiers. Both are defined in `keymap_behaviors.dtsi`.
  - `rolio.keymap`: layers QWR/NAV/SYM/FUN/HAX/BLE. The `#define` indices at the top must match the order of layer nodes. Each layer has `sensor-bindings` for the two encoders. Bindings use the ASCII box-drawing comments as a visual grid — keep the column alignment when editing. The BLE layer (index 5) holds output/profile selection and the display toggles; `CONFIG_VISTA508_SETTINGS_LAYER=5` in `config/rolio.conf` must equal its index, because the display shows toggle confirmations only while that layer is the highest active one.
  - `keymap_aliases.dtsi` (`__` = `&trans`, `xx` = `&none`, mod/hold-tap helper macros), `keymap_behaviors.dtsi` (hold-taps, mod-morphs, sensor-rotate, display toggles), `keymap_macros.dtsi` (app-launch / desktop / screenshot macros, mostly macOS shortcuts; `&output_usb` and `&output_bt N` force the output and select BT profile N). These are `#include`d by `rolio.keymap`.
- `boards/shields/vista508/` — display shield with a custom LVGL status screen (adapted from ZMK's nice!view; keep the MIT attribution). It is the default status screen (`Kconfig.defconfig`). `CMakeLists.txt` splits by role:
  - Central: `src/status_screen.c`, `src/styles.c`, `src/widgets/` (battery, output, layer, `multi_widget` with its views under `multi_widget/`), `src/assets/` (images, animations, fonts), plus a caps word override in `src/events` and `src/behaviors` that raises a state-changed event for the modifier indicator.
  - Peripheral: `peripheral_screen.c`, `widgets/peripheral_status.c`, `widgets/art.c`, `widgets/bolt.c`, `widgets/util.c`.
  - Kconfig: `VISTA508_SETTINGS_LAYER` (layer index whose active state shows setting-change notices, default -1), `VISTA508_MESSAGE_DISPLAY_MS` (default 5000), `VISTA508_CHARGE_FULL_PCT` (charging is inferred as USB powered and below this, default 95), `VISTA508_WIDGET_INVERTED`.
- `config/rolio.conf` — Kconfig for both halves (keyboard name, pointing/scroll, sleep + peripheral smart sleep, soft-off, backlight, BLE TX power, split battery, passkey, Studio). The built-in EC11 driver is off (`CONFIG_EC11=n`) in favour of `ZMK_PATCH_EC11`; enabling both causes linker conflicts.
- `config/rolio.json` — layout definition for the web Keymap Editor; not used by the build.
- `image_converter/` — Python tool that turns a folder of images into `art.c` for the peripheral display (`./setup.sh`, `source venv/bin/activate`, `python3 converter/main.py --inDir <dir> --outDir <out>`). Copy the result to `boards/shields/vista508/widgets/art.c`. Images are scaled to 144×147; keep the count low (≤ ~25) or the firmware may not fit. Its readme's `sample_images` dir doesn't exist in this repo. `art.c` currently holds one image (`donkey.png`, 144×144); its palette `#if` uses `CONFIG_VISTA508_WIDGET_INVERTED`.
