# F4 build path — STM32F4 panda firmware (dos board)

This branch (`f4-revive`) restores the **STM32F4 firmware build path** that
upstream `commaai/panda` removed in commit
[`1ce986f7`](https://github.com/commaai/panda/commit/1ce986f7) "bye bye f4"
(Aug 2025).

The F4 firmware targets the **dos board** — the internal panda inside the
**comma 3** device. It is still required by users running Tesla Pre-AP
(Model S 2014) and any other vehicle that relies on the C3 internal panda
for safety-critical communication with the car.

## Why this matters

`comma 3` is officially deprecated, but it is *not* end-of-life for the
community:

- Tesla Pre-AP (Model S 2014) integrations like NAP / Tinkla still ship on C3.
- Some open-source forks (NotAutopilot, MagZu/openpilot, etc.) continue to
  develop against the platform.
- The F4 firmware *cannot* be cross-substituted by H7 — `dos.h` board
  definition is STM32F413-specific and the boot/USB/CAN drivers differ.

After `1ce986f7`, the only way to flash the C3 internal panda was via
**pre-built binary blobs committed by hand to forks**. That is fragile and
non-reproducible. This branch makes the F4 build path **first-class
buildable from source again**.

## What's in this branch

Three logical commits on top of `90387239` (MagZu's `HW_TYPE_DOS` host
mapping, last commit with C3 device awareness):

1. **`69ef7bf6` Resurrect `board/stm32f4` + `dos.h` + SConscript F4 target**
   — restores files from `1ce986f7^` (= `3dc21386`, last commit with F4),
   re-adds `base_project_f4` and `build_project("panda", base_project_f4, ...)`,
   plus three build-fixes applied on top:

   - `_estack` overflow fix: F407 linker had top-of-SRAM at `0x2001FFFC`
     (128K). F413 has 256K. Without the fix, `.data + .bss` spilled past
     the stack and corrupted it during startup zero-init.
   - `HEALTH_PACKET_VERSION 16 → 18`: matches what the modern panda
     Python library and pandad expect (`sound_output_level` replacing
     `fan_stall_count`).
   - Tesla Model S `0x348` GTW_status ignition-CAN-detect:
     `ignition_can_hook` in `board/drivers/can_common.h` did not cover
     the Tesla Model S 2014 ignition message. Without this, `ignition_can`
     stayed `false` even with the car ON, and pandad force-set
     `safety_mode = NO_OUTPUT`.

2. **`0d616c08` Track overlay dependencies** — `certs/`, `crypto/`, and
   `opendbc/safety/can.h` were present in upstream pre-bye-bye-f4 but not
   tracked in `90387239`. Build needs them on disk; this commit adds them
   to source control instead of relying on out-of-band copies.

3. **`F4_REVIVE.md` (this file) + `.github/workflows/build-f4.yml`** —
   documentation + CI so anyone can verify and consume builds without
   running the toolchain locally.

## Building locally

Requirements:

```bash
sudo apt-get install -y gcc-arm-none-eabi scons python3-pip
pip3 install pycryptodome 'opendbc @ git+https://github.com/commaai/opendbc.git@master'
```

Then from this branch:

```bash
scons -Q -j$(nproc) board/obj/panda.bin.signed board/obj/bootstub.panda.bin
```

Outputs:

```
board/obj/panda.bin.signed    # main firmware (debug-signed via certs/debug)
board/obj/bootstub.panda.bin  # F4 bootstub
board/obj/version             # gitversion baked into firmware
```

The firmware's `gitversion` string contains the active commit short hash, so
two builds from different commits are intentionally byte-different. Builds
of the *same* commit on the same toolchain version are deterministic.

## Building via CI (recommended)

Every push and PR on this repo triggers `.github/workflows/build-f4.yml`,
which installs the toolchain on `ubuntu-latest`, runs the build, and
uploads `panda-f4-firmware-<sha>.zip` as a GitHub Actions artifact (30-day
retention). Reviewers and downstream forks can grab a verified build
directly from CI without setting up a local toolchain.

## Deploy notes (out of scope for this branch, FYI)

The CI build links against vanilla `commaai/opendbc master`, so the output
is the *upstream-compatible* baseline. If you need a NAP-specific build
(e.g. with `tesla_preap` safety mode), you must build against the
NAP-fork opendbc — that is downstream of this branch.

For flashing the C3 internal panda when `panda.flash()` hangs on SPI,
the working path is DFU via STM32 ROM bootloader. See the NAP / Tinkla
deploy scripts for details — out of scope for this branch.

## What this branch is **not**

- Not a vehicle for safety-mode changes (those go in `opendbc`).
- Not a NAP-specific firmware (no `tesla_preap` C code is added here).
- Not a replacement for the pre-built binary blob workflow currently used
  by forks — those forks can continue to ship pre-built blobs on their own
  branches; this branch just makes the *source path* available again.

## Acknowledgements

- `commaai/panda` for the original F4 stack (pre-`1ce986f7`).
- `MagZu/panda` for keeping C3 alive with pre-built blobs + host-side
  `HW_TYPE_DOS` after F4 was removed upstream.
- `daggerhashimoto` and other NotAutopilot contributors for maintaining
  the broader C3 fork ecosystem.
