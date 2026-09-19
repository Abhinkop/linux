# AGENTS.md — microkernel-linux (kernel fork)

Read this before doing anything in this repo. It is loaded into context at
the start of every Claude Code session here (Claude Code reads it via the
`CLAUDE.md` file next to this one, which just imports this file).

## What this repo is

A fork of mainline Linux (`github.com/Abhinkop/linux`), branch
`microkernel-linux`, currently tracking ~7.3-rc2. Goal: turn Linux into (the
start of) a microkernel by moving device drivers out of kernel space into
ordinary userspace processes, one driver at a time. **Only the QEMU aarch64
`virt` machine is supported.** Nothing else is a target.

## Current state — keep this section honest as work lands

- `arch/arm64/configs/virt_uart_defconfig` — tinyconfig-based. Boots to a
  UART console and almost nothing else: MULTIUSER, FUTEX, EPOLL, SIGNALFD,
  EVENTFD, SHMEM, BLOCK, PROC_FS, SYSFS, FILE_LOCKING are all off.
  `SERIAL_AMBA_PL011`(+`_CONSOLE`) is currently **on** — the UART is still
  an ordinary in-kernel driver as of today.
- `arch/arm64/boot/dts/qemu/qemu-virt-uart-min.dts` — hand-trimmed
  devicetree: PSCI, GIC, arch timer, memory, PL011 only. Matches the fixed
  QEMU invocation below exactly. No virtio-mmio, PCIe, CFI flash, PL031,
  PL061, fw-cfg, platform-bus.
- Userspace already works end to end with this config: PID 1 exec, the
  basic syscall ABI, and `/dev/console` I/O for a shell are all proven.
  Don't redo this — it's done.
- **Current milestone: run the UART driver in userspace.** End state: the
  kernel contributes exactly two generic, reusable mechanisms — CUSE
  (expose a device file to userspace) and UIO (expose hardware to
  userspace) — and the UART driver is 100% userspace code built on both.
  No device-specific driver code in the kernel.

## Fixed QEMU boot command — the only supported target, do not vary it

```
qemu-system-aarch64 -M virt -cpu cortex-a57 -smp 1 -m 256 -nographic \
    -kernel arch/arm64/boot/Image \
    -dtb arch/arm64/boot/dts/qemu/qemu-virt-uart-min.dtb \
    -initrd <initramfs>.cpio.gz
```

Build:

```
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make virt_uart_defconfig
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make -j$(nproc) Image dtbs
```

**A kernel-side change that "should" work but hasn't actually been booted
is not done.** Always build and boot after a change, every time.

## Scope discipline — read this before touching any Kconfig symbol

This is the most common way an agent quietly makes this project worse:
growing the kernel config surface "while already in there."

- v1 scope is UART (required, the actual milestone) + GPIO (recommended
  rehearsal device — same CUSE+UIO mechanism, proven on a non-console
  device first so console and protocol bugs are never debugged at the same
  time). Nothing else.
- **Explicitly out of scope** until a human says otherwise: `BLOCK` /
  `IO_URING` (block storage / `ublk`), FUSE proper as opposed to CUSE, and
  PCI/VFIO-style userspace drivers. Do not enable any of these, or their
  prerequisites, as a side effect of another task.
- The only kernel-side additions for the whole MVP are `CONFIG_FUSE_FS`,
  `CONFIG_CUSE`, `CONFIG_UIO`, `CONFIG_UIO_PDRV_GENIRQ`. If a task seems to
  need anything else in Kconfig, **stop and ask a human** rather than
  enabling it yourself.
- Turning off `CONFIG_SERIAL_AMBA_PL011` kills the kernel's own console
  (`SERIAL_AMBA_PL011_CONSOLE` selects `SERIAL_EARLYCON`) — only do this
  once the CUSE+UIO path is already proven end-to-end against GPIO, and
  keep a throwaway debug defconfig variant with earlycon intact so boot
  failures are still bisectable.

## Userspace driver-daemon conventions

- Daemons are hand-linked, freestanding binaries using
  `tools/include/nolibc` — not glibc, not musl, not autotools/cmake.
  Compile roughly as:
  `${CC} -nostdlib -static -I<path-to>/tools/include/nolibc ...`
- They implement the `/dev/cuse` wire protocol directly — the
  `CUSE_INIT` handshake, then `FUSE_OPEN`/`FUSE_READ`/`FUSE_WRITE`/
  `FUSE_RELEASE` framed with `struct fuse_in_header`/`fuse_out_header`
  from `<linux/fuse.h>` — no libfuse.
- One daemon bridges two file descriptors: `/dev/cuse` (client-facing,
  registers the device node) and `/dev/uioN` (hardware-facing, opened
  once the CUSE side is already working).
- Stage the work the way the plan stages it: prove the CUSE protocol
  against a dummy backend (canned/logged responses, no real hardware)
  *before* wiring in real UIO register access. Don't merge protocol work
  and hardware work into one untested step — if it breaks, you won't know
  which half broke.
- `FUSE_READ` replies can be deferred (this is standard FUSE behavior) —
  use that for a `read()` that should block until real hardware data
  arrives, rather than inventing separate blocking machinery.

## The build/verify loop expected of every change

1. Make the smallest change that completes one numbered step of the
   current phase. (The full phase breakdown lives in `../docs/plan.md` —
   a local copy of this project's design doc, kept in the coordinator
   repo so it's readable from any session without claude.ai access. If
   you don't have the current step in front of you, read it there rather
   than guessing scope.)
2. Build, then boot with the exact QEMU command above.
3. Confirm the expected behavior from serial output, or via
   `scripts/smoke-test.sh` once Phase 3 adds one — until then, say
   explicitly what you checked for in the boot log.
4. Show the diff and the relevant boot-log excerpt before committing.
   Commit only that one verified step — not a whole phase at once.

The `boot-verifier` subagent in `.claude/agents/` automates steps 2–3; use
it after implementing a step, before calling that step done.

## What never gets touched

Mainline Linux subsystems unrelated to this project (schedulers, mm
internals, unrelated drivers, unrelated arches) are off-limits — this is a
scoped fork, not a general refactor. If a step seems to require touching
anything outside `arch/arm64/configs/`, `arch/arm64/boot/dts/qemu/`, or the
userspace-daemon source tree, stop and ask before doing it.

## Git

Branch: `microkernel-linux`. One commit per completed, boot-verified step —
not one giant commit per phase. Reference the plan step in the message,
e.g. `Phase 1 step 2: dummy CUSE daemon answers OPEN/READ/WRITE/RELEASE`.
Show the diff for review before committing; don't commit unreviewed.
