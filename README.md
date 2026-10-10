<p align="center"><img src="docs/mirlo64.png" width="480" alt="MIRLO64"></p>

# MIRLO64

MIRLO64 runs Nintendo 64 games from their `.z64` ROMs on the [Analogue Pocket](https://www.analogue.co/pocket), as an openFPGA core. It is an original system-on-chip written in SystemVerilog for the Pocket's Cyclone V FPGA: a MIPS CPU of the VR4300 class, an RSP and an RDP of its own, and the video, audio, controller and save hardware around them. No Nintendo code or data is included: the game's own code and its own libultra run on the CPU, its microcode runs on the RSP, and a small runtime does what is not in hardware.

This is the **`tlb` branch**: version 0.1 (`main`) with the VR4300's real TLB in the game CPU and a video output that is the N64 VI's own 640 x 240 field. It is experimental. Read [Status and limitations](#status-and-limitations) before expecting a particular game to work.

> **Status of this branch: no bitstream of it that closes timing has been built yet.** The release v0.1 and the fitter seeds listed below belong to `main`. The real TLB and the new scan-out change the CPU's and the video's logic and their timing paths, and the seeds that close for `main` say nothing about this tree. Everything below about this branch's behaviour comes from the docs and the simulations, not from a Pocket.

## What is in it

* **Game CPU** (`rtl/mips/mips_core.sv`): a 32-bit MIPS of the VR4300 class, big-endian as the N64, at 93.75 MHz, with a single-precision FPU (`fpu_n64.sv`), direct-mapped 32 KiB instruction and 32 KiB data caches, and the VR4300's 64-bit integer instructions (the registers, `ld`/`sd`, the 64-bit ALU, shifts and `dmult` in hardware; `ddiv` and anything else left out is done by the runtime). It has the VR4300's own TLB (32 entries of two pages each, 4 KiB to 16 MiB, searched by a scanner with micro-TLBs in front): `tlbwi`, `tlbwr`, `tlbr` and `tlbp` are the CPU's, and a TLB exception is the game's own handler's, so the runtime has no TLB tables and patches nothing of the game's libultra. A fetch from a mapped page whose entry is cacheable goes through the I-cache ([docs/mips.md](docs/mips.md), "The TLB"). `main` (v0.1) has translation tables in block RAM instead, filled by the runtime.
* **RSP** (`rtl/rsp`): the N64's scalar and vector units, running the game's own microcode ([docs/rsp.md](docs/rsp.md)).
* **RDP** (`rtl/rdp`): draws the N64's own command stream: triangles walked with the N64's coverage rules, the 4 KiB TMEM with its three-point filter, the colour combiner and blender, and a 16-bit Z buffer ([docs/rdp.md](docs/rdp.md)).
* **Video (VI)** (`rtl/soc/rcp_vi.sv`, `rtl/soc/n64_video.sv`): the VI's registers, the anti-alias, divot and dedither filters, gamma, and the scan-out to the Pocket's scaler: one 640 x 240 field as the VI sends it (its 640-clock line, 240 lines, progressive; 59.94 Hz NTSC or 50 Hz PAL), with the picture placed in it by the VI's window and scale registers ([docs/resolution.md](docs/resolution.md)). `video.json` declares that one 640 x 240, 4:3 mode.
* **Audio (AI)** (`rtl/soc/rcp_ai.sv`): the AI's DMA and queue, resampled to the Pocket's 48 kHz.
* **MI, PI, SI and the PIF** (`rtl/soc/rcp_mi.sv`, `rcp_si.sv`): the registers and the DMA engines are hardware; the PIF's joybus commands (controllers, accessories, EEPROM) are done by the runtime from the Pocket's controllers.
* **Controllers, accessories and saves**: up to four controllers (2-4 through the dock), a Controller Pak, a Transfer Pak that reaches a Game Boy cartridge in the Pocket's own slot, and EEPROM, SRAM and FlashRAM saves, with a save-type database on the SD card (`n64savedb.txt`).
* **Runtime** (`lang/n64rt`, built as `n64rt.sys`, loaded by the core as data slot 3): starts the ROM with the memory set up as the console's boot code leaves it, then stays resident at the CPU's *monitor level* to do what is not in hardware: the RCP registers that are not hardware blocks, the PIF's commands, the FlashRAM's command set, the controllers' mapping. The game runs with its own libultra, unmodified.

## Status and limitations

What the repository records, and where it was checked:

* **Simulation.** [docs/mips.md](docs/mips.md) reports Verilator runs of the whole SoC (`sim/mips_soc`) with real ROMs. The ROMs it names are 1080, Banjo-Kazooie, Banjo-Tooie, Donkey Kong 64, GoldenEye, Mario Kart 64, Majora's Mask, Ocarina of Time, Pilotwings 64, Pokemon Stadium, Star Fox 64 (both revisions), Super Mario 64 (including the European one) and Wave Race 64. What it describes of them is bounded: Super Mario 64 reaches its title, file select and intro and creates and re-reads its EEPROM save; Star Fox 64 reaches its logos and title; Pokemon Stadium reaches its title and the Game Pak Check; Ocarina of Time reaches its title frames and creates its SRAM save; the others "reached the same screens" with the 8 MiB of RDRAM that the core gives every game (Donkey Kong 64, Majora's Mask and Banjo-Tooie need it). The docs do not claim that these games are playable.
* **Hardware.** The docs do not keep a per-game record of runs on a Pocket, and nothing of this branch is recorded as run on one: not the real TLB, not the 640 x 240 slot, and not the 50 Hz PAL output (the docs say "not yet seen on a Pocket"). The bridge side of `core_top` (a game loaded from the card, a save written back) is covered by no simulation. See the status note at the top.
* **What this branch changes, as simulated.**
  * *GoldenEye*: its 440-pixel-wide framebuffer is shown whole, by the VI's own scale registers (X_SCALE 0x2C0, Y_SCALE 0x580: 330 rows in 237 lines); `sim/vi_filter` checks that picture against a model. Its boot, which runs from a 4 MiB window at `0x7000_0000` and pages 4 KiB pairs at `0x7F00_0000` with its own refill handler, runs on the real TLB (about 17.9 M CPU cycles from its first fetch in the window to its first refill, in the docs' simulation). This does not say the game is playable.
  * *Hi-res and serrated pictures* (Pokemon Stadium's menus, Donkey Kong 64's logos, 640-wide modes): a 640-wide picture is shown at its full width. A serrated 480-row picture is shown as 640 x 240 **progressive**, each line the average of a pair of rows; it is not sent as 480i, because the Pocket's handling of interlaced slots is not documented and an earlier attempt did not display correctly on the Pocket ([docs/hires_output_notes.md](docs/hires_output_notes.md)).
  * *PAL windows*: the European Super Mario 64 (237 lines) and Mario Golf (a window taller than the field, fitted to it) are shown whole.
* **Video limits that remain.** The VI's anti-alias, divot and dither filters are off for hi-res pictures. Reading a serrated picture takes 36.8 MB/s of the SDRAM (14.7 % of its cycles); `tb_vi` shows no pixel lost with the arbiter's worst-case grant for the SoC's six masters, and one more master would lose some. A window the VI makes taller than the field is squeezed into it.
* **The real TLB.** A mapped page is physically the N64's RDRAM only: a game that maps the cartridge or the RCP's registers through the TLB gets the RDRAM wrapped. KSEG2, KSEG3 and `0x2000_0000`-`0x5FFF_FFFF` are not a game's to map (the n64-systemtest cases that do fail). The area and timing of the TLB are not fitted.
* **ROMs.** Up to 48 MiB. The larger ones (the 64 MiB cartridges) are not supported in this build: the part past 48 MiB is dropped.
* **RSP and RDP.** The RSP does not implement LHV, LFV, LWV, SHV, SFV, SWV, VRNDP, VRNDN, VMULQ and VMACQ (the docs say no game they looked at uses them). The RDP draws only into 16-bit colour images, does not decode YUV, and skips "set key" and "set convert". Neither is cycle-accurate to the console (the RSP's DMA moves 4 bytes a cycle where the N64's moves 8, for one); the docs give CPU times from simulation but no frame rates.
* **Console behaviour not reproduced.** [docs/mips.md](docs/mips.md) lists what still fails of n64-systemtest's groups: the MI's repeat mode, the cartridge's write latch, some PI_STATUS bits, DMAs with a misaligned PI_DRAM_ADDR, and, in `main`, the TLB group.
* **Accessories.** No Rumble Pak: the Pocket and its dock give a core nothing to vibrate. Only controller 1 can have a pak. The Transfer Pak is checked in simulation against a model of a Game Boy cartridge.
* **Saves.** The EEPROM's size (4 or 16 Kbit) cannot be told from the game's requests, so it comes from `n64savedb.txt`; a game that is not in the file (or no file) gets 4 Kbit. The type `sram96k` is not supported.
* **Timing is tight.** The design fills about 95 % of the device's ALMs (17.4-17.5 thousand of 18,480, per docs/mips.md), and whether a build closes timing depends on the fitter's seed: see [Building](#building).

## Requirements

* An Analogue Pocket (firmware 1.1 or later, per `core.json`) and an SD card. A Dock for controllers 2-4 and docked pads.
* Nintendo 64 ROMs of your own, in `.z64` format. None is provided.
* Optional: a Game Boy or Game Boy Color cartridge in the Pocket's slot, for the Transfer Pak.

## Installing

1. Download `tortuga.Mirlo64_<version>.zip` from the [releases](https://github.com/tortugaveloz/MIRLO64/releases). (The release v0.1 is `main`'s; there is none of this branch.)
2. Unzip it onto the **root** of the SD card, merging with the folders already there. It adds only its own files:

   ```
   Assets/mirlo/common/          n64rt.sys (the runtime), n64savedb.txt, PUT_ROMS_HERE.txt
   Cores/tortuga.Mirlo64/        the core: mirlo.rev (the bitstream), core.json, ...
   Platforms/mirlo.json          the platform, in the Console category
   Platforms/_images/mirlo.bin   its image
   ```

3. Put your ROMs in `Assets/mirlo/common/`. They must be `.z64` (big-endian: the file starts with the bytes `80 37 12 40`); the runtime stops with "the ROM is not a .z64 ROM" otherwise. A `.v64` (byte-swapped in pairs) or `.n64` (each 32-bit word reversed) ROM can be converted with this script (`python3 conv.py in.v64 out.z64`):

   ```python
   import sys
   src, dst = sys.argv[1:3]
   d = bytearray(open(src, "rb").read())
   if d[:4] == bytes.fromhex("80371240"):
       pass                                    # already .z64
   elif d[:4] == bytes.fromhex("37804012"):    # .v64
       d[0::2], d[1::2] = d[1::2], d[0::2]
   elif d[:4] == bytes.fromhex("40123780"):    # .n64
       d[0::4], d[1::4], d[2::4], d[3::4] = d[3::4], d[2::4], d[1::4], d[0::4]
   else:
       sys.exit("not a Nintendo 64 ROM in a known byte order")
   open(dst, "wb").write(d)
   ```

4. On the Pocket: **openFPGA > Console > Mirlo**, then pick a ROM. To update, unzip a newer release the same way.

**The bitstream and `n64rt.sys` of this branch are a pair.** COP0 register 23 reads a capability word (`'T' 'L' 'B'` in bits 31:8; bit 0 the monitor level, bit 1 a real TLB, bit 2 cached fetches from mapped pages). The runtime checks bit 1 as it starts and stops with a message on a bitstream that does not have the real TLB (`main`'s and earlier), and an older runtime on this core finds no translation tables and does not work either. A release of this branch carries both files (`mirlo.rev` and `n64rt.sys`) and they are updated together; do not mix them with `main`'s.

## Using the core

### Settings

The core's menu (`interact.json`; the Pocket's own Controls menu still remaps each button first). Defaults in bold.

| setting | values | what it does |
|---|---|---|
| Crop Vertical | None, **8 lines**, 12 lines | blanks that many lines at the top and bottom of the picture |
| Crop Right | None, **1 column** | blanks the picture's last column |
| Controller 1 pak | **None**, Controller Pak, Transfer Pak | the accessory in controller 1 |
| Stick | **D-pad**, L stick, R stick | what drives the N64's analog stick |
| D-pad | **None**, D-pad, L stick, R stick | what drives the N64's D-pad |
| Z button | None, **L**, R, ZL, ZR | the Pocket's key for Z |
| L button | **None**, L, R, ZL, ZR | the Pocket's key for the N64's L |
| R button | None, L, **R**, ZL, ZR | the Pocket's key for the N64's R |
| C buttons | **None**, Right stick | the right stick as the C buttons, besides the face buttons |
| Start button | None, **Start**, Select+Start | the Pocket's key for Start |
| Modifier | **None**, L, R, ZL, ZR | held, turns the face buttons X, B, Y, A into C-up, C-down, C-left, C-right and the D-pad into half a stick |

The face buttons are A and B as they are, and C-left on Y, C-right on X, C-down on Select (C-up is through the Modifier). Controllers 2-4 come and go with the dock: a pad plugged in while a game runs answers from the game's next poll. The layout is described in docs/mips.md, "The core's settings for the controller".

### Saves

The Pocket writes them back to `Saves/mirlo/common/` when you quit the core. A file only appears once the game has saved.

| file | what |
|---|---|
| `<game>.sav` | the cartridge's save: EEPROM (512 bytes, or 2 KiB for the 16 Kbit games), SRAM (32 KiB) or FlashRAM (128 KiB). The runtime sees which the game uses from its first access; for the EEPROM's size it reads `n64savedb.txt` |
| `<game>.mpk` | the Controller Pak (32 KiB), when the setting is on; a new one is formatted |

`Assets/mirlo/common/n64savedb.txt` (448 rows, built from public sources by `tools/build_savedb.py`) lists cartridge IDs with their save type; only the EEPROM's size is acted on today (which games have the 16 Kbit chip). To add a game, edit the file by hand: the format is in docs/mips.md, "The cartridge's EEPROM size and the save-type database".

### JTAG UART

The core's log and the serial boot of a program of your own go over a JTAG UART, with a USB Blaster on the Pocket's JTAG port and `tools/jtag/`. **The shipped bitstream has no JTAG UART**: `projects/mirlo.qsf` defines `MIRLO_NO_JTAG_UART=1`, which leaves the UART out (the runtime's log goes nowhere). To use it, remove that line and rebuild; the fit and its timing have not been checked without it.

```bash
tools/jtag/jtag_program_and_run.sh <bitstream.sof> <program.bin> <log> [seconds]
```

This programs the FPGA with `quartus_pgm` (set `QUARTUS_PGM` if it is not under `~/altera/25.1std`), uploads the program through the boot ROM's serial boot over the UART and logs its output; it needs `openocd`. `tools/jtag/jtag_boot.sh <program.bin>` uploads and drops into a console; `tools/jtag/jtag_uart_relay.py` alone gives the UART on a PTY. Use a genuine USB Blaster: clones have damaged Pockets.

## Repository layout

| directory | contents |
|---|---|
| `rtl/soc/` | the SoC (`mirlo_mips.sv`) and its parts: the CPU's system (`mips_sys.sv`, `cpu_port.sv`), the SDRAM controller (`sdr_*.sv`), the RCP blocks (`rcp_mi`, `rcp_vi`, `rcp_ai`, `rcp_si`, `rcp_dpc_view`), the scan-out (`n64_video.sv`), the registers (`mirlo_regs.sv`, generated decode in `mirlo_regs.svh`), the Game Boy cartridge port (`gb_cart.sv`), the JTAG UART |
| `rtl/mips/` | the game CPU `mips_core.sv` and its FPU `fpu_n64.sv`; `mips_geom.sv` and `mips_lite.sv` belong to the earlier system (see below) |
| `rtl/rsp/`, `rtl/rdp/` | the RSP and the RDP |
| `rtl/mrdp/`, `rtl/geom/`, `rtl/vpu/`, `rtl/audio/` | the earlier system's rasterizer, geometry core, vector unit and audio core: not in the bitstream |
| `target/pocket/`, `platform/pocket/` | the openFPGA glue: top level (`core_top.sv`), PLLs, data loader, APF bridge |
| `projects/` | the Quartus project (`mirlo.qpf`, `mirlo.qsf`, `mirlo.sdc`) |
| `lang/mips/` | the SDK's base: toolchain flags (`mips.mk`), picolibc and compiler-rt (`lib.mk`), program rules (`program.mk`), linker files, headers (`include/`, with the generated `csr.h`), the boot ROM (`boot/`), CPU and SoC tests (`test/`) |
| `lang/n64rt/` | the runtime that becomes `n64rt.sys` (`loader/main.c`, `n64rt_*.c`) and its tests |
| `lang/c/` | C examples and the earlier system's 3D SDK, firmware and model (see below) |
| `sim/` | Verilator benches and models |
| `tools/` | register-map generator, packaging, JTAG tools, the save-type database builder, `area2/` (fit and timing scripts) |
| `pkg/pocket/` | the files that go on the SD card: core definition, platform, `n64savedb.txt` |
| `docs/` | design notes |
| `vendor/` | git submodules: picolibc and compiler-rt, for the C library |

## Building

### Tools

* **Quartus Prime Lite 25.1std** with Cyclone V support. `fit.sh` below looks for it under `~/altera/25.1std/quartus/bin` (`QUARTUS_BIN` to change).
* **A GCC for MIPS and its binutils.** `lang/mips/mips.mk` names them with `MIPS_CC` (default `mips-linux-gnu-gcc`) and `MIPS_BIN` (the binutils' prefix, default `mips-linux-gnu-`; a path prefix works), and the tools must be on `PATH`. Ubuntu's `gcc-mips-linux-gnu` and `binutils-mips-linux-gnu` are the defaults. The flags choose the byte order and ABI.
* **Python 3** with **meson** and **ninja** (picolibc is built with them).
* **Verilator**, for the simulations only.

### Steps

Run from the repository root.

```bash
git clone --recursive https://github.com/tortugaveloz/MIRLO64.git     # --recursive: vendor/ holds picolibc and compiler-rt
cd MIRLO64
export MIPS_CC=mips-linux-gnu-gcc MIPS_BIN=mips-linux-gnu-            # or your toolchain's

# 1. The C library for the game CPU: picolibc, compiler-rt, the UART runtime -> lang/mips/build/game/
make -C lang/mips -f lib.mk VARIANT=game

# 2. The boot ROM -> lang/mips/boot/build/boot.hex
make -C lang/mips/boot

# 3. The RSP's reciprocal ROM (git ignores *.hex, so it is not in a clone)
python3 sim/rsp/make_div_rom.py rtl/rsp/rsp_div_rom.hex

# 4. The images the bitstream is built with: rtl/soc/init/boot.hex and rtl/soc/mirlo_mips_init.svh
#    (both generated, ignored by git; the .svh holds absolute paths, so run this again if the tree moves)
python3 tools/mips_inits.py

# 5. The bitstream (see "Fitter seeds" below), then bit-reverse it as the Pocket wants it
(cd projects && quartus_sh --flow compile mirlo)
python3 tools/package_bitstream.py projects/output_files/mirlo.rbf mirlo.rev

# 6. The runtime -> lang/n64rt/build/n64rt.bin (about 54 KB), which the core loads as n64rt.sys; built for this branch's bitstream
make -C lang/n64rt
cp lang/n64rt/build/n64rt.bin n64rt.sys

# 7. The SD-card zip (both files from this tree: they are a pair)
python3 tools/make_release.py mirlo.rev n64rt.sys tortuga.Mirlo64_0.1.zip
```

Notes on the steps:

* The register map (`tools/mirlo_regs.py`) generates `rtl/soc/mirlo_regs.svh` and `lang/mips/include/generated/{csr,soc,mem}.h`. Those files are committed, so a build does not need it; run `python3 tools/mirlo_regs.py` after changing the map, and then rebuild everything that includes the headers (the boot ROM, the runtime, the bitstream). The registers' addresses can change between revisions: take them from `csr.h`.
* Neither the geometry core's nor the audio core's firmware is part of this bitstream or of `n64rt.sys` (the runtime is built with `RSP=1`; `lang/n64rt/Makefile` fixes it). The only generated memory images the bitstream is built with are the boot ROM and the RSP's reciprocal ROM: after changing the boot ROM, repeat steps 2, 4 and 5.
* `lang/n64rt/Makefile` checks that `n64rt.bin` stays under 960 KiB: the ROM is loaded at 1 MiB, above the program.
* `lang/mips/lib.mk` with `VARIANT=lite` is only for the earlier system's geometry and audio firmware.
* Quartus warns that two files under `analogue-pocket-utils` (listed in `target/pocket/core.qip`) are missing. They are PSRAM modules; the PSRAMs are not used.

### Fitter seeds

The design fills most of the device and its timing is tight: a fit takes about ten minutes, and whether it closes timing depends on the seed. **`quartus_sh --flow compile mirlo` on the tree as it is builds the right design, but with the project's seed, 34, which according to docs/mips.md ("What fits") does not close: a hold violation of 0.03 ns on the CPU's clock in the fast corner.** Two things about the options:

* The design's feature macros (`MIRLO_RSP`, `MIRLO_MON`, `MIRLO_VI`, `MIRLO_TLB_RAM`, `MIRLO_CPU_CLK`, ...) are no longer read by the RTL: the features are always in and the CPU's clock is fixed by the PLL (93.75 MHz). The macros the RTL still reads are `MIRLO_CPU_IC_LOG2`, `MIRLO_CPU_DC_LOG2` and `MIRLO_RSP_LANES` (their defaults, 10, 10 and 4, are the release's), `MIRLO_NO_FPG` and `MIRLO_NO_JTAG_UART` (set in `projects/mirlo.qsf`, with the few others the project needs). `tools/area2/fit.sh` still appends the old macros to the project it compiles: they are inert, and so are the same names in docs/mips.md.
* What decides the result is the seed. The seeds that closed timing for v0.1, in `main`, are **12, 27 and 5** (docs/mips.md's table of earlier fits also closes with 20, 3 and 41, and fails with 34 and 9). **None is known for this branch yet**: its TLB and video logic are new, so seeds have to be tried again and the result checked, not assumed. Either edit `set_global_assignment -name SEED` in `projects/mirlo.qsf` and compile, or let the script do it:

```bash
tools/area2/fit.sh <tag> <seed>      # e.g. tools/area2/fit.sh s12 12
```

`fit.sh` copies the tree into `s_<tag>/` (so edits cannot reach a run in progress; it and the symlink `projects_a2_<tag>` it leaves in the root are yours to delete), sets the seed there and starts the compile in the background: the log is `projects_a2_<tag>.compile.log` and the bitstream `s_<tag>/projects/output_files/mirlo.rbf`. Several tags can run in parallel. When a compile ends, `python3 tools/area2/closure.py s_<tag>/projects/output_files` gives its worst slack in all four timing corners (a negative number: it does not close); `tools/area2/summ.sh` gives the area. Then continue from `package_bitstream.py` in the steps above. A bitstream that did not close timing may misbehave in ways that look like a bug in the design.

## Simulation and tests

The SoC benches need Verilator (`VERILATOR` and `VERILATOR_ROOT` name it, defaulting to `~/local/usr/...`), and the C tests the MIPS toolchain of the steps above (and the libraries of step 1).

| what | where |
|---|---|
| the whole SoC against an SDRAM chip model, with a game's ROM | `sim/mips_soc`: `./build.sh soc`, then `obj_soc/tb_soc <program.bin> -c <Mcycles> -o <dir>`; `ROM=<file.z64>` is the game, `SLOT<n>=<file>` the other data slots, the options are in `tb_soc.cpp` and docs/mips.md, "Simulation" |
| SDRAM controller, PHY and arbiter | `sim/mips_soc`: `./build.sh sdr` |
| CPU, RCP, TLB, video, controller and save tests, as programs for the SoC | `lang/mips/test`: `make -C lang/mips/test <test>` (the list is in its Makefile) builds `build/<test>.bin`, run on `tb_soc`; `tools/area2/soc_regress.sh` runs a set. The real TLB's: `tlbreal_test` (all seven page sizes, the micro-TLBs, the exceptions, Index / Random / Wired, caches, LL/SC, `0x6000_0000`), `tlbmon_test`, `tlbgame_test` (the game's own handlers) and `tlbpg_test` (cached fetches from mapped pages); the video field's: `hires_test`, `vwin_test` |
| 8 MiB of RDRAM at every block | `sim/mips_soc/ram8_check.sh` |
| the game CPU in lockstep with a reference model of the N64 | `sim/mips` (needs the model, which is not in this repository: see the script); `sim/mips/tlb_rom.sh OUT.z64` builds a directed TLB program (page sizes, ASID and global, invalid and clean pages, refills, delay slots, Random / Wired) for it |
| RSP RTL against its model, DMA queue | `sim/rsp`: `build_rtl.sh`, `build_dma.sh`; `gen_rand_task.py` writes random tasks |
| RDP RTL against the C model, random and real task dumps | `sim/rdp`: `build_rtl.sh`; `tools/area2/rdp_regress.sh` runs the random cases; `sim/rdp_dither` checks the dither tables |
| VI filters and the 640 x 240 scan-out against a model | `sim/vi_filter`: `./regress.sh` builds `obj_vi/tb_vi` and runs its 42 cases (320, 440, 384, 292 and hi-res pictures, windows, steps, the crop options, PAL, the TV system changing between fields, the SDRAM loaded) and `vifit_test.c`, the host test of the runtime's window fit |
| AI queue, interrupt and reader | `sim/ai`: `build.sh`, `obj_ai/tb_ai`; `tools/ai_check.py` compares the hardware's audio with the software algorithm's |
| the Game Boy cartridge port | `sim/gb_cart/run.sh <rom.gb>` (`lang/n64rt/test/make_gb.py` makes one) |
| runtime, host side (no toolchain: `gcc` and Python) | `python3 lang/n64rt/test/pad_test.py` (the controller mapping over every setting), `lang/n64rt/test/savedb_host.c` (the save-type lookup; the build line is in its header) |
| runtime, on the SoC | `lang/n64rt/test`: the Controller Pak, Transfer Pak and SI tests need a decompilation of libultra (`ULTRALIB`), which is not in the repository; `rcp_bench.mk` and `tlb_bench.mk` build games that check the console's answers for the RCP's registers and run the TLB instructions as a game |
| the earlier system's units | `tools/mrdp_ucode.py test 2000`, `sim/mrdp`, `sim/vpu`, `sim/audio`, `sim/geom_full`, `make hosttest` in `lang/c/geom` |

## Programs of your own

`lang/mips` is a small SDK for programs that run on the game CPU, in C: `mips.mk` (flags: `-march=vr4300 -mabi=32 -EB -msingle-float`; doubles in software), `lib.mk`, `program.mk`, the linker files and C runtime, `include/` with the generated register headers. Examples are in `lang/c/examples` (`helloworld`, `fungus`, `dhrystone`, `keytest`); the tests of `lang/mips/test` are programs of this kind too. See [lang/README.md](lang/README.md) and [lang/c/README.md](lang/c/README.md).

In this SoC the Pocket loads the game ROM at 1 MiB and the runtime at address 0, and the boot ROM jumps to `0x4000_0000`. A program of your own therefore takes the runtime's place: `lang/mips/test/padtest.c` is run that way (docs/mips.md, "The core's settings for the controller": copy the program as `Assets/mirlo/common/n64rt.sys`; the core still wants some ROM in the game slot, which is ignored). Comments in the Makefiles (`program.mk`, the examples) that say "data slot 0" describe an earlier memory layout. With a bitstream that has the JTAG UART, `tools/jtag` uploads a program over it instead. This path has not been rebuilt and run for this README.

The project began as a different system, "Mirlo": a geometry core, an audio core and its own rasterizer (MRDP) beside the game CPU, with a 3D SDK (`lang/c/game`, `lang/c/geom`, `lang/c/audio`). The RTL (`rtl/mips/mips_geom.sv`, `mips_lite.sv`, `rtl/mrdp`, `rtl/geom`, `rtl/vpu`, `rtl/audio`) and the SDK stay in the tree, with their benches and [docs/geometry_core.md](docs/geometry_core.md) and [docs/mrdp.md](docs/mrdp.md), but **the SoC no longer instantiates those cores: programs written for them do not run on the current bitstream**. The RDP keeps MRDP's pixel pipeline, and `lang/c/mrdp/mrdp.c` is still its C model; those documents and `lang/c/game/README.md` describe the earlier system and are out of date in places.

## Documentation

* [docs/mips.md](docs/mips.md): the SoC: memory map, the game CPU, its real TLB, the RCP blocks in hardware, the monitor and the runtime, saves and the save-type database, controllers, timing and area. It is a design log, and some passages name macros the RTL no longer reads.
* [docs/rsp.md](docs/rsp.md) and [docs/rdp.md](docs/rdp.md): the RSP and the RDP, with how each is verified.
* [docs/resolution.md](docs/resolution.md): the 640 x 240 field, the VI's window and scales in it, serrated pictures, PAL windows, and how to change the resolution. Its first section describes an earlier core.
* [docs/hires_output_notes.md](docs/hires_output_notes.md): what was tried for 640 x 480 interlaced output on the Pocket.
* [docs/control.md](docs/control.md): the register blocks (the map itself is `tools/mirlo_regs.py`).
* [docs/geometry_core.md](docs/geometry_core.md), [docs/mrdp.md](docs/mrdp.md): the earlier system.

## Credits and license

MIRLO64 is released under the [Apache License 2.0](LICENSE). It includes and builds on third-party work under its own licenses (the Pocket's platform glue, Analogue's APF files, picolibc, compiler-rt and some ported code), listed in [NOTICE](NOTICE). Nintendo 64 is a trademark of Nintendo; this project is not affiliated with or endorsed by Nintendo or Analogue.
