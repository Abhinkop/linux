---
description: Build the kernel and rootfs via the project container and boot them, showing serial output.
---

Build and boot-test exactly as the `boot-verifier` subagent does. Nothing
runs on the host — there is no `make`, `gcc` or `qemu-system-aarch64` here.

Run from the coordinator workspace root (`..` from the kernel tree):

1. Build:
   ```
   scripts/dev-container.sh bash scripts/build.sh core-image-minimal
   ```
   (`scripts/build.sh linux-microkernel` for kernel-only.) Dev mode builds
   straight from the `linux/` tree — no commit or push needed.

2. Boot, with an argument for the timeout in seconds (default 180, e.g.
   `/boot-test 240`):
   ```
   scripts/dev-container.sh bash scripts/boot.sh $ARGUMENTS
   ```
   `boot.sh` owns the QEMU invocation — do not write your own command line.

3. Show the serial output and give an explicit verdict — don't dump the log
   with no conclusion. Note that `boot.sh` being killed on its timeout is
   expected: init reaches a login prompt and waits there. Judge by the log
   contents, not the exit path.
