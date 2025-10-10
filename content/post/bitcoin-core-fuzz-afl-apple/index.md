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

```bash
brew install afl++
```

This comes with the AFL++ instrumentation compiler `afl-clang-fast++` (located in `/opt/homebrew/opt/afl++/bin`) and LLVM from Homebrew (`/opt/homebrew/Cellar/llvm`). Note that `afl-clang-lto` is [not available on macOS](https://github.com/AFLplusplus/AFLplusplus/blob/a3dbd38977fa1e87d29a9222aa7647422fdb0d43/docs/INSTALL.md?plain=1#L157).

**Note:** `afl-clang-fast++` is a wrapper around Homebrew's LLVM clang compiler (`/opt/homebrew/Cellar/llvm/<VERSION>/bin/clang++`). When you compile with it, it produces binaries with the required runtime instrumentation that enables the AFL++ fuzz engine to track code coverage and guide fuzzing.

## Configuration

Configure the fuzz build using `afl-clang-fast` (replacing `afl-clang-lto` in the command mentioned in [Bitcoin Core's fuzzing docs](https://github.com/bitcoin/bitcoin/blob/200150beba6601237036bc01ee10f6a0a2246c3d/doc/fuzzing.md?plain=1#L221-L224)):

```bash
cmake -B build_fuzz \
    -DCMAKE_C_COMPILER="/opt/homebrew/bin/afl-clang-fast" \
    -DCMAKE_CXX_COMPILER="/opt/homebrew/bin/afl-clang-fast++" \
    -DBUILD_FOR_FUZZING=ON
```

## Troubleshooting LLVM Issues

However, you may run into the same issue I did after attempting to build with `cmake --build build_fuzz`:

```bash
Undefined symbols for architecture arm64:
  "std::__1::__hash_memory(void const*, unsigned long)", referenced from:
      Arena::Arena(void*, unsigned long, unsigned long) in libbitcoin_util.a[32](lockedpool.cpp.o)
      ...
ld: symbol(s) not found for architecture arm64
```

This is caused by a mismatch: `afl-clang-fast++` compiles with Homebrew's LLVM headers (which declare `__hash_memory`), but the linker defaults to Apple's older system libc++ (which lacks this symbol).

Fix by explicitly using Homebrew's LLVM for both compilation and linking:

```bash
cmake -B build_fuzz \
    -DCMAKE_C_COMPILER="/opt/homebrew/bin/afl-clang-fast" \
    -DCMAKE_CXX_COMPILER="/opt/homebrew/bin/afl-clang-fast++" \
    -DBUILD_FOR_FUZZING=ON \
    -DCMAKE_CXX_FLAGS="-stdlib=libc++ -I/opt/homebrew/Cellar/llvm/<YOUR_VERSION>/include/c++/v1" \
    -DCMAKE_EXE_LINKER_FLAGS="-L/opt/homebrew/Cellar/llvm/<YOUR_VERSION>/lib/c++ -lc++ -lc++abi"
```

Replace `<YOUR_VERSION>` with your installed LLVM version (e.g., 21.1.2).

Now build:

```bash
cmake --build build_fuzz
```

## Running the Fuzzer

To run a specific fuzz target (I'm running the `cmpctblock` target introduced in [PR#33300](https://github.com/bitcoin/bitcoin/pull/33300)):

Create input and output directories:

```bash
mkdir -p fuzz-inputs/ fuzz-outputs/
```

Generate initial test input:

```bash
head -c 1000 /dev/urandom > fuzz-inputs/input.dat
```

Configure system for AFL++ (may require sudo):

```bash
sudo afl-system-config
```

Start fuzzing:

```bash
FUZZ=cmpctblock afl-fuzz -i fuzz-inputs -o fuzz-outputs -- build_fuzz/bin/fuzz
```

You should see AFL++ running successfully:

```bash
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

Using `AFL_DEBUG` and `AFL_NO_UI` environment variables provides debug logs in a more readable format for troubleshooting:

```bash
FUZZ=cmpctblock AFL_DEBUG=1 AFL_NO_UI=1 afl-fuzz -i fuzz-inputs -o fuzz-outputs -- build_fuzz/bin/fuzz
```

```bash
[+] Enabled environment variable AFL_DEBUG with value 1
[+] Enabled environment variable AFL_DEBUG with value 1
[+] Enabled environment variable AFL_NO_UI with value 1
afl-fuzz++4.33c based on afl by Michal Zalewski and a large online community
[+] AFL++ is maintained by Marc "van Hauser" Heuse, Dominik Maier, Andrea Fioraldi and Heiko "hexcoder" Eißfeldt
[+] AFL++ is open source, get it at https://github.com/AFLplusplus/AFLplusplus
[+] NOTE: AFL++ >= v3 has changed defaults and behaviours - see README.md
[+] No -M/-S set, autoconfiguring for "-S default"
[*] Getting to work...
[+] Using exploration-based constant power schedule (EXPLORE)
[+] Enabled testcache with 50 MB
[+] Generating fuzz data with a length of min=1 max=1048576
[*] Checking CPU scaling governor...
[!] WARNING: Could not check CPU min frequency
[+] Disabling the UI because AFL_NO_UI is set.
[+] You have 8 CPU cores and 3 runnable tasks (utilization: 38%).
[+] Try parallel jobs - see /opt/homebrew/Cellar/afl++/4.33c_1/share/doc/afl/fuzzing_in_depth.md#c-using-multiple-cores
[*] Setting up output directories...
[+] Output directory exists but deemed OK to reuse.
[*] Deleting old session data...
[+] Output dir cleanup successful.
[*] Validating target binary...
[+] Persistent mode binary detected.
[*] Scanning 'fuzz-inputs'...
[*] Creating hard links for all input files...
[+] Loaded a total of 1 seeds.
[*] Spinning up the fork server...
```

## Troubleshooting Fork Server Issues

With the fork server optimization enabled, you may face unexpected worker process terminations. I investigated the unexpected crashes caused by these terminations in the `cmpctblock` fuzz harness and documented my findings in [this GitHub comment](https://github.com/bitcoin/bitcoin/pull/33300#discussion_r2417231848).

To avoid such issues, [disable the fork server optimization](https://github.com/AFLplusplus/AFLplusplus/blob/474ff18ba2a7999a518a4d194fcd5a1f87c3625d/docs/INSTALL.md?plain=1#L168-L170):

```bash
FUZZ=cmpctblock AFL_NO_FORKSRV=1 afl-fuzz -i fuzz-inputs -o fuzz-outputs -- build_fuzz/bin/fuzz
```