# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

RGBtoHDMI is bare-metal firmware for a Raspberry Pi Zero (also Pi 2/3/4) plus a hat carrying a small Xilinx CPLD. Together they sample "digital" (TTL) or analog RGB/YUV video from retro computers (it started with the BBC Micro and Electron, and now covers many other machines) and output it over HDMI. There is no OS: `kernel_main` in `src/rgb_to_hdmi.c` is the entry point after the Pi GPU firmware boots `kernel*.img`. User documentation is in the GitHub wiki.

## Building

You need the `arm-none-eabi-gcc` cross toolchain and CMake. Builds run **inside `src/scripts/`**, which is the CMake build directory (`../` is the source dir). Generated files there are gitignored.

```sh
cd src/scripts
./clobber.sh                 # wipe CMake cache/outputs (do this when switching targets)
./configure_rpi.sh           # or configure_rpi4.sh / configure_rpiA.sh / configure_rpiA4.sh
make -j                      # produces kernelrpi.img (etc.)
./configure_rpi.sh -DDEBUG=1 # debug build: serial logging at 115200 baud
./release.sh                 # builds all 4 targets x {normal,debug} and zips a release; refuses if the tree is dirty
```

The build targets are chosen by toolchain file (`src/toolchain-arm-none-eabi-*.cmake`):
- `rpi` builds `kernelrpi.img` (Pi Zero/1/2/3, GPU capture)
- `rpi4` builds `kernelrpi4.img` and defines `RPI4`
- `rpiA` / `rpiA4` define `USE_ARM_CAPTURE` (the ARM does the sampling itself, the legacy path). They are booted via `configA.txt`, which the user swaps in with `Use_ARM_Capture.bat`. GPU capture is preferred.

Code is compiled with ARMv6 C flags but ARMv7 ASM flags. One binary serves several Pi models, and runtime detection sets `BIT_RPI234`.

`version.sh` runs on every build and writes `gitversion.h`. Version strings get a `?` suffix when tracked files are modified.

**VideoCore GPU code:** `src/videocore.s` is assembled with the bundled `tools/vasm` by `src/scripts/buildvc.sh`, which regenerates `videocore.asm`/`videocore.lst`/`videocore.c` (an `xxd -i` blob). The normal CMake build does **not** run this, so after editing `videocore.s` you must run `buildvc.sh` and commit the generated files.

The repo has no tests or linter. To verify a change, build it and run it on hardware. `src/scripts/Profiles/*/_Test` holds test profiles.

Adding a new source file means adding it to the explicit list in `src/CMakeLists.txt`. The file glob there names every file, so nothing is picked up automatically.

## Architecture

**Capture pipeline.** The CPLD samples the video at a multiple of the source pixel clock. The Pi generates that clock via GPCLK/PLL, calibrated in `calibrate_sampling_clock`. The CPLD presents packed pixel data on GPIOs and toggles PSYNC. Then either:
- **GPU capture** (default): `videocore.s` runs on a VideoCore VPU core, polls the GPIOs, and hands sample words to the ARM through the mailbox registers (`GPU_COMMAND`/`GPU_DATA_*`, offsets in `defs.h`), or
- **ARM capture** (`USE_ARM_CAPTURE`): the ARM reads the GPIOs directly.

`rgb_to_fb.S` is the hand-written ARM assembly frame loop. It waits for sync, picks a per-line capture routine from the `capture_line_*_table` jump tables, and writes into the multi-buffered framebuffer. The `capture_line_*.S` files are specialised inner loops for each bit depth (1/3/6/9/12-bit input → 4/8/16bpp output) and mode (normal, double, half, odd/even, Mode 7 teletext, NTSC artifact colour, Atari XL, C64, …). They share macros in `macros.S`. These loops are cycle-critical, so be careful about instruction counts and register use.

**The `r3` control word.** The C code and the assembly communicate through a 32-bit flags word, kept in `r3` inside the assembly. Its bits (`BIT_*`) and return codes (`RET_*`) are defined in `src/defs.h`. Many `BITDUP_*` bits reuse the same bit positions for a different meaning inside the inner loops, so check `defs.h` before adding or reusing a bit. `defs.h` is included from both C and assembly.

**`capture_info_t` (`capinfo`)** describes the current capture: geometry, bpp, sample width, pitch, framebuffer pointers, chosen capture function, and so on. `geometry.c` fills it from the active profile's geometry params. The CPLD driver's `update_capture_info` adjusts it. The assembly consumes it.

**CPLD abstraction (`src/cpld.h`).** `cpld_t` is a vtable with these implementations:
- `cpld_rgb.c`: 6/12-bit RGB TTL and analog, plus YUV analog via the frontend
- `cpld_simple.c`
- `cpld_atom.c`
- `cpld_yuv.c`
- `cpld_null.c`: unprogrammed CPLD

`cpld_init` in `rgb_to_hdmi.c` reads the CPLD version register and selects the driver by `DESIGN_*` ID. Each driver supplies its own UI params, calibration, and info pages. The CPLD itself is flashed from the SD card (`.xsvf` under `src/scripts/cpld_firmware/`) by the Xilinx JTAG player in `src/jtag/`. The CPLD sources are the `vhdl*/` directories (Xilinx ISE projects). The prebuilt `.xsvf` there are copied into `cpld_firmware` for releases.

**Main loop.** `rgb_to_hdmi_main` (in `rgb_to_hdmi.c`) loops over: set up the profile, init the framebuffer and HDMI mode, call the assembly frame capture, then act on the `RET_*` flags it returns (mode change, button presses, sync/interlace changes). HDMI clock genlocking to the source frame rate is done in `recalculate_hdmi_clock*`. The second core (`USE_MULTICORE`) is used for OSD and other background work.

**OSD / UI (`src/osd.c`, largest C file).** This file holds the on-screen menu system (menu tables, feature params, info and help pages), button handling, and profile and config persistence. The UI works with `param_t` arrays exposed by `geometry.c`, the active CPLD driver, and `osd.c`'s own `features[]`. Each param's `property_name` is the key used in saved profile files.

**Profiles and SD-card layout.** `src/scripts/` is also the template for the SD card:
- `config.txt`/`configA.txt`: Pi firmware boot config, per-model `kernel=` selection
- `default_config.txt`
- `Profiles/<CPLD type>/<Manufacturer>/<Machine>.txt`: plain `key=value` files, e.g. `sampling=`/`geometry=` comma lists matching the param order, with numbered variants like `sampling2=` for alternate modes
- `Resolutions/`
- `firmware/`: Pi GPU blobs

`filesystem.c` (on FatFS in `src/fatfs/`) loads and saves these. Changing the order or meaning of `param_t` entries in `geometry.c` or the CPLD drivers changes how existing profile files are interpreted.

## Other directories

- `kicad*/`, `Kicad*/`: PCB designs for the hat and the various adapters and buffers
- `contrib/`: third-party contributions
- `tools/vasm`: VideoCore assembler binary used by `buildvc.sh`
