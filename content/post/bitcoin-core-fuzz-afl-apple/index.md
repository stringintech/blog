---
title: "Fuzzing Bitcoin Core using AFL++ on Apple Silicon"
date: 2025-09-28 14:25:00+0000
image:
tags:
  - Bitcoin
weight: 1
---

Steps to build and run a Bitcoin Core fuzz target on Apple Silicon using [AFL++](https://github.com/AFLplusplus/AFLplusplus) (tested on M1 and M3):

## Installation

First, install AFL++ via Homebrew:

```
brew install afl++
```

Which installs:


1) afl++ compilers:

```
ls /opt/homebrew/opt/afl++/bin | grep afl-clang

afl-clang
afl-clang++
afl-clang-fast
afl-clang-fast++
```

Note that `afl-clang-lto` is [not available on macOS](https://github.com/AFLplusplus/AFLplusplus/blob/a3dbd38977fa1e87d29a9222aa7647422fdb0d43/docs/INSTALL.md?plain=1#L157).

2) And LLVM:
```
ls /opt/homebrew/Cellar/llvm

# Shows your installed LLVM version (e.g., 21.1.2)
```

## Configuration

Since `afl-clang-lto` is not available on macOS, you can replace it with the `afl-clang-fast` variant from [this command](https://github.com/bitcoin/bitcoin/blob/200150beba6601237036bc01ee10f6a0a2246c3d/doc/fuzzing.md?plain=1#L221-L224) in the Bitcoin Core fuzzing documentation and configure your fuzz build using the command below:

```
cmake -B build_fuzz \
    -DCMAKE_C_COMPILER="/opt/homebrew/bin/afl-clang-fast" \
    -DCMAKE_CXX_COMPILER="/opt/homebrew/bin/afl-clang-fast++" \
    -DBUILD_FOR_FUZZING=ON
```

## Troubleshooting LLVM Issues

However, you may run into the same issue I did after attempting to build with `cmake --build build_fuzz`:

```
Undefined symbols for architecture arm64:
  "std::__1::__hash_memory(void const*, unsigned long)", referenced from:
      Arena::Arena(void*, unsigned long, unsigned long) in libbitcoin_util.a[32](lockedpool.cpp.o)
      Arena::alloc(unsigned long) in libbitcoin_util.a[32](lockedpool.cpp.o)
      ...
ld: symbol(s) not found for architecture arm64
clang++: error: linker command failed with exit code 1 (use -v to see invocation)
make[2]: *** [bin/fuzz] Error 1
make[1]: *** [src/test/fuzz/CMakeFiles/fuzz.dir/all] Error 2
make: *** [all] Error 2
```

The issue is a header/library version mismatch between two different LLVM toolchains:

- **AFL++ uses**: Newer LLVM headers (which declare the `__hash_memory` symbol)
- **macOS defaults to**: Apple's LLVM (~16.x) libc++ library (missing the `__hash_memory` symbol)

The `afl-clang-fast++` wrapper uses newer LLVM headers during compilation, but the linker still defaults to Apple's older system libc++ during linking.

**Note:** `afl-clang-fast` is a wrapper around the real clang compiler. When you compile with it, it produces binaries with the required runtime instrumentation that enables the AFL++ fuzz engine to track code coverage and guide fuzzing.

The solution that worked for me was to explicitly pass the required compiler and linker flags when configuring the CMake build to use the newer LLVM:

```
cmake -B build_fuzz \
    -DCMAKE_C_COMPILER="/opt/homebrew/bin/afl-clang-fast" \
    -DCMAKE_CXX_COMPILER="/opt/homebrew/bin/afl-clang-fast++" \
    -DBUILD_FOR_FUZZING=ON \
    -DCMAKE_CXX_FLAGS="-stdlib=libc++ -I/opt/homebrew/Cellar/llvm/<YOUR_VERSION>/include/c++/v1" \
    -DCMAKE_EXE_LINKER_FLAGS="-L/opt/homebrew/Cellar/llvm/<YOUR_VERSION>/lib/c++ -lc++ -lc++abi"
```

Replace `<YOUR_VERSION>` with your installed LLVM version (e.g., 21.1.2).

Now build:

```
cmake --build build_fuzz
```

## Running the Fuzzer

To run a specific fuzz target (I'm running the `cmpctblock` target introduced in [PR#33300](https://github.com/bitcoin/bitcoin/pull/33300)):

Create input and output directories:

```
mkdir -p fuzz-inputs/ fuzz-outputs/
```

Generate initial test input:

```
head -c 1000 /dev/urandom > fuzz-inputs/input.dat
```

Configure system for AFL++ (may require sudo):

```
sudo afl-system-config
```

Start fuzzing:

```
FUZZ=cmpctblock afl-fuzz -i fuzz-inputs -o fuzz-outputs -- build_fuzz/bin/fuzz
```

You should see AFL++ running successfully:

```
american fuzzy lop ++4.33c {default} (build_fuzz/bin/fuzz) [explore]
┌─ process timing ────────────────────────────────────┬─ overall results ────┐
│        run time : 0 days, 0 hrs, 0 min, 20 sec      │  cycles done : 0     │
│   last new find : none seen yet                     │ corpus count : 1     │
│last saved crash : none seen yet                     │saved crashes : 0     │
│ last saved hang : none seen yet                     │  saved hangs : 0     │
├─ cycle progress ─────────────────────┬─ map coverage┴──────────────────────┤
│  now processing : 0.0 (0.0%)         │    map density : 1.45% / 1.47%      │
│  runs timed out : 0 (0.00%)          │ count coverage : 1.11 bits/tuple    │
├─ stage progress ─────────────────────┼─ findings in depth ─────────────────┤
│  now trying : trim 4/4               │ favored items : 1 (100.00%)         │
│ stage execs : 87/250 (34.80%)        │  new edges on : 1 (100.00%)         │
│ total execs : 332                    │ total crashes : 0 (0 saved)         │
│  exec speed : 10.16/sec (zzzz...)    │  total tmouts : 0 (0 saved)         │
├─ fuzzing strategy yields ────────────┴─────────────┬─ item geometry ───────┤
│   bit flips : 0/0, 0/0, 0/0                        │    levels : 1         │
│  byte flips : 0/0, 0/0, 0/0                        │   pending : 1         │
│ arithmetics : 0/0, 0/0, 0/0                        │  pend fav : 1         │
│  known ints : 0/0, 0/0, 0/0                        │ own finds : 0         │
│  dictionary : 0/0, 0/0, 0/0, 0/0                   │  imported : 0         │
│havoc/splice : 0/0, 0/0                             │ stability : 99.08%    │
│py/custom/rq : unused, unused, unused, unused       ├───────────────────────┘
│    trim/eff : n/a, n/a                             │             [cpu: 24%]
└─ strategy: explore ────────── state: started :-) ──┘
```
