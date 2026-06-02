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

## Battery calibration

The battery percentage is computed **on each keyboard half** from its own
ADC voltage divider (voltage → % happens in the peripheral firmware via
ZMK's lithium-ion curve). The dongle only receives the final, already
clamped, integer percentage over BLE — it has no access to the raw
voltage. **Calibration therefore cannot be done on the dongle; it must be
done in the keyboard-half firmware, independently for left and right.**

### Where the knobs live

Each half has its own `full-ohms` value:

| Half  | File                                          | Current value | Notes                              |
| ----- | --------------------------------------------- | ------------- | ---------------------------------- |
| Left  | `boards/jzf/cornix/cornix_left.dts`  | `2783000`     | ~-0.8% vs nominal (trimmed, see below) |
| Right | `boards/jzf/cornix/cornix_right.dts` | `2806000`     | Nominal (reference half)               |

Nominal hardware value: `2000000 + 806000 = 2806000` (`output-ohms` stays
`2000000`). The shared default is defined in
`boards/jzf/cornix/nrf_e73.dtsi`; the board files above override it.

### How to adjust

`full-ohms` scales the reported voltage linearly:

- **Lower** `full-ohms` → the half reports a **lower** voltage / lower %.
- **Higher** `full-ohms` → the half reports a **higher** voltage / higher %.

To shift a half's reading by a known amount, scale `full-ohms` by the
ratio of (desired mV / currently-reported mV) and rebuild + reflash that
half. Example: reading is 34 mV too high at 4226 mV →
`2806000 × (4192 / 4226) ≈ 2783000`.

### Why left is trimmed

At a simultaneous full charge the two halves measured:

- Left  ≈ **4.23 V** (read ~34 mV / ~0.8% high → sat above the 4.20 V
  clamp → stuck at 100%).
- Right ≈ **4.19 V** (just below the clamp → tracked normally).

Both cells are effectively full; the difference is within normal resistor
/ charge-IC tolerance, but it was enough to pin the left half at 100%.
Trimming the left `full-ohms` down ~0.8% aligns it with the right so both
track together.

### Re-measuring (diagnostic trick)

ZMK clamps anything ≥ 4200 mV to 100%, which hides the real voltage near
full charge. To turn the on-screen % into a coarse voltmeter, temporarily
**lower** `full-ohms` in `nrf_e73.dtsi` (e.g. to `2500000`) so a full cell
maps to ~3.74 V instead of clamping, rebuild both halves, read the two
numbers off the dongle, then back out the voltage:

```
mv_reported = (pct + 459) * 7.5            # invert ZMK's curve
v_adc       = mv_reported / (full / output)   # here full/output = 1.25
v_battery   = v_adc * (2806000 / 2000000)     # nominal divider ratio
```

This is the method used to derive the values above. Revert the diagnostic
change afterwards. (The `diag/battery-unclamp` branch keeps an example.)

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

### Incremental rebuild

After the first `west build`, editing a `.conf`, `.overlay` or source
file does **not** require a fresh `west build`. Re-run the build with
CMake instead — it re-runs the devicetree / Kconfig generation and
recompiles only what changed, which is much faster:

```bash
cd zmk_exts
cmake --build ../.build/prospector_dongle
```

### macOS: git in the dev shell

The Nix dev shell (`mkShellNoCC`) does **not** ship `git`, so it falls
back to the Xcode `/usr/bin/git` stub. Under some shells that stub
reports `error: tool 'git' not found`, which makes west fail with
`cannot import contents of app/west.yml`. If you hit that, put the
Command Line Tools git first on `PATH` before building:

```bash
export DEVELOPER_DIR=/Library/Developer/CommandLineTools
export PATH="/Library/Developer/CommandLineTools/usr/bin:$PATH"
```

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
- Upstream zmk-keyboard-cornix repo: [hitsmaxft](https://github.com/hitsmaxft/zmk-keyboard-cornix)
