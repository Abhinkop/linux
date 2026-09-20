# AGENTS.md — microkernel-linux (kernel fork)

Read this before doing anything in this repo. It is loaded into context at
the start of every Claude Code session here (via the `CLAUDE.md` next to it,
which just imports this file).

## What this repo is

A fork of mainline Linux (`github.com/Abhinkop/linux`), branch
`microkernel-linux`, currently tracking ~7.3-rc3. Goal: turn Linux into (the
start of) a microkernel by moving device drivers out of kernel space into
ordinary userspace processes.

**Only the QEMU aarch64 `virt` machine is supported.** Nothing else is a
target.

The metric is **how much source still runs at EL1**, and the reason that
metric matters is **static analysis** — a kernel small enough to analyse
exhaustively is a different object from one that isn't. Every symbol you
enable is source running at EL1.

## This repo does not build on its own — it is built by the coordinator workspace

This fork is the kernel source only. It is built by Yocto from a separate
coordinator workspace, and **this checkout is expected to sit at `linux/`
inside it**:

```
<workspace>/linux/                 <- you are here
<workspace>/plan.md                <- THE plan; authoritative
<workspace>/RESUME.md              <- current state / handoff
<workspace>/scripts/               <- every entry point
<workspace>/yocto/                 <- layers + build tree
```

**There is no host toolchain.** This machine has no `gcc`, no
`qemu-system-aarch64`, and no Yocto host dependencies — by design
(requirement: "the host provides Docker and nothing else"). Every build and
boot runs inside the project's container.

So **do not run `make`, `CROSS_COMPILE=… make`, or `qemu-system-aarch64`
directly.** They are not installed and their absence is not a bug to fix.
Earlier versions of this file told you to; that was wrong.

The kernel is built by Yocto in **dev mode**, where `EXTERNALSRC` points at
this tree. That means: **edit a file here, re-run the build, done.** No
commit, no push, no `SRCREV` bump. Build output goes outside this tree, so
`git status` here stays meaningful.

## The build/boot loop — the only supported one

From the workspace root (`..` from here):

```
scripts/dev-container.sh bash scripts/build.sh core-image-minimal
scripts/dev-container.sh bash scripts/boot.sh 180
```

`scripts/build.sh linux-microkernel` builds just the kernel when you don't
need the rootfs rebuilt.

**`scripts/boot.sh` is the single source of truth for the QEMU invocation.**
Do not write out a `qemu-system-aarch64` command line anywhere else — this
file used to carry its own copy, it drifted out of agreement with the real
one, and that is what made this document dangerous. If the invocation needs
to change, change `boot.sh`.

`boot.sh` exiting on its timeout is **not** a failure: init reaches a login
prompt and sits there, so QEMU gets killed on the timer. Read the serial log
for what you actually wanted to check.

## Current state — keep this section honest as work lands

Stage 1 (dev environment) is **complete and verified** as of 2026-09-20: the
kernel builds from this tree via `EXTERNALSRC` and `core-image-minimal` boots
on QEMU aarch64 `virt` to a login prompt. The version string
`7.3.0-rc3-gca44a7dbff38` confirms the build came from this checkout.

That baseline is deliberately the **stock arm64 `defconfig`** with no custom
devicetree — a known-good reference, so that a later failure means the config
is wrong rather than the setup.

**Current milestone: stage 2 — minimal configuration.** Strip the config to
the least that still boots on `virt`: keep the PL011 UART, GIC, architected
timer, PSCI; everything else comes out. Stage 2 is not done when it boots —
it is done when minimality is **demonstrated**, i.e. for each symbol still
enabled, turning it off breaks the build or the boot.

**Two artifacts in this tree predate the current plan and are UNVERIFIED
under the current setup — do not assume they work:**

- `arch/arm64/configs/virt_uart_defconfig` (commit `ae3b635f96b4`) —
  tinyconfig-based, claimed to boot to a UART console with MULTIUSER, FUTEX,
  EPOLL, SIGNALFD, EVENTFD, SHMEM, BLOCK, PROC_FS, SYSFS and FILE_LOCKING
  off. `SERIAL_AMBA_PL011`(+`_CONSOLE`) on.
- `arch/arm64/boot/dts/qemu/qemu-virt-uart-min.dts` (commit `5a644e5ea249`)
  — hand-trimmed devicetree: PSCI, GIC, arch timer, memory, PL011 only.

They were built and booted with a host cross-toolchain and a hand-passed
`-dtb`, which is **not** how this project builds now. They are a strong
head start on stage 2, quite possibly most of it — but they have never been
through the Yocto path, and `boot.sh` passes no `-dtb` at all (QEMU
generates a devicetree matching the machine it built). Re-verify before
claiming stage 2 progress, and expect the DTS to need a decision about
whether a hand-written DT is wanted at all yet.

Earlier notes in this file also claimed userspace "already works end to end
— PID 1 exec, syscall ABI, `/dev/console` I/O — don't redo this." Nothing in
this tree demonstrates that. Treat it as an unverified inherited claim.

## The plan, and what supersedes what

**`../plan.md` is the only plan.** Six stages:

1. Dev environment — **done**
2. Minimal config, minimality demonstrated — **current**
3. GPIO (PL061) in userspace
4. **Kernel-API compatibility library** — the core of the project
5. Every driver but the UART in userspace
6. Subsystems out of the kernel, console first

**Superseded design — do not resurrect it.** Earlier work here aimed at
hand-written, per-driver freestanding daemons using `tools/include/nolibc`
that spoke the `/dev/cuse` wire protocol directly (`CUSE_INIT` handshake,
then `FUSE_OPEN`/`READ`/`WRITE`/`RELEASE` framed with `struct
fuse_in_header`), with a fixed "run the UART driver in userspace via
CUSE+UIO" milestone.

Stage 4 replaces that: a library against which **upstream in-kernel driver
source compiles unmodified** and runs in userspace, supplying the kernel-side
API the driver calls, backed by IPC to the kernel for what only the kernel
can do — mapping MMIO, delivering interrupts, allocating memory. Porting a
driver should be a recompile, not a rewrite. CUSE may still survive as a
redirect mechanism, but hand-written per-driver protocol daemons are not the
design.

Also: earlier text pointed at `../docs/plan.md` for a phase breakdown, and
framed work as "Phase 1/2/3". **That file does not exist** and those phase
numbers are not the plan. Use `../plan.md` and its stage numbers.

## Scope discipline — read this before touching any Kconfig symbol

This is the most common way an agent quietly makes this project worse:
growing the kernel config surface "while already in there."

- Minimality applies at **every** stage, not just stage 2. Each stage adds
  capability; each addition is held to the smallest kernel footprint that
  delivers it.
- Do not enable a symbol as a side effect of another task. If a task seems
  to need a Kconfig addition that the current stage doesn't explicitly call
  for, **stop and ask a human** rather than enabling it yourself.
- The specific set of kernel-side symbols stage 4 needs is **not yet
  decided** — it follows from what the compatibility library's IPC requires.
  Do not invent an allow-list, and do not carry over the old design's list
  (`FUSE_FS`/`CUSE`/`UIO`/`UIO_PDRV_GENIRQ`) as if it were settled.
- Explicitly out of scope until a human says otherwise: block storage
  (`BLOCK`/`IO_URING`/`ublk`), FUSE proper as opposed to CUSE, and
  PCI/VFIO-style userspace drivers.
- Turning off `CONFIG_SERIAL_AMBA_PL011` kills the kernel's own console
  (`SERIAL_AMBA_PL011_CONSOLE` selects `SERIAL_EARLYCON`). The UART is
  deliberately the **last** thing to leave the kernel (plan.md stage 5) —
  the kernel needs something it can log through whether or not userspace is
  alive. Keep a debug config variant with earlycon intact so boot failures
  stay bisectable.

## What never gets touched

Mainline Linux subsystems unrelated to this project (schedulers, mm
internals, unrelated drivers, unrelated arches) are off-limits — this is a
scoped fork, not a general refactor. If a step seems to require touching
anything outside `arch/arm64/configs/`, `arch/arm64/boot/dts/qemu/`, or the
userspace/compat-library source, stop and ask first.

## Verification discipline

**A kernel-side change that "should" work but hasn't actually been booted is
not done.** Always build and boot after a change, every time. The
`boot-verifier` subagent in `.claude/agents/` automates it.

Say explicitly what you checked for in the serial log — don't just assert it
booted, and don't treat `boot.sh`'s timeout as the result either way.

## Git

Branch: `microkernel-linux`. One commit per completed, boot-verified step —
not one giant commit per stage. Reference the plan stage in the message, e.g.
`Stage 2 step 3: drop BLOCK, PROC_FS from virt defconfig`.

Show the diff for review before committing; don't commit unreviewed.
