# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build System

OpenOCD uses GNU Autotools (Autoconf/Automake/Libtool). The typical build sequence:

```bash
./bootstrap          # Run autotools (aclocal, libtoolize, autoconf, autoheader, automake)
./configure          # Generate Makefiles (many --enable-* flags for adapters/features)
make                 # Build; output binary is src/openocd (or src/openocd.exe on Windows)
make install         # Install to system
```

For out-of-tree builds (recommended):

```bash
mkdir build && cd build
../configure
make
```

### Windows (MSYS2 UCRT64)

Install dependencies first:

```bash
pacman -S mingw-w64-ucrt-x86_64-libusb \
          mingw-w64-ucrt-x86_64-hidapi \
          mingw-w64-ucrt-x86_64-libftdi \
          mingw-w64-ucrt-x86_64-libjaylink
```

Then configure and build:

```bash
./bootstrap
./configure --prefix=/ucrt64 \
            --enable-jlink \
            --enable-ftdi \
            --enable-cmsis-dap \
            --enable-remote-bitbang \
            --disable-doxygen-html \
            --disable-doxygen-pdf
make
```

To do a full clean rebuild: `make distclean`, then re-run `./configure` and `make`. Run `./bootstrap` first if `configure.ac` or the `jimtcl` submodule changed.

Documentation:

```bash
make info    # Texinfo manual
make doxygen # API documentation (output in doc/doxygen/html/)
```

## Linting and Style Checks

```bash
tools/checkpatch.sh          # Validate the most recent commit
tools/checkpatch.sh HEAD~3.. # Validate last 3 commits
```

`checkpatch.sh` wraps `tools/scripts/checkpatch.pl` (derived from the Linux kernel). Requires: `perl`, `python`, `python-ply`, `pymarkdownlnt`. Configuration is in `.checkpatch.conf`. Key rules:
- Tabs for indentation, tab width = 4
- Max line length: 120 characters
- UTF-8, LF line endings (enforced by `.editorconfig`)
- Must be endian-clean (no assumptions about host byte order)

## Tests

Tests live in `testing/` and use a Tcl framework. There is no automated unit test suite runnable with `make check` under normal builds; testing is primarily done by running OpenOCD against real or simulated hardware with Tcl scripts.

## Commit Message Format

```
subsystem/area: short description

Longer explanation if needed, max 75 chars per line. Use 'Link:' on its
own line for URLs.

Signed-off-by: Your Name <email@example.com>
```

Examples of subsystem prefixes: `flash/nor/atsame5:`, `target/cortex_m:`, `jtag/drivers/ftdi:`, `server/gdb:`, `doc:`

Commits must be submitted to Gerrit for review, not pushed directly:

```bash
git push review HEAD:refs/for/master
```

## Architecture Overview

OpenOCD is organized in clear layers:

```
Clients: GDB (port 3333), Telnet (port 4444), Tcl (port 6666)
                        |
              src/server/   — protocol servers (gdb_server.c, telnet_server.c, tcl_server.c, rtt_server.c)
                        |
              src/helper/   — command framework, logging, configuration parser, utilities
                        |
    +-----------+----------+-----------+
    |           |          |           |
src/jtag/   src/target/ src/flash/  src/transport/
adapters,   ARM, MIPS,  NOR/NAND    JTAG, SWD, SPI
TAP state   RISC-V,     flash       protocol
machine     Xtensa,     drivers     abstraction
            ARC, etc.
```

**Key subsystems:**

- `src/helper/` — Foundation: command registration/dispatch (`command.c`), logging (`log.c`), configuration file parsing (`configuration.c`), Jim Tcl integration, binary buffers, CRC, time utilities.

- `src/jtag/` — JTAG core and 50+ adapter drivers. The `drivers/` subdirectory has one file per adapter family (ftdi, stlink, cmsis_dap, jlink, etc.). `core.c` manages the JTAG TAP state machine and scan chain queuing.

- `src/transport/` — Abstracts the physical transport (JTAG vs SWD vs SPI). Adapters register supported transports; targets select one.

- `src/target/` — Debug target implementations. Each architecture has its own file(s): `arm/` (Cortex-A/R/M, ARM7/9/11), `riscv/`, `xtensa/`, `espressif/`, `mips32.c`, `arc.c`, etc. Core abstractions: `target.c` (register access, memory read/write, step/resume/halt), `breakpoints.c`, `algorithm.c`.

- `src/flash/` — Flash drivers split into `nor/` (50+ chips) and `nand/`. New NOR drivers implement the `struct flash_driver` interface.

- `src/server/` — Network servers. `gdb_server.c` (~3K lines) implements the GDB remote serial protocol. `tcl_server.c` exposes the command interface over TCP.

- `src/rtos/` — RTOS-aware debugging (FreeRTOS, ThreadX, uCOS, etc.). Hooks into the target layer to provide thread-aware backtraces.

- `tcl/` — Tcl configuration scripts for boards (`tcl/board/`), interfaces (`tcl/interface/`), and targets (`tcl/target/`). These are the `.cfg` files users pass with `-f`.

- `jimtcl/` — Git submodule: embedded Jim Tcl interpreter used for all scripting.

**Entry points:**
- `src/main.c` — Minimal CLI wrapper, calls `openocd_main()`
- `src/openocd.c` — Core initialization, config file loading, event loop

**Command registration pattern:** Subsystems register commands with `COMMAND_REGISTRATION` macros and `register_commands()`. Handlers use `COMMAND_HANDLER` / `COMMAND_HELPER` macros with `CMD_ARGC`/`CMD_ARGV`.

**New documentation** for commands must be added to `doc/openocd.texi` in the same commit that adds the command.
