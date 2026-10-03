[![License][license-badge]][license-url]

# sysbench — native Windows x64 port (fork)

This is a fork of [akopytov/sysbench](https://github.com/akopytov/sysbench)
that restores a **native Windows x64 build** via MinGW-w64/MSYS2 — no WSL,
no Linux subsystem, no virtual machine. Upstream dropped native Windows
support as of sysbench 1.0 and currently points Windows users to WSL
instead.

There's no binary package for this fork (yet): it's build-from-source
only, following the steps below. The result is a self-contained
`sysbench.exe` — it only links against standard Windows system DLLs
(`ntdll`, `kernel32`, `msvcrt`), so once built it can be copied anywhere
and run without MSYS2 on the `PATH`.

Everything else about sysbench itself — what it is, how to use it, its
license — is unchanged from upstream; see
[akopytov/sysbench](https://github.com/akopytov/sysbench) for that.

## Building on Windows

Native x64 build via [MSYS2](https://www.msys2.org/):

``` shell
# in a plain MSYS2 shell (not MINGW64) - installs the toolchain once
pacman -S --noconfirm mingw-w64-x86_64-toolchain autoconf automake \
  libtool make git pkg-config

# from here on, run everything in an MSYS2 "MINGW64" shell
git clone https://github.com/Atarit0/sysbench.git
cd sysbench
git checkout mingw64-port
./autogen.sh
./configure --without-mysql
make -j$(nproc)
```

`src/sysbench.exe` is the result. Tested with the `cpu` and `fileio`
benchmarks; `--without-mysql` skips the MySQL driver, which hasn't been
touched for MinGW and will very likely need its own round of fixes.

## How we got there

Upstream sysbench's Windows support predates MinGW-w64 having a real
pthreads implementation, and nobody has exercised that code path against
a current MinGW-w64 toolchain in years — it no longer builds out of the
box. Getting from "doesn't build" to a working `sysbench.exe` took five
separate fixes (one commit each, see the
[`mingw64-port`](https://github.com/Atarit0/sysbench/commits/mingw64-port)
branch history for the full detail on each):

1. **Concurrency Kit** (bundled dependency) only recognized `MINGW32*`/
   `MSYS_NT*` in its own `configure` script's OS detection; a 64-bit
   toolchain reports `MINGW64_NT-...`, which fell through to
   "unsupported" and aborted the build immediately.
2. Every file that did `#ifdef _WIN32 / #include "sb_win.h"` pulled in
   a legacy hand-rolled pthreads shim (`sb_win.h`, predating MinGW-w64's
   real `<pthread.h>`) *unconditionally* — it collided with the real
   pthreads headers MinGW-w64 already provides, which also get included
   right after it. Excluded MinGW from that guard everywhere it appears.
3. `fileio`'s `sb_open()` picked Win32 `CreateFile()` over POSIX
   `open()` based on a bare `#ifndef _WIN32` (true for MinGW too, which
   defines `_WIN32`) — needed `<windows.h>` (lost in the previous fix)
   and returned the wrong handle type besides. Flipped it to treat
   MinGW like POSIX, and added `pread`/`pwrite`/`fsync` emulation, since
   MinGW's C library doesn't provide any of the three.
4. `sb_threads.c`/`sb_util.c` call genuinely Windows-only API
   (`SwitchToThread`, `VirtualAlloc`, `GetSystemInfo`) that needs
   `<windows.h>` directly — plus a real, previously-dead bug: a
   `VirtualAlloc` branch assigned to a variable called `buffer` that
   doesn't exist (the function declares `buf`); harmless until a
   Windows toolchain finally tried to compile it.
5. `SIGALRM`/`alarm()`/`srandom()`/`random()` don't exist on Windows
   (no POSIX interval timers, no BSD rand extensions) — emulated or
   stubbed. Separately, the `-DDATADIR=\"...\"`/`-DLIBDIR=\"...\"`
   command-line defines never survived MSYS2's Make→shell invocation
   intact regardless of quoting style tried; dropped from the command
   line in favor of a `#ifndef DATADIR` fallback in `sysbench.h`.

None of this touches sysbench's actual benchmark logic — every fix is
either a compatibility shim for something Windows genuinely lacks, or
telling the preprocessor to treat MinGW like the POSIX host it mostly
is instead of like 2005-era MSVC.

## License

GPL-2.0-or-later, same as upstream — see [COPYING](COPYING). Original
copyright: MySQL AB (2004) and Alexey Kopytov (2004-2017).

[license-badge]: https://img.shields.io/badge/license-GPL--2.0--or--later-blue.svg
[license-url]: COPYING
