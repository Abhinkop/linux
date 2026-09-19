---
description: Build the current defconfig and boot it under the project's fixed QEMU invocation, showing serial output.
---

Build the kernel and boot-test it exactly as the `boot-verifier` subagent
does:

1. `ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make -j$(nproc) Image dtbs`
2. Boot with:
   ```
   timeout 30 qemu-system-aarch64 -M virt -cpu cortex-a57 -smp 1 -m 256 -nographic \
       -kernel arch/arm64/boot/Image \
       -dtb arch/arm64/boot/dts/qemu/qemu-virt-uart-min.dtb \
       -initrd $ARGUMENTS
   ```
   (pass the initramfs path as the command's argument, e.g.
   `/boot-test rootfs.cpio.gz`)
3. Show the full serial output, and call out explicitly whether it looks
   like a clean boot or not — don't just dump the log with no verdict.
