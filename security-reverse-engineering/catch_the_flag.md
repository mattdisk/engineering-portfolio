# Binary Exploitation & Systems Security

A hands-on portfolio of low-level systems security work: reverse engineering native
binaries, reasoning about process memory, and developing exploits against a wide range
of memory-corruption and logic vulnerabilities across multiple CPU architectures.

## Overview

This portfolio collects work completed as part of **Georgia Tech CS 6265 (Information
Security Lab)**, a graduate course structured as a sequence of capture-the-flag style
exploitation challenges. Each lab targets a specific class of vulnerability or mitigation,
building from basic reverse engineering up to heap allocator internals and remote,
architecture-agnostic exploitation.

The course supplies the vulnerable programs; the analysis, debugging, and exploit
development represented here are my own work. The goal of this repository is to document
*what each lab explores and what skills it demonstrates* without publishing material that
would reveal challenge solutions.

## Technical Areas

The work spans the following areas, each grounded in the labs below:

- **Reverse engineering** — reading x86/x86-64 disassembly, recovering control flow and
  data structures (arrays, linked lists, binary search trees, jump tables) from compiled
  binaries, and working around decompiler limitations.
- **Shellcode development** — writing position-independent shellcode for 32-bit and 64-bit
  Linux, size-constrained shellcode, architecture-polyglot shellcode, and alphanumeric
  shellcode, all under system-call sandbox (seccomp) restrictions.
- **Stack-based memory corruption** — classic buffer overflows, control-flow hijacking,
  stack pivoting, and shellcode injection via multiple delivery channels.
- **Exploit mitigations and bypasses** — stack canaries / stack-smashing protection,
  DEP/NX, and ASLR, including the techniques used to defeat or sidestep each.
- **Format-string vulnerabilities** — information disclosure and arbitrary memory writes.
- **Return-oriented programming (ROP)** — ret-to-libc and gadget-based control flow on
  x86, x86-64, ARM, and RISC-V, including dynamic-linker abuse.
- **Heap exploitation** — allocator metadata corruption, use-after-free, and several
  well-known heap exploitation primitives.
- **Logic and concurrency vulnerabilities** — integer overflow, race conditions,
  time-of-check-to-time-of-use (TOCTOU), type-confusion, and application-level
  deserialization bugs.
- **Remote exploitation** — interacting with networked services over sockets, including a
  timing side channel and automated, per-connection exploit generation.

## Tools

Tools actually used across the writeups:

- **GDB** with **pwndbg** — dynamic analysis, memory inspection, debugging.
- **Ghidra** and **objdump** — static disassembly and decompilation.
- **pwntools** — exploit scripting, ROP chain construction, process/remote I/O.
- **Python** — automation, including `ctypes` for mirroring C library behavior.
- **ae64** — alphanumeric shellcode encoding.
- **GNU assembler** (`.intel_syntax`) — hand-written shellcode.
- Standard networking and shell tooling (`nc`, symlinks, file-descriptor manipulation).

## Labs

| Lab | Focus | Skills / Techniques |
|-----|-------|---------------------|
| Lab 01 | Reverse engineering a multi-stage "binary bomb" | x86 disassembly, GDB, recovering data structures and control flow |
| Lab 02 | Shellcode writing | 32/64-bit shellcode, size-constrained & polyglot & alphanumeric shellcode, seccomp |
| Lab 03 | Stack buffer overflows | Control-flow hijack, shellcode injection, stack pivoting, pwntools |
| Lab 04 | Stack canary protections | Canary leaking/bypass, off-by-one, TLS internals, GOT/PLT overwrite |
| Lab 05 | DEP / NX and ASLR | Format strings, ret-to-libc, libc base computation, GOT overwrite |
| Lab 06 | Return-oriented programming | ROP on x86 / x86-64 / RISC-V, gadget discovery, RELRO, dynamic linker |
| Lab 07 | Remote exploitation | Sockets, timing side channel, multi-arch ROP, TOCTOU, return-to-dl-resolve |
| Lab 08 | Logic & concurrency bugs | Integer overflow, race/TOCTOU, type confusion, deserialization, sandbox escape, automated exploit generation |
| Lab 09 | Heap exploitation | Allocator metadata corruption, use-after-free, heap grooming primitives |

The table stays at a high level; detailed per-lab material is kept private (see below).

## What This Work Demonstrates

- Debugging and understanding unfamiliar native programs with no source available.
- Reasoning precisely about process memory layout: the stack, the heap, shared libraries,
  the GOT/PLT, and thread-local storage.
- Reading and writing assembly across four instruction-set architectures (x86, x86-64,
  ARM, RISC-V).
- Understanding calling conventions and how compilers lay out code and data.
- Analyzing control flow and reconstructing high-level logic from machine code.
- Reasoning about memory-corruption vulnerabilities and the mitigations designed to stop
  them, including the practical limits of those mitigations.
- Automating interaction with local and remote binaries, including fragile,
  timing-sensitive and probabilistic exploits.
- Developing structured, repeatable approaches to unfamiliar low-level problems.

## Repository Access / Detailed Material

Detailed technical writeups and implementation artifacts are kept in a private version of
this repository because they contain challenge-specific solution material. They can be
shared for interview review on request.

## Attribution

The challenges originate from **Georgia Tech CS 6265 (Information Security Lab)**. The
course provides the vulnerable programs and the surrounding infrastructure; the exploitation
analysis and development documented here are my own.

---

*This page reproduces the project's README. The source files and supporting documents it mentions are kept in a private source archive and are available on request.*
