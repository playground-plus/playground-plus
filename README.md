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
 ║            v2026.2.0 - by Tony Mattke aka tonhE              ║
 ║                                                              ║
 ╚══════════════════════════════════════════════════════════════╝
```
<p align="center">
  <img src="https://img.shields.io/badge/version-2026.1.1-blue?style=flat-square" alt="Version 2026.1.1" />
  <img src="https://img.shields.io/badge/GCC-10%2B-green?style=flat-square" alt="GCC 10+" />
  <img src="https://img.shields.io/badge/platform-Linux%20x86__64-lightgrey?style=flat-square" alt="Linux x86_64" />
  <img src="https://img.shields.io/badge/warnings-0-brightgreen?style=flat-square" alt="Zero Warnings" />
</p>

# Playground Plus v2026.2.0

> [!NOTE]
> This repository is not an official repository maintained by the original Playground+ developers. It is a community effort to modernize and preserve the PG+ codebase. The companion [playground-plus-patches](https://github.com/playground-plus/playground-plus-patches) repository is also unofficial.

> [!CAUTION]
> **Security bulletin — HCAdmin login flow**
>
> Two vulnerabilities affect talkers with HCAdmin names defined in `config.msg` that lack active pfiles:
>
> - **Unlimited password attempts** against the HC password challenge — no rate limiting or lockout.
> - **HCAdmin privileges granted before password verification** — connecting clients are momentarily treated as HCAdmin before `check_hcadmin_pw` runs.
>
> Both were fixed in v2026.1.1. Operators running prior or modified PG+ codebases should review the writeup and apply the patches: **[playground-plus#2](https://github.com/playground-plus/playground-plus/issues/2)**.

**The classic talker server, modernized for 2026.**

Playground Plus is a multi-user chat server ("talker") with a rich history stretching back to the early 1990s. Originally built on the EW-Too codebase, it evolved through Summink and Playground 96 into Playground+, which became one of the most widely deployed talker platforms of its era.

This release brings PG+ into the modern age with encrypted connections, server-side character-mode editing, and robust color handling. It builds cleanly on current 64-bit Linux systems with zero errors and zero warnings, fixes critical security vulnerabilities, and replaces deprecated system APIs - all while preserving the talker experience that operators and users know.

---

## What's New in v2026.2.0

**Released 26th March 2026**

### Frontend Proxy

A new frontend proxy architecture (`bin/proxy`) sits in front of the talker, handling encryption and authentication externally. The proxy listens on the public port and relays plain TCP to the talker on localhost.

- **Telnet** - Standard telnet connections, same as always
- **STARTTLS** - Plain telnet clients can upgrade to TLS mid-session via the STARTTLS telnet option (IAC DO 46)
- **TLS** - Direct TLS connections, auto-detected via TLS ClientHello on the same port
- **SSH** - Public key, password, and keyboard-interactive authentication handled by the proxy. SSH users see the `connect.msg` banner before login. If the SSH client provides a username, only a password prompt is shown; otherwise a telnet-like name and password prompt is presented. Characters must be created via telnet/TLS first.

The proxy enables **zero-drop seamless reboots** for all connection types. During a reboot, the proxy holds all encryption state while the talker restarts - no sessions are dropped, including TLS and SSH. Command history is preserved across seamless reboots. Send `SIGUSR1` to the proxy to hot-reload TLS certificates without dropping existing sessions.

All proxy features are individually configurable via `make config`. Self-signed TLS certificates are auto-generated during configuration if none exist. Partial writes are handled robustly throughout the proxy, and ident/hostname fields are sanitized before inclusion in proxy headers.

### Character-Mode Input

Server-side character-mode input replaces the old line-mode model. Enabled by default via `HAVE_CHARMODE` in `make config`.

- **Readline-style editing** - Arrow keys, Home/End, Ctrl+A/E (home/end), Ctrl+B/F (left/right), Ctrl+U (kill line), Ctrl+K (kill to end), Ctrl+W (delete word), Delete key
- **Command history** - Up/Down arrows browse a 50-entry per-player command history ring buffer (runtime only, not persisted to disk, but preserved across seamless reboots)
- **Clean output redraw** - When other players' messages arrive while you're typing, the server erases your input line, displays the output, then redraws your prompt and input with cursor position preserved. No more clobbered input.
- **Escape sequence parser** - Full VT100/xterm CSI and SS3 support covering 99%+ of terminals, including Ctrl+Left/Right for word-at-a-time navigation
- **Graceful fallback** - Charmode is negotiated per-connection via telnet WILL ECHO + WILL SGA. Clients that don't support it fall back to line mode automatically. SSH connections enable charmode immediately.
- **Password-safe** - Character echo and history are suppressed during password entry

### Color Termination (Boots and Braces)

A layered defense-in-depth strategy ensures color codes from one piece of output can never bleed into the next.

- **Layer 1 (Output reset)** - `process_output()` emits a terminal reset at both the start and end of every output. Every message begins and ends in a known-good color state.
- **Layer 2 (Field termination)** - All user-settable display fields (prompt, title, pretitle, description, plan, enter/exit/logon/logoff messages, idle message, favorites, etc.) are automatically terminated with `^N` at storage time so they can never bleed color regardless of where they're interpolated.
- **Layer 3 (Charmode prompt)** - The charmode prompt pre-processes talker `^X` color codes into real ANSI sequences and always appends a terminal reset, since the charmode redraw path bypasses `process_output()`.

### Build and Compatibility Fixes

- Removed `HAVE_SIGEMPTYSET` ifdefs -- always use `sigemptyset()` (standard POSIX everywhere, old fallback `sa.sa_mask = 0` fails on modern glibc where `__sigset_t` is a struct)
- Removed deprecated `sa_restorer` field assignment in ident server
- Replaced deprecated `egrep` with `grep -E` in all configure scripts
- Removed stray backslash escapes in configure scripts that triggered warnings
- Fixed `sprintf` overflow in `examine.c` staff list (buffer bumped, switched to `snprintf`)
- Fixed compiler warnings when robot code is disabled in `make config` (`#ifdef ROBOTS` guards)
- Fixed inline telnet IAC parsing to work correctly with buffered reads
- Fixed intercom `sigchld` handler to preserve `errno` across `waitpid` reaping
- Removed dead signal re-registration blocks from `glue.c` (pre-sigaction leftovers, never compiled on Linux)
- Removed dead `sigsys` handler (Linux handles SIGSYS via seccomp, not traditional signals)
- Fixed proxy `strncpy` truncation warning on username fields (switched to `snprintf`)
- Fixed intercom `sync_talkers` permanently replacing hostnames with resolved IPs in `intercom.dbase`
- Zero errors, zero warnings across all five binaries with GCC `-Wall`

---

## What's New in v2026.1.1

**Released 30th April 2026**

### Security Fixes

- **HCAdmin login flow hardened** — see the [security bulletin](#playground-plus-v202611) above and [playground-plus#2](https://github.com/playground-plus/playground-plus/issues/2) for full details. Three changes:
  - HC password challenge now fires only when the named hcadmin has no pfile (saved hcadmins authenticate via their pfile password as expected).
  - HCAdmin privileges (`HCADMIN_INIT`, `ressied_by`) are now granted only after a successful HC password match, gated on a new `HC_AUTH` session flag — never on name match alone.
  - Per-IP failure tracking with a 10-minute lockout after 5 failed HC password attempts (`add_ip_fail` / `is_ip_locked` in `socket.c`); enforced at connection accept.

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
make config       # configure features (proxy, charmode, etc.)
make install      # builds all binaries
```

### Requirements

- GCC 10 or later (tested on GCC 13.3.0)
- 64-bit Linux (tested on Linux 6.8.0 x86_64, glibc 2.39)
- Standard POSIX libraries (`libcrypt`)
- OpenSSL development headers (for TLS proxy support)
- libssh development headers (for SSH proxy support)

---

## Applying Patches to Modified Talkers

If you're running a customized PG+ talker, see the companion [playground-plus-patches](https://github.com/playground-plus/playground-plus-patches) repository which provides the same fixes as modular, per-category patches designed to apply cleanly to modified codebases. Tag `v2026.1.1` in the patches repository corresponds to this release.

---

## ctl Management Script

A drop-in server management script installed to `bin/ctl` by `apply.sh`.

```bash
bin/ctl start     # Start the server stack (angel + proxy + talker)
bin/ctl stop      # Stop everything gracefully
bin/ctl restart   # Full restart (stop + start, disconnects everyone)
bin/ctl status    # Show running processes and uptime
bin/ctl reboot    # Seamless talker reboot (proxy stays, no disconnects)
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
