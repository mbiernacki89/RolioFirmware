# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is **RolioFirmware** — a ZMK-based keyboard firmware for the Rolio46, a 46-key split wireless keyboard with dual MIP displays (Vista508), rotary encoders, and backlight. The primary controller is the nice!nano v2.

## Build System

Firmware is built exclusively via **GitHub Actions** using ZMK's reusable workflow. There is no local build toolchain — push to the repo to trigger a build, then download `.uf2` artifacts from the Actions run.

- **Trigger:** Push to any branch, or manually via `workflow_dispatch`
- **Config:** `.github/workflows/build.yml` → `build.yaml` (defines left/right/reset targets)
- **Artifacts:** `zmk-rolio-left.uf2`, `zmk-rolio-right.uf2`, `zmk-nicenano_v2-settings_reset.uf2`

To flash: double-tap the reset button to enter bootloader, then drag `.uf2` onto the mounted drive.

## Dependency Management

Dependencies are pinned in `config/west.yml`:
- **ZMK:** pinned to a specific commit (not latest) for stability
- **ls0xxvcom-driver:** custom VCOM display driver for contrast reliability
- **zmk-userspace (elpekenin):** additional ZMK modules

When updating ZMK or modules, update the `revision:` field in `west.yml` and verify the build still succeeds.

## Architecture

### Shield/Board Separation
ZMK separates keyboard hardware (shield) from controller (board):
- `boards/shields/rolio/` — keyboard matrix, encoders, keymap, behaviors
- `boards/shields/vista508/` — display shield (separate from keyboard shield)
- `boards/shields/rolio/boards/nice_nano_v2.overlay` — controller-specific pin assignments

The `build.yaml` combines shields: `rolio_left + vista508` and `rolio_right + vista508`.

### Keymap File Structure
The keymap is split across multiple `.dtsi` files included by `rolio.keymap`:
- `keymap_aliases.dtsi` — human-readable key aliases (e.g., `LSPC`, `SYM`, `NAV`)
- `keymap_behaviors.dtsi` — custom hold-taps (homerow mods), mod-morphs, sensor bindings
- `keymap_macros.dtsi` — macros for workspace switching, currency symbols
- `rolio.keymap` — the 6-layer layout itself (QWR, NAV, SYM, FUN, HAX, BLE)

### Layers
| # | Name | Purpose |
|---|------|---------|
| 0 | QWR | QWERTY base |
| 1 | NAV | Arrows, media, Bluetooth switching |
| 2 | SYM | Symbols, numbers, brackets |
| 3 | FUN | F1–F12 |
| 4 | HAX | Gaming (WASD + arrows) |
| 5 | BLE | Bluetooth profile management |

### Display Widgets
`boards/shields/vista508/custom_status_screen.c` orchestrates the display. Widgets in `widgets/` render battery level, WPM, connection status, and custom art. Art images are converted to C arrays via `image_converter/` (Python, requires Pillow/numpy) — run the converter then commit the updated `art.c`.

### Key Custom Behaviors
- **Homerow mods:** Hold letters on home row to activate modifiers (defined in `keymap_behaviors.dtsi`)
- **Mod-morphs:** Shift+`,` → `;`, Shift+`.` → `:`, Alt+`'` → `` ` ``, Shift+`?` → `!`
- **Encoders:** Per-layer behavior — brightness, volume, scroll, or media control depending on active layer

## Keymap Editor

`config/rolio.json` and `boards/shields/rolio/rolio.zmk.yml` enable editing the keymap via [ZMK Studio](https://zmk.studio) or Keymap Editor web tools. ZMK Studio support is enabled in `config/rolio.conf` (`CONFIG_ZMK_STUDIO=y`).
