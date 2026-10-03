[![License][license-badge]][license-url]

# sysbench — native Windows x64 port (fork)

This is a fork of [akopytov/sysbench](https://github.com/akopytov/sysbench)
that restores a **native Windows x64 build** via MinGW-w64/MSYS2 — no WSL,
no Linux subsystem, no virtual machine. Upstream dropped native Windows
support as of sysbench 1.0 and currently points Windows users to WSL
instead.

This is an unofficial, community fork: it is not affiliated with or
endorsed by upstream, and the changes here have not been submitted to
or reviewed by the upstream maintainer. Treat it as experimental.

There's no binary release for this fork (yet): it's build-from-source
only, following the steps below.

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

The real binary is `src/.libs/sysbench.exe` — `src/sysbench.exe` is
only a libtool wrapper that launches it. The binary imports
`libwinpthread-1.dll` from the MinGW-w64 toolchain (besides the standard
Windows `kernel32.dll` and `msvcrt.dll`), so to run it outside an MSYS2
shell, copy `/mingw64/bin/libwinpthread-1.dll` next to it.

## What has been tested

- **Run on Windows x64:** the `cpu` and `fileio` benchmarks.
- **Compiled, only briefly smoke-run:** `memory`, `threads`, `mutex`.
- **Not built at all:** the MySQL driver (excluded via `--without-mysql`)
  and the PostgreSQL driver (off by default). Neither has been touched
  for MinGW, so the bundled `oltp_*` database benchmarks are not
  available in this build.

## Known issues

- **Crash on exit.** Every run currently segfaults during shutdown, after
  the results have been printed (exit status 139 / `0xC0000005`). On
  MinGW, `sb_memalign()` takes its `VirtualAlloc()` branch, but callers
  release that memory with `free()`. The printed results are complete,
  but the exit status is not usable in scripts yet.
- `alarm()` is stubbed out as a no-op, so the watchdogs that use it
  (thread-init timeout, and the `--time` + `--timeout` hard stop) never
  fire on Windows.
- `random()`/`srandom()` are mapped to `rand()`/`srand()`. MSVCRT's
  `rand()` only returns 15 bits, so the per-thread RNG seeds have less
  entropy than on Linux.
- The `-DDATADIR`/`-DLIBDIR` removal (fix 5 below) applies to every
  platform on this branch, not only Windows: bundled Lua scripts are
  looked up relative to the current directory instead of the install
  prefix.

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

None of this changes the benchmark workloads themselves. Every fix is
either a compatibility shim for something Windows genuinely lacks, or
telling the preprocessor to treat MinGW like the POSIX host it mostly
is instead of like 2005-era MSVC. The side effects of those shims are
listed under [Known issues](#known-issues).

## License

GPL-2.0-or-later, same as upstream — see [COPYING](COPYING). Original
copyright: MySQL AB (2004-2006) and Alexey Kopytov (2004-2018), as
stated in the individual source file headers. The changes in this fork
are distributed under the same license.

[license-badge]: https://img.shields.io/badge/license-GPL--2.0--or--later-blue.svg
[license-url]: COPYING
