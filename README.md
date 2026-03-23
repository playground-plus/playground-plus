```
 ╔══════════════════════════════════════════════════════════════╗
 ║                                                              ║
 ║     ██████╗  ██████╗ ██████╗ ██╗     ██╗   ██╗███████╗       ║
 ║     ██╔══██╗██╔════╝ ██╔══██╗██║     ██║   ██║██╔════╝       ║
 ║     ██████╔╝██║  ███╗██████╔╝██║     ██║   ██║███████╗       ║
 ║     ██╔═══╝ ██║   ██║██╔═══╝ ██║     ██║   ██║╚════██║       ║
 ║     ██║     ╚██████╔╝██║     ███████╗╚██████╔╝███████║       ║
 ║     ╚═╝      ╚═════╝ ╚═╝     ╚══════╝ ╚═════╝ ╚══════╝       ║
 ║                                                              ║
 ║            v2026.1.0 - by Tony Mattke aka tonhE              ║
 ║                                                              ║
 ╚══════════════════════════════════════════════════════════════╝
```
<p align="center">
  <img src="https://img.shields.io/badge/version-2026.1.0-blue?style=flat-square" alt="Version 2026.1.0" />
  <img src="https://img.shields.io/badge/GCC-10%2B-green?style=flat-square" alt="GCC 10+" />
  <img src="https://img.shields.io/badge/platform-Linux%20x86__64-lightgrey?style=flat-square" alt="Linux x86_64" />
  <img src="https://img.shields.io/badge/warnings-0-brightgreen?style=flat-square" alt="Zero Warnings" />
</p>

# Playground Plus v2026.1.0

> [!NOTE]
> This repository is not an official repository maintained by the original Playground+ developers. It is a community effort to modernize and preserve the PG+ codebase. The companion [playground-plus-patches](https://github.com/playground-plus/playground-plus-patches) repository is also unofficial.

**The classic talker server, modernized for 2026.**

Playground Plus is a multi-user chat server ("talker") with a rich history stretching back to the early 1990s. Originally built on the EW-Too codebase, it evolved through Summink and Playground 96 into Playground+, which became one of the most widely deployed talker platforms of its era.

This release brings PG+ into the modern age. It builds cleanly on current 64-bit Linux systems with zero errors and zero warnings, fixes critical security vulnerabilities, and replaces deprecated system APIs - all while preserving the talker experience that operators and users know.

---

## What's New in v2026.1.0

**Released 8th February 2026**

### Security Fixes

- **Password hashing upgraded from DES to SHA-512** via `crypt()`, with backwards-compatible login for existing DES hashes and an automatic upgrade prompt for users
- **Command injection vulnerability fixed** in the email welcome feature - `system()` replaced with `fork()`/`execlp()` so crafted email addresses cannot execute shell commands
- **Buffer overflow protection** across the codebase - `sprintf()` replaced with `snprintf()` in admin, commands, glue, plists, and room code
- **Intercom hardening** - buffer overflows patched, `strcpy` replaced with `strncpy`, division-by-zero guard added, `vsprintf` replaced with `vsnprintf`, and non-null-terminated packet data fixed
- **Format-string vulnerabilities** — `printf(stack)` and similar calls allowed format-string attacks. Changed to `printf("%s", stack)` across multiple files (17+ locations).
- **Command injection in email** — `system()` call in `commands.c` passed user-controlled data (`email`, `inet_addr`) into a shell command without sanitization.
- **`vsprintf()` stack buffer overflows** — Unbounded `vsprintf()` into 150-byte buffers replaced with `vsnprintf()` throughout.
- **`strcpy()` buffer overflows** — Numerous unbounded `strcpy()` calls into fixed-size buffers replaced with bounds-checked alternatives.


### Critical Fixes

- **100% CPU busy-spin** — `select()` had a zero timeout, causing the main loop to spin continuously. Added a 200ms timeout for proper pacing.
- **64-bit pointer truncation** — Pointer-to-int casts throughout the codebase silently truncated addresses on 64-bit systems, causing crashes and memory corruption. Fixed across 10+ files by using proper pointer subtraction and `%p`/`%zu` format specifiers.
- **Hardcoded ROOT path** — Boot would fail immediately due to a wrong hardcoded install path in `root.h`.
- **Deprecated `sigpause(0)`** — Legacy BSD call in the main event loop did nothing on modern glibc; removed.

### Network Modernization

- **`gethostbyname`/`gethostbyaddr` replaced with `getaddrinfo`/`getnameinfo`** in both the talker and intercom server - the old functions are removed in some modern libc implementations
- **`select()` replaced with `poll()`** to remove the `FD_SETSIZE` 1024 file descriptor limit
- **Byte-at-a-time socket reads replaced with 4KB bulk buffered reads** for a massive performance improvement


### Intercom Subsystem (23 bugs found, disabled by default)

- **`SIGCHLD` handler called `fatal_error()`** — Intercom crashed every time a child process exited normally. Replaced with proper `waitpid()` reaping.
- **Broken `calloc` macro** — A trailing semicolon in the `#define` broke all `if/else` blocks using `calloc`.
- **Network data never null-terminated** — `strcpy()` on raw packet data read past buffer boundaries. Replaced with `memcpy` + explicit termination.
- **6 `vsprintf()` stack overflows** — All replaced with `vsnprintf()`.
- **Unix socket path overflow** — Unbounded `strcpy()` into 108-byte `sun_path`.
- **Wrong socket address family** — Used `hp->h_addrtype` instead of `AF_INET`.


### Compiler & Build Fixes

- Builds cleanly on **GCC 13.3** with `-Wall` - zero warnings across all four binaries (talker, angel, ident_server, intercom)
- Fixed `-Wrestrict` warnings from `sprintf()` writing into its own source buffer
- Fixed `-Wstringop-truncation` warnings with explicit null terminators after `strncpy()` calls
- Added `IGNORE_RET` macro for intentionally unchecked `write()` return values
- Fixed unused variables, operator precedence, misleading indentation, and signal mask initialization
- Fixed `wc -l` display in Makefile build output


---

## Building

```bash
cd src
make configure    # generates Makefile from your system config
make install      # builds all four binaries
```

### Requirements

- GCC 10 or later (tested on GCC 13.3.0)
- 64-bit Linux (tested on Linux 6.8.0 x86_64, glibc 2.39)
- Standard POSIX libraries (`libcrypt`)

---

## Applying Patches to Modified Talkers

If you're running a customized PG+ talker, see the companion [playground-plus-patches](https://github.com/playground-plus/playground-plus-patches) repository which provides the same fixes as modular, per-category patches designed to apply cleanly to modified codebases. Tag `v2026.1.0` in the patches repository corresponds to this release.

---

## ctl Management Script

A drop-in server management script installed to `bin/ctl` by `apply.sh`.

```bash
bin/ctl start     # Start the server stack (angel + talker)
bin/ctl stop      # Stop everything gracefully
bin/ctl restart   # Full restart (stop + start)
bin/ctl status    # Show running processes and uptime
bin/ctl reboot    # Seamless reboot via angel
bin/ctl log error # Tail a log file (angel|error|boot|connection|intercom)
```

The script auto-detects its location relative to your talker's directory structure. No configuration needed. If proxy support is enabled (`HAVE_PROXY` in `autoconfig.h` or a `proxy` binary in `bin/`), proxy management is automatically included.

---

## Heritage

This code has a long and storied lineage:

```
EW-Too (1992-93)          Simon Marsh (Burble)
    |
Summink (1994)            Neil Peter Charley (Athanasius),
    |                     Michael Simms (Grim), Nicolai Plum (Nicolai)
    |
Playground 96 (1995)      Mike Bourdaa (traP), Chris Allegretta (astyanax),
    |                     Hans Peterson (Nogard), Valerie Kelly (vallie)
    |
Playground+ 1.0 (1998)    Richard Lawrence (Silver),
    |                     J. Bradley Christian (phypor),
    |                     Geoffrey Swift (blimey)
    |
Playground+ 2026.1.0      Tony Mattke (tonhe)
```

## Credits

### Playground Plus 

**Silver** (Richard Lawrence), **phypor** (J. Bradley Christian), and **blimey** (Geoffrey Swift) - the authors of Playground Plus, a stable, bug-fixed, and improved version of Playground 96 with a wealth of additional features.

Sadly development stopped in 1998 at the height of talker population. 

### Playground 96

**traP** (Mike Bourdaa), **astyanax** (Chris Allegretta), **Nogard** (Hans Peterson), and **vallie** (Valerie Kelly) - who drastically reworked the Summink code into Playground 96.

### Summink

**Athanasius** (Neil Peter Charley), **Grim** (Michael Simms), and **Nicolai** (Nicolai Plum) - who made significant changes to the EW-Too base.

### EW-Too

**Burble** (Simon Marsh) - who wrote the original EW-Too code, with significant advancements by Chris Hughes and Jonathan Sloman.

### Contributing Code

The following people contributed code included in Playground+ by kind permission:

**Slaine**, **Oolon**, **Grim**, **subtle**, **tonhe**, **Segtor**, **Nightwolf**, **Accolade**, **Sammael**, **Bron**, **Nevyn**, **Maverick**, **Alaundo**, and **ekto**.

### Ident Server

Copyright 1995-1998 **Neil Peter Charley** (Athanasius), with Solaris/BSD compatibility portions copyright 1996 **James William Hawtin**. Distributed as part of the PG+ package with permission.

### IChan / Intercom

IChan (intercom channel) copyright **Richard Lawrence** and **Mike Bourdaa**. The intercom code copyright **Michael Simms** (Grim).


## License

Playground Plus is free software distributed under the terms established by
its original authors. The code and its components carry the following terms:

- The Playground+ codebase may be freely used and distributed for running
  talker servers, provided that credit is given to the original authors in
  the talker's `version` command and/or help credits.
- The `version` command credits must not be removed or altered beyond adding
  entries for your own installed modules.
- Individual components carry their own terms:
  - **Ident server code**: Copyright Neil Peter Charley and James William
    Hawtin. May be used as part of PG+ with credit. May not be
    redistributed separately without the express permission of the authors.
  - **Spodlist code**: Copyright Richard Lawrence. Licensed for distribution
    within PG+ only. Redistribution in other forms requires written consent.
  - **IChan/Intercom code**: Copyright Richard Lawrence, Mike Bourdaa, and
    Michael Simms. May not be redistributed without prior permission.

The 2026 modernization patches are provided freely for the PG+ community
under the same terms as the original codebase.
