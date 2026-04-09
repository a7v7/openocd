# Getting Started

## Building on Windows (MSYS2 UCRT64)

### Prerequisites

Install the required libraries via MSYS2's UCRT64 environment:

```bash
pacman -S mingw-w64-ucrt-x86_64-libusb \
          mingw-w64-ucrt-x86_64-hidapi \
          mingw-w64-ucrt-x86_64-libftdi \
          mingw-w64-ucrt-x86_64-libjaylink
```

MSYS2 does not provide a prebuilt `libjim` package, so jimtcl must be built from the bundled submodule and installed before configuring OpenOCD. See [Building and Installing jimtcl](#building-and-installing-jimtcl) below.

These libraries enable the USB transport drivers:

| Package | Enables |
|---------|---------|
| `libusb` | Base USB support required by most adapters |
| `hidapi` | CMSIS-DAP v1 (HID-based) adapters |
| `libftdi` | FTDI MPSSE-based adapters (e.g. FTDI FT2232) |
| `libjaylink` | SEGGER J-Link adapters |

### Configure and Build

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

The resulting binary is `src/openocd.exe`.

### Configure Flag Reference

| Flag | Description |
|------|-------------|
| `--prefix=/ucrt64` | Install into the MSYS2 UCRT64 prefix |
| `--enable-jlink` | SEGGER J-Link (requires `libjaylink`) |
| `--enable-ftdi` | FTDI MPSSE adapters (requires `libftdi`) |
| `--enable-cmsis-dap` | CMSIS-DAP v1 HID adapters (requires `hidapi`) |
| `--enable-remote-bitbang` | Remote bitbang over TCP/UNIX socket |
| `--disable-doxygen-html` | Skip HTML API doc generation |
| `--disable-doxygen-pdf` | Skip PDF API doc generation |

All `--enable-*` adapter flags default to `auto`, so they will be included automatically if the required library is detected. Pass `--enable-<name>=yes` to force-fail the build if the library is missing rather than silently skipping the driver.

### Cleaning for a From-Scratch Rebuild

`make distclean` removes all compiled objects, generated `Makefile`s, and configure output (`config.h`, `config.status`, `config.log`), leaving the source tree as if `configure` had never been run:

```bash
make distclean
```

Then re-run from the configure step:

```bash
./configure --prefix=/ucrt64 \
            --enable-jlink \
            --enable-ftdi \
            --enable-cmsis-dap \
            --enable-remote-bitbang \
            --disable-doxygen-html \
            --disable-doxygen-pdf
make
```

If you also need to regenerate the `configure` script itself (e.g. after modifying `configure.ac` or updating the `jimtcl` submodule), run `./bootstrap` before `./configure`.

## Building and Installing jimtcl

OpenOCD requires `libjim >= 0.79`. MSYS2 does not ship this package, so build and install it from the `jimtcl/` git submodule.

First ensure the submodule is populated:

```bash
git submodule update --init jimtcl
```

Then configure, build, and install into the UCRT64 prefix:

```bash
cd jimtcl
./configure --prefix=/ucrt64 --with-ext=json --minimal --disable-ssl --disable-docs
make
make install
cd ..
```

The `--minimal` flag omits optional extensions (IPv6, math, Readline) that OpenOCD does not need. `--with-ext=json` is required because OpenOCD uses Jim's JSON extension.

After installation, `pkg-config --modversion jimtcl` should report the installed version. The OpenOCD `./configure` step will then find `libjim` automatically via pkg-config without any extra flags.
