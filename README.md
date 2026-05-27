# ZMK Keyboard for Cornix (Prospector edition)

ZMK firmware for the Cornix split keyboard, configured to run with a
[Prospector](https://github.com/carrefinho/prospector-zmk-module)
dongle (Seeed XIAO nRF52840 + display module).

Both halves of the keyboard act as BLE peripherals and pair to the
Prospector dongle, which acts as the central and exposes the keyboard
to the host (USB / BLE).

![image](images/cornix_with_dongle.png)
![image](images/cornix_layout.png)

## Hardware

- **Cornix split keyboard** (Jezail Funder) — Corne-inspired 3×6
  column-staggered split with a 3-key thumb cluster per half, Choc V2
  hot-swap, EC11 encoder support.
- **Prospector dongle** — Seeed XIAO nRF52840 + the Prospector display
  module. Acts as the BLE central and the USB host endpoint.

## Boards and shields

### Boards

- **`cornix_left`** — left half of Cornix. BLE peripheral that pairs
  with the dongle.
- **`cornix_right`** — right half of Cornix. BLE peripheral.

### Shields

- **`cornix_prospector`** — Cornix-specific glue for the Prospector
  module: matrix layout and Cornix-side pinout for the dongle build.
  Used together with `prospector_adapter` below.
- **`prospector_adapter`** — Provided by the
  [prospector-zmk-module](https://github.com/carrefinho/prospector-zmk-module).
  Defines the XIAO nRF52840 + display hardware on the dongle side. Always
  combined with `cornix_prospector` for the Cornix dongle build.
- **`cornix_indicator`** *(optional)* — Drives the on-board RGB LEDs as
  battery / connection indicators on the keyboard halves. Consumes
  noticeably more power. To enable it, add the shield to the
  `cornix_left` / `cornix_right` entries in `build.yaml`:

  ```yaml
  - board: cornix_left
    shield: cornix_indicator
    artifact-name: cornix_left
  - board: cornix_right
    shield: cornix_indicator
    artifact-name: cornix_right
  ```

## Firmware targets

`build.yaml` (full matrix, run manually) builds everything:

| Artifact            | Board                    | Notes                                |
| ------------------- | ------------------------ | ------------------------------------ |
| `cornix_left`       | `cornix_left`            | Left peripheral                      |
| `cornix_right`      | `cornix_right`           | Right peripheral                     |
| `cornix_reset`      | `cornix_right`           | `settings_reset` for either half     |
| `prospector_dongle` | `xiao_ble/nrf52840/zmk`  | Prospector dongle (central + studio) |
| `prospector_reset`  | `xiao_ble/nrf52840/zmk`  | `settings_reset` for the dongle      |

`build.dongle.yaml` is a reduced matrix that only builds
`prospector_dongle`. It is what the push-triggered CI uses, so iterating
on the dongle does not burn time rebuilding the keyboard halves.

## CI workflows

Two workflows live in `.github/workflows/`:

- **Build Dongle Firmware (on push to main)** — runs automatically on
  every push to `main` that touches `boards/**`, `config/**`, or
  `build.dongle.yaml`. Uses `build.dongle.yaml`, so only the
  `prospector_dongle` UF2 is produced.
- **Build All Firmware (manual)** — runs only on `workflow_dispatch`.
  Uses `build.yaml` and produces every artifact in the table above.

Download the resulting `.uf2` files from **Actions → run → Artifacts**.

## Flashing

1. Put the target board into UF2 bootloader mode (double-tap reset).
2. Drag the corresponding `.uf2` onto the mounted drive.
3. First-time setup or after pairing issues: flash `cornix_reset.uf2`
   on each half and `prospector_reset.uf2` on the dongle, then flash
   the regular firmware in the same order.

Bootloader notes: since v2.3 the flash layout no longer includes the
SoftDevice partition, so the regular UF2 can be flashed directly. If
you need to roll back to the stock RMK firmware or restore the
SoftDevice, see [`bootloader/README.md`](./bootloader/README.md) and
the backup files under `rmkfw/`.

## Customize and build via GitHub Actions

Recommended path for keymap-only changes.

1. **Fork** this repository on GitHub.
2. Edit `config/cornix.keymap` directly or with
   [ZMK Keymap Editor](https://nickcoutsos.github.io/keymap-editor/).
3. Commit & push to `main`. The push-triggered CI builds the dongle
   firmware automatically.
4. To rebuild the keyboard halves too, run the **Build All Firmware
   (manual)** workflow from the Actions tab.
5. Download UF2s from the run's Artifacts and flash.

## Build locally

This repo ships a Nix `flake.nix` providing `west`, the Zephyr SDK and
all Python deps. The west workspace lives inside `zmk_exts/` (kept out
of git) so that it does not collide with this repo's own `zephyr/`
module manifest.

### First-time setup

```bash
nix develop                                # enter the dev shell

# Create an isolated west workspace under zmk_exts/
mkdir -p zmk_exts/config
cp config/west.yml zmk_exts/config/west.yml

cd zmk_exts
west init -l config
west update --fetch-opt=--filter=blob:none
west zephyr-export
```

You will end up with `zmk_exts/` containing `zmk/`, `zephyr/`,
`zmk-helpers/`, `zmk-dongle-display/`, `prospector-zmk-module/` and
related modules.

### Build a firmware target

All `west build` invocations run from inside `zmk_exts/` and point at
this repo via `-DZMK_CONFIG` and `-DZMK_EXTRA_MODULES`:

```bash
cd zmk_exts

# Left half
west build -s zmk/app -d ../.build/cornix_left -b cornix_left -- \
    -DZMK_CONFIG="$PWD/../config" -DZMK_EXTRA_MODULES="$PWD/.."

# Right half
west build -s zmk/app -d ../.build/cornix_right -b cornix_right -- \
    -DZMK_CONFIG="$PWD/../config" -DZMK_EXTRA_MODULES="$PWD/.."

# Prospector dongle
west build -s zmk/app -d ../.build/prospector_dongle \
    -b xiao_ble/nrf52840/zmk -S studio-rpc-usb-uart -- \
    -DZMK_CONFIG="$PWD/../config" -DZMK_EXTRA_MODULES="$PWD/.." \
    -DSHIELD="cornix_prospector prospector_adapter" \
    -DCONFIG_ZMK_STUDIO=y
```

The resulting UF2 lands at `.build/<artifact>/zephyr/zmk.uf2`.

A `Justfile` is also included for the repo author's own workflow, but
the raw `west build` commands above are the source of truth and what
CI ends up running.

## Use the Cornix shield from another zmk-config

Add this repo as a west module:

```yaml
# config/west.yml
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
    - name: cornix-shield
      url-base: https://github.com/hitsmaxft
    - name: urob
      url-base: https://github.com/urob
    - name: englmaxi
      url-base: https://github.com/englmaxi
    - name: carrefinho
      url-base: https://github.com/carrefinho
  projects:
    - name: zmk
      remote: zmkfirmware
      revision: main
      import: app/west.yml
    - name: zmk-keyboard-cornix
      remote: cornix-shield
      revision: main
    - name: zmk-helpers
      remote: urob
      revision: main
    - name: zmk-dongle-display
      remote: englmaxi
      revision: main
    - name: prospector-zmk-module
      remote: carrefinho
      revision: feat/new-status-screens
```

Then mirror the entries in this repo's `build.yaml` into your own.

## About RGB

Cornix has two RGB LEDs per half, driven by PWM in the stock RMK
firmware. The `cornix_indicator` shield uses them as battery and
connection indicators, but full RGB underglow parity with the stock
firmware is not yet implemented. PRs welcome.

## TODO

- [x] 52-key full layout keymap (v2.0)
- [x] EC11 encoder support (v2.2)
- [x] No-SoftDevice flash layout (v2.3)
- [x] Prospector dongle support
- [x] Zephyr 4.1 / LVGL 9 (v2.7)
- [ ] RGB underglow parity with stock firmware

## Credits

- Cornix hardware: Jezail Funder
- Prospector ZMK module: [carrefinho](https://github.com/carrefinho/prospector-zmk-module)
- ZMK helpers: [urob](https://github.com/urob/zmk-helpers)

The maintainer also contributes to [RMK](https://rmk.rs/) — give it a
look if you want a Rust-based alternative to ZMK.
