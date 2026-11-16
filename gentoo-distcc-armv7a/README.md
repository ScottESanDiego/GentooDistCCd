# ARMv7 GCC and Clang distcc server

This image runs native host compilers that generate ARMv7 Linux hard-float
objects. GCC remains available through the existing crossdev toolchain; Clang
uses LLVM's ARM backend. No Go or Rust compilation service is included.

## Build and run

From this directory:

```sh
docker build --build-arg CLANG_SLOT=22 -t scotte/gentoo-distcc-armv7a:26.11.1 .
docker run --rm -p 3632:3632 scotte/gentoo-distcc-armv7a:26.11.1
```

Set `CLANG_SLOT` to the Clang major version installed on the ARM client. The
selected slot must be available in the synced Gentoo repository. Match the full
compiler version on both machines where possible, including GCC for GCC jobs.
Building LLVM from source can take substantial time and memory.
The final stage copies the cleaned filesystem so Portage downloads and
repository data are excluded from the shipped image's layers. The compiler
build layers remain available in Docker's local build cache.

The daemon accepts all client addresses by default. Publish its port only on
your trusted build network, or override the command with an appropriate
`--allow` range.

## ARM client setup

Install distcc and the matching native compilers on the ARM system. Add the
server to `/etc/distcc/hosts`, for example `192.168.1.10/8`, replacing the address
and job count for your server. Use ordinary distcc mode (no `,cpp`/pump mode).
Enable `FEATURES="distcc"` in `/etc/portage/make.conf`, preserving existing
features. Keep the existing GCC compiler settings for normal GCC builds.
GCC requests must use the matching `armv7a-unknown-linux-gnueabihf-` compiler
prefix; distcc normally adds the client's native GCC target prefix itself.

For packages that use Clang, ensure Portage's compiler wrappers route their
Clang invocations through distcc. One explicit per-package configuration is:

```sh
# /etc/portage/env/clang-distcc.conf
CC="distcc clang"
CXX="distcc clang++"
```

Select that environment for a Clang-compatible package in
`/etc/portage/package.env`, using a line such as
`category/package clang-distcc.conf`. Do not switch the entire system to Clang
just to use this server. Ebuilds that disable distcc or override compiler
selection may still compile locally.

The server supports `clang`, `clang++`, `clang-22`, `clang++-22`, and the same
names prefixed with `armv7a-unknown-linux-gnueabihf-` (version suffixes follow
`CLANG_SLOT`). Invoke compilers by these names rather than slot-specific absolute
paths. All these names target ARM; host-native Clang outside the daemon is
unmodified. Pass explicit CPU/FPU flags appropriate for your ARM client, and
avoid `-march=native` and `-mcpu=native`.

## Verify from the ARM client

Use a server-only host list and disable local fallback so failures are visible:

```sh
export DISTCC_HOSTS=192.168.1.10/8
export DISTCC_FALLBACK=0
export DISTCC_VERBOSE=1
printf 'int main(void) { return 0; }\n' > /tmp/distcc-check.c
printf 'int main() { return 0; }\n' > /tmp/distcc-check.cc
distcc clang -c /tmp/distcc-check.c -o /tmp/distcc-clang.o
distcc clang++ -c /tmp/distcc-check.cc -o /tmp/distcc-clangxx.o
readelf -h -A /tmp/distcc-clang.o /tmp/distcc-clangxx.o
clang /tmp/distcc-clang.o -o /tmp/distcc-clang-check
clang++ /tmp/distcc-clangxx.o -o /tmp/distcc-clangxx-check
/tmp/distcc-clang-check
/tmp/distcc-clangxx-check
distcc armv7a-unknown-linux-gnueabihf-gcc -c /tmp/distcc-check.c -o /tmp/distcc-gcc.o
distcc armv7a-unknown-linux-gnueabihf-g++ -c /tmp/distcc-check.cc -o /tmp/distcc-gxx.o
```

Check verbose output and container logs for remote compilation, and check that
`readelf` reports ARM objects and the expected ABI attributes. Repeat the Clang
checks with versioned names if your packages use them. Then test an actual
Clang-selected package with Portage. Linking, preprocessing, configure steps,
and tests still run on the ARM client. LTO, precompiled headers, and C++ modules
are outside this setup's validated compilation path.
