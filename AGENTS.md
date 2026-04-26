# Repository Guidelines

This repository contains a ZMK configuration for the Keyboardio Preonic.

## Layout

- `config/keyboardio_preonic.keymap` is the primary keymap.
- `config/west.yml` pins the ZMK and Keyboardio module dependencies used by west.
- `build.yaml` defines the GitHub Actions build matrix.
- `boards/shields/` is reserved for local shield files if needed.

## Editing

- Keep keymap edits focused in `config/keyboardio_preonic.keymap`.
- Preserve the existing layer names and formatting style unless a broader keymap cleanup is requested.
- Do not commit west checkout or build output directories such as `.west/`, `build/`, `modules/`, `zephyr/`, `zmk/`, or `keyboardio-preonic-zmk-module/`.
- Treat `config/west.yml` changes as dependency changes. Commit them separately from keymap-only edits.

## Validation

- For keymap-only changes, inspect the diff before committing.
- When dependencies or board configuration change, run a local ZMK build or rely on the GitHub Actions workflow.

## Local Build

Do not run `west update` directly in this repository. This repository is also a
ZMK extra module and has a tracked `zephyr/module.yml`, so a west workspace in
the repo root will collide with generated checkout directories. Build in a
separate temporary west workspace and point `ZMK_EXTRA_MODULES` back to this
repository.

Known-good local build setup on macOS:

```sh
brew install ninja

rm -rf /tmp/keyboardio-preonic-zmk-local
mkdir -p /tmp/keyboardio-preonic-zmk-local/config
cp -R config/. /tmp/keyboardio-preonic-zmk-local/config/

cd /tmp/keyboardio-preonic-zmk-local
west init -l config
west update --fetch-opt=--filter=tree:0
west zephyr-export

python3 -m venv .venv
. .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install west -r zephyr/scripts/requirements.txt -r zmk/app/scripts/requirements.txt
python -m pip install 'setuptools<81'

ZEPHYR_TOOLCHAIN_VARIANT=gnuarmemb \
GNUARMEMB_TOOLCHAIN_PATH=/opt/homebrew \
west build -s zmk/app -d build/keyboardio_preonic -b keyboardio_preonic -- \
  -DZMK_CONFIG=/Users/chae.suyeong/Projects/private/zmk/keyboardio-preonic-zmk-config/config \
  -DZMK_EXTRA_MODULES=/Users/chae.suyeong/Projects/private/zmk/keyboardio-preonic-zmk-config
```

The `setuptools<81` pin is intentional. The pinned ZMK/nanopb tooling imports
`pkg_resources`; newer setuptools versions may remove it and fail with
`ModuleNotFoundError: No module named 'pkg_resources'`.

After the first build, rebuild from the temporary workspace with:

```sh
cd /tmp/keyboardio-preonic-zmk-local
. .venv/bin/activate
ZEPHYR_TOOLCHAIN_VARIANT=gnuarmemb \
GNUARMEMB_TOOLCHAIN_PATH=/opt/homebrew \
west build -d build/keyboardio_preonic
```

Expected firmware outputs:

- `/tmp/keyboardio-preonic-zmk-local/build/keyboardio_preonic/zephyr/zmk.uf2`
- `/tmp/keyboardio-preonic-zmk-local/build/keyboardio_preonic/zephyr/zmk.hex`

## Bootloader Mode

Reliable Keyboardio Preonic bootloader entry:

1. Unplug the USB cable.
2. Hold the `HYPER` key.
3. Plug the USB cable back in while still holding `HYPER`.
4. Release `HYPER` after the board enters bootloader mode.

In bootloader mode, macOS should expose:

- A volume named `PREONICBOOT`
- A serial device like `/dev/cu.usbmodem2023201`

Do not rely on a keymap `&bootloader` binding unless that binding is already
known to be flashed on the board.

## Flashing

Preferred flashing path on this Mac is serial DFU via `adafruit-nrfutil`.
UF2 copy may fail because macOS can mount `PREONICBOOT` as read-only even
though the media itself is not read-only.

Install the DFU tool in the build venv and create a DFU package:

```sh
cd /tmp/keyboardio-preonic-zmk-local
. .venv/bin/activate
python -m pip install adafruit-nrfutil

adafruit-nrfutil dfu genpkg \
  --dev-type 0x0052 \
  --application build/keyboardio_preonic/zephyr/zmk.hex \
  build/keyboardio_preonic/zephyr/zmk-dfu.zip
```

Enter bootloader mode, find the serial port, then flash:

```sh
ls /dev/cu.*

adafruit-nrfutil --verbose dfu serial \
  --package build/keyboardio_preonic/zephyr/zmk-dfu.zip \
  -p /dev/cu.usbmodem2023201 \
  -b 115200
```

Use the actual `/dev/cu.usbmodem*` path shown on the machine. A successful
flash ends with `Device programmed.` The bootloader volume and serial port
usually disappear after the new firmware activates.

If `PREONICBOOT` is mounted read-write, copying the UF2 can also work:

```sh
cp build/keyboardio_preonic/zephyr/zmk.uf2 /Volumes/PREONICBOOT/
sync
```

If macOS reports `Read-only file system`, switch to serial DFU. Avoid spending
time on `mount_msdos`, raw `/dev/rdisk*`, or `mtools` workarounds on this Mac;
they were blocked by macOS permissions during testing.

## Git

- Commit keymap changes separately from generated files and dependency updates.
- Before committing, check `git status --short --branch` and `git diff --cached --name-only`.
