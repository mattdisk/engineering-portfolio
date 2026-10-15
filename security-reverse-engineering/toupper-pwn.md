# toupper — A 32-bit Linux Binary Exploitation CTF Challenge

`toupper` is a binary exploitation ("pwn") challenge I **designed and
implemented** for Fall 2025 TKCTF. It is a 32-bit Linux challenge built around a small
networked text-transformation service: you connect over the network, send input,
and the service transforms and prints it.

This repository contains the full challenge — source, build configuration,
containerized deployment, the material originally handed to competitors, and a
reference solution. It is a challenge-*design* project, not a write-up of
someone else's challenge.

> **Publication permission:** This challenge is published here publicly **with
> explicit permission from the TKCTF organizers.** It was originally created for
> Fall 2025 TKCTF, the Capture-the-Flag event associated with Georgia Tech CS 6265
> (Information Security Lab). Publishing it here does not imply any endorsement
> by Georgia Tech or by the CS 6265 course staff.

---

## Competition context

- **Event:** TKCTF, the CTF associated with Georgia Tech CS 6265, a hands-on
  information-security / binary-exploitation course. TKCTF challenges are
  authored by course staff and participating teams.
- **My role:** I designed and built this challenge. I did not create TKCTF or
  CS 6265.
- **Result:** Solved by **2 of 42 participating teams** during the event.
- **Rules note:** AI-assisted solving was permitted during the competition.

---

## Challenge description

This is what competitors saw when they started the challenge.

The service is a simple text transformer. It accepts input, applies a
case-oriented transformation to it, and prints the result. The stated premise
handed to participants was:

> All digits and lowercase letters will be converted to uppercase.
> Output `FLAG` (without quotes) to get the **FAKE** flag.
> Output `flag` (without quotes) to get the **REAL** flag.

The challenge runs as a network service; competitors also received the compiled
target binary to inspect locally.

---

## Repository structure

```text
.
├── docker/          # Container deployment
│   ├── Dockerfile   # Ubuntu 24.04 + i386 runtime, socat service as `ctf`
│   ├── target       # Compiled challenge binary (served)
│   └── flag         # The real flag (organizer-side only)
├── release/         # What participants originally received
│   ├── README       # Challenge description shown to competitors
│   └── target       # The challenge binary handed out
├── source/          # Challenge development + reference solution  ← SPOILERS
│   ├── src/
│   │   ├── target.c # Challenge source                           ← SPOILER
│   │   └── Makefile # Build configuration
│   ├── exploit.py   # Reference exploit (pwntools)                ← SPOILER
│   ├── writeup.txt  # Intended-solution notes                     ← SPOILER
│   └── test.sh      # Build → run → exploit, end to end
├── Makefile         # Top-level dist/build/run/test targets
├── NAME             # Challenge identifier metadata
├── PORT             # Service port metadata
└── README.md        # This file
```

- **`release/`** holds exactly what competitors were given: the challenge
  description and the target binary. Start here if you want to attempt the
  challenge.
- **`source/`** holds challenge-development and reference-solution material. It
  contains the source code, the intended write-up, and a working exploit, so it
  **reveals the solution**.

The three copies of `target` under `docker/`, `release/`, and `source/src/` are
byte-for-byte identical; they exist because the build distributes one compiled
binary to each location.

---

## ⚠️ Spoiler warning

**Everything above this point is intended to be safe for someone who wants to
attempt the original challenge.** If you want to solve it yourself, start with
`release/` and **stop reading here.**

**Everything below this point** discusses the challenge implementation, the
vulnerability, the intended exploitation path, and the reference solution.

---

## Challenge design

The twist is in the name. The service's advertised behavior is also the obstacle:
because it upper-cases lowercase letters, the lowercase string the rules ask you
to produce (`flag`) can never survive the input path intact — typing it simply
turns it into `FLAG`, which only yields the decoy. Reaching the real flag
therefore cannot be done by "just sending the right text"; it requires getting
control of the program and reconstructing the forbidden string another way.

The elegant part — and the reason for the name — is that the very
transformation that blocks the obvious solution becomes the tool used to defeat
it. The case-conversion helpers, which exist only to implement the advertised
feature, double as arithmetic primitives in the intended exploit.

## Technical design

The target is a single C program compiled for 32-bit x86 and deployed as a
network service. The interesting engineering is in how a handful of ordinary
features combine into a constrained, non-obvious exploitation path.

### Build and hardening

From the build configuration (`source/src/Makefile`):

| Setting | Flag | Effect |
| --- | --- | --- |
| Architecture | `-m32` | 32-bit x86 |
| Optimization | `-O0` | predictable, un-optimized codegen |
| Block ordering | `-fno-reorder-blocks` | stable layout for the challenge |
| Non-executable stack | `-z noexecstack` | NX on; no shellcode on the stack |
| Stack canaries | `-fstack-protector-all` | a canary in **every** function |
| Position dependence | `-no-pie` | fixed code/data addresses |

The combination matters. NX removes classic stack-shellcode. `-no-pie` gives
fixed addresses (so no leak is needed to locate code and data), but
`-fstack-protector-all` places a stack canary in front of the saved return
address in every function — including the vulnerable one — so a naïve overflow
trips `__stack_chk_fail` before control is ever transferred.

### The vulnerable program

The core routine reads a caller-supplied `size`, then uses it directly to bound
the input read and the subsequent copy. There is no upper bound on that size
relative to the destination buffers, so a large size produces a classic stack
buffer overflow. The copy itself is the custom transformation described above.

Two small helper functions perform the transformation one byte at a time: one
shifts a lowercase letter into the uppercase range, and one remaps a digit into
the uppercase-letter range. These exist to implement the service's advertised
behavior — but because each performs a fixed, known byte-level shift, they also
turn out to be useful arithmetic primitives in the intended exploit. The exact
constants are in `source/src/target.c` and `source/writeup.txt`.

### Deployment

The service is containerized (`docker/Dockerfile`):

- Base image `ubuntu:24.04` with the i386 multilib runtime (`libc6:i386`,
  `libstdc++6:i386`) added so the 32-bit binary runs.
- Exposed over TCP with `socat`, forking one process per connection, under an
  unprivileged `ctf` user, with a per-connection timeout.
- The flag is mounted read-only; the real flag is only reachable through the
  program's own privileged-looking code path, not by reading arbitrary files.

A helper script (`source/test.sh`) builds the image, launches the container,
and runs the reference exploit against the live service end to end.

## Why the challenge is interesting

Beyond the central trap described above, the difficulty comes from an
interaction between ordinary mechanisms that closes off the usual shortcuts:

- **The canary has to be defeated, not avoided.** With
  `-fstack-protector-all`, the overflow crosses a canary. The intended approach
  exploits the symmetric stack layout of the two functions involved: the saved
  canary from the caller sits at a known relative offset inside the
  attacker-controlled region, so the overflow can reproduce the correct canary
  value at the vulnerable function's canary slot and survive the check. This is
  grounded in the layout, not in leaking the secret.
- **Standard ROP cleanup is unavailable.** The binary lacks a convenient
  multi-pop gadget, which rules out the textbook pattern of calling library
  functions with controlled arguments and cleaning up the stack between calls.
  That constraint is what pushes the solver toward the unusual primitive below.
- **The transformation routines become the exploit.** Because the ROP chain's
  length is bounded (overrun too far and you clobber the canary you just fixed),
  the intended solution repurposes the program's own transformation functions as
  arithmetic primitives within a constrained ROP chain, using them to
  *synthesize* the forbidden string in writable memory rather than sending it as
  input. Keeping that construction short enough to fit the length budget is part
  of the puzzle. The concrete construction is documented in
  `source/writeup.txt`.
- **Reliability is part of the design.** The construction can fail for certain
  runtime values of the canary bytes, so the exploit is written to retry a
  small number of times. Making a bruteforce-tolerant exploit *reliable within
  a few attempts* — rather than relying on pure guessing — was a deliberate
  design target, consistent with the event's "well-designed bruteforcing is
  fine, pure guessing is not" guidance.

## Intended solution

Putting the pieces above together, the reference exploit overflows via the
unbounded size, restores the canary from the symmetric stack layout, synthesizes
the required string in a fixed buffer using the transformation functions as ROP
primitives, and triggers the privileged path — retrying a few times for
reliability.

Exact offsets, addresses, per-character arithmetic, and the full chain are
intentionally not reproduced here. The complete intended solution lives in
`source/writeup.txt` (notes) and `source/exploit.py` (working pwntools exploit).

## What this project demonstrates

This repository is primarily an exercise in *designing* a vulnerable system
with a controlled, intended exploitation path — which is a different skill from
solving someone else's challenge. Concretely, it involves:

- **Challenge and adversarial design** — shaping a bug so that the obvious
  approaches are closed off and a specific, teachable technique is the way
  through.
- **C and low-level memory behavior** — stack layout, buffer overflows, and how
  saved canaries and return addresses sit relative to local buffers.
- **Exploit mitigations** — working with (and deliberately around) stack
  canaries, NX, and position-dependent code, and reasoning about what each
  mitigation does and does not prevent.
- **Return-oriented programming under constraints** — building a chain when the
  convenient gadgets aren't present, and repurposing in-binary functions as
  arithmetic primitives.
- **Exploit development with pwntools** — ELF/ROP handling and a
  retry-tolerant, reliable exploit.
- **Reproducible deployment** — a Dockerized 32-bit service over `socat`, with
  an end-to-end build-and-verify script.

## Reproducing / running the challenge

The included tooling documents the workflow (top-level `Makefile`):

```text
make dist     # build the target and copy it to docker/ and release/
make build    # build the Docker image
make run      # run the service container
make exploit  # run the reference exploit against the service
make test     # build image, launch it, run the exploit end to end
```

Building and running require a Linux environment with a 32-bit toolchain and
Docker. These commands come directly from the repository's existing build
scripts; I have not re-executed them as part of writing this README, so treat
them as the documented workflow rather than a freshly re-verified run.

## Attribution

- **Challenge author / designer:** me.
- **Created for:** TKCTF, the CTF associated with Georgia Tech CS 6265.
- **Public release:** made with explicit permission from the TKCTF organizers.

I did not create TKCTF or CS 6265, and this public release does not imply
endorsement by Georgia Tech or the CS 6265 course staff. The competition
statistics above are provided as factual context, not as an official
endorsement.
