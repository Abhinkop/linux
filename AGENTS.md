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

So **do not run `make`, `CROSS_COMPILE=… make`, or `qemu-system-aarch64` on
the host.** They are not installed and their absence is not a bug to fix.
Earlier versions of this file told you to; that was wrong.

Inside the container is different — the cross toolchain and QEMU are right
there, and a direct `make` is the correct tool when you need many kernel
builds in a row (a config search does hundreds; a bitbake round trip each
time would be impractical). See `stage2-minimal-config/harness/` in the
coordinator repo for that pattern. For anything you intend to *ship*, go
through bitbake, so what you verified is what gets deployed.

The kernel is built by Yocto in **dev mode**, where `EXTERNALSRC` points at
this tree. That means: **edit a file here, re-run the build, done.** No
commit, no push, no `SRCREV` bump. Build output goes outside this tree, so
`git status` here stays meaningful.

## The build/boot loop — the only supported one

From the workspace root (`..` from here):

```
scripts/dev-container.sh bash scripts/build.sh linux-microkernel microkernel-initramfs
scripts/dev-container.sh bash scripts/boot.sh 60
```

`scripts/build.sh linux-microkernel` builds just the kernel when the
initramfs has not changed.

**`core-image-minimal` will not boot this kernel.** `virt_min_defconfig` has
no `MULTIUSER`, `PROC_FS`, `SYSFS` or `FILE_LOCKING`, so sysvinit and udev
cannot run on it. `microkernel-initramfs` is the matching userspace: one
freestanding binary and a `/dev/console` node.

**`scripts/boot.sh` is the single source of truth for the QEMU invocation.**
Do not write out a `qemu-system-aarch64` command line anywhere else — this
file used to carry its own copy, it drifted out of agreement with the real
one, and that is what made this document dangerous. If the invocation needs
to change, change `boot.sh`.

`boot.sh` timing out **is** a failure now. `microkernel-initramfs` powers the
machine off when it is done, so a healthy boot exits by itself in a few
seconds and `boot.sh` returns 0. (This was inverted during stage 1, when the
rootfs dropped to a login prompt and waited forever.)

## Current state — keep this section honest as work lands

**Stage 1 (dev environment): complete**, 2026-09-20. The kernel builds from
this tree via `EXTERNALSRC` and boots on QEMU aarch64 `virt`.

**Stage 2 (minimal configuration): the config is found and verified**,
2026-09-20. `arch/arm64/configs/virt_min_defconfig`.

It was not written by hand. It is the output of a delta-debugging search
(`stage2-minimal-config/` in the coordinator repo, with the harness and the
evidence), and it is **1-minimal**: disabling any symbol it enables breaks
the build or the boot. That is stage 2's done-condition satisfied by
construction rather than asserted.

**There are two configs, differing by exactly one symbol.**
`virt_min_defconfig` runs a single freestanding binary and cannot be typed at;
`virt_shell_defconfig` adds `CONFIG_BINFMT_SCRIPT=y` and gives an interactive
busybox shell (with `microkernel-shell-image`). Use the shell one to develop
in; `virt_min_defconfig` is the number the project is judged on.

Measured, not estimated — kbuild records the source of every object in a
`.cmd` file, so the set of compiled translation units is exact:

| config | `.c`/`.S` files | code lines | + headers | `Image` | symbols |
|---|---|---|---|---|---|
| stock arm64 `defconfig` | 4,401 | 2,707,870 | 3,360,334 | 52,374,016 | 4,946 |
| **`virt_min_defconfig`** | **545** | **309,905** | **581,231** | **2,875,400** | **439** |
| `virt_shell_defconfig` | 546 | 310,001 | 581,327 | 2,875,400 | 440 |

**88.6% less compiled source than the stock defconfig — 8.7x fewer lines
running at EL1.** A shell costs +1 file (`fs/binfmt_script.c`, 159 lines), +96
lines of code and zero bytes of `Image`.

`BINFMT_SCRIPT` is needed only because `/init` is a `#!` script — the kernel
cannot exec it otherwise. An `/init` that is the busybox binary directly might
cost nothing; untested.

Verified through bitbake end to end, not just a direct `make`: the deployed
`Image` is byte-identical to the searched one, and `scripts/boot.sh` returns
0 with `Run /init as init process` followed by userspace writing to the
PL011 console and powering off.

Seven symbols carry it, each proven necessary: `BINFMT_ELF`,
`BLK_DEV_INITRD`, `PRINTK`, `SERIAL_AMBA_PL011`, `SERIAL_AMBA_PL011_CONSOLE`,
`TTY`, `RD_GZIP` — plus explicit disables for the 76 symbols Kconfig would
otherwise switch on by default.

**Two results from that search worth carrying forward:**

- **`CONFIG_VT` is not needed for a serial console.**
  `SERIAL_AMBA_PL011_CONSOLE` registers directly. `VT` was only ever on
  because it is `default y` under `TTY`, and it drags in `INPUT`, `HID`,
  `KEYBOARD_ATKBD`, seven `MOUSE_PS2_*` drivers and both PTY layers.
  Dropping it is most of the reduction.
- **`CONFIG_PRINTK` is required for the console to work at all**, not merely
  for kernel log messages. That constrains stage 6, which moves the console
  subsystem out of the kernel first: the printk core is entangled with it.

**The devicetree is now passed explicitly.**
`arch/arm64/boot/dts/qemu/qemu-virt-uart-min.dts` is built by the kernel
recipe (`KERNEL_DEVICETREE`) and passed by `scripts/boot.sh`. With a config
this small the drivers for anything extra are absent, so booting without a
`-dtb` also works — it is passed anyway so the device set is *declared*
rather than inferred from whatever QEMU generates, and the config and the DT
can be checked against each other. Its memory node hardcodes 256M, and the
DT wins over `-m`, which is why `boot.sh` uses `-m 256`.

`arch/arm64/configs/virt_uart_defconfig` still works (it builds and boots
through bitbake; that was checked) but it is superseded — 60 symbols of the
input/HID/PS2/PTY and decompressor machinery it enables are provably
unnecessary.

**Current milestone: stage 3 — GPIO (PL061) in userspace.** Config additions
needed to make that possible stay minimal, and go through the same
demonstrate-don't-assert bar.

A caveat before reusing `virt_min_defconfig` anywhere else: it is minimal
**for one freestanding init**. For an interactive shell use
`virt_shell_defconfig` — which is only one symbol larger, measured rather than
guessed. (An earlier version of this file claimed a shell would need
`MULTIUSER`, `PROC_FS`, `SYSFS` and `FILE_LOCKING`. The search eliminated all
four: `MULTIUSER` off merely stubs setuid, and ash needs neither `/proc` nor
`/sys` mounted to give a prompt.)

Earlier notes in this file claimed userspace "already works end to end — PID
1 exec, syscall ABI, `/dev/console` I/O — don't redo this." That is now
actually true and demonstrated, by `microkernel-initramfs` rather than by
whatever produced the original claim.

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
