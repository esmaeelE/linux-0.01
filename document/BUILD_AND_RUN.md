# Compiling & Running Linux 0.01 on Debian 13 + QEMU

## The Problem

Linux 0.01 was written for GCC 1.x and GNU AS on 32-bit i386. Debian 13 ships GCC 14+ which drops support for old syntax (`-fcombine-regs`, `-fstrength-reduce`, K&R function declarations). You cannot compile this with the default system toolchain.

## Option 1: Docker + GCC 2.95 (Recommended)

Fully reproducible, zero host system pollution:

```bash
# Build in a container with old GCC
docker run --rm -v "$(pwd)":/src -w /src gcc:2.95 bash -c "make clean && make"

# Or build just the image
docker run --rm -v "$(pwd)":/src -w /src gcc:2.95 bash -c "make Image"
```

Run with QEMU on the host:

```bash
# Boot from floppy image
qemu-system-i386 -fda Image -boot a

# Or boot from kernel image directly
qemu-system-i386 -kernel Image

# Serial console (no GUI needed)
qemu-system-i386 -fda Image -nographic
```

## Option 2: Install Compatible Toolchain on Host

```bash
sudo apt install build-essential gcc-multilib nasm qemu-system-x86
```

Then patch the Makefile:

- CFLAGS: add `-m32 -fno-stack-protector -fno-pie -no-pie`
- Remove `-fcombine-regs` (removed in GCC 4+)
- Remove or conditionalize `-fstrength-reduce` (ignored since GCC 4)
- Boot `.s` files need `--32` flag for GAS (or use `nasm`)
- The `tools/build.c` `chmem` call needs to be skipped or replaced

## Option 3: Create Bootable Hard Disk Image

```bash
# After building Image:
dd if=/dev/zero of=disk.img bs=1M count=50
mkfs.ext2 -F disk.img
mkdir /tmp/rootfs
# mount and populate rootfs, then:
dd if=Image of=boot.img bs=512 count=2880
qemu-system-i386 -hda disk.img -fda boot.img
```

## QEMU Options

| Flag | Purpose |
|------|---------|
| `-fda Image` | Boot from floppy image |
| `-kernel Image` | Load kernel directly (skips boot sector) |
| `-hda disk.img` | Attach hard disk |
| `-nographic` | Serial console, no GUI |
| `-m 16` | Set RAM to 16MB (default is enough for 0.01) |
| `-s` | Enable GDB debugging on port 1234 |
| `-S -s` | Pause at start, wait for GDB connect |

## GDB Debugging

```bash
qemu-system-i386 -fda Image -s -S &
gdb vmlinux -ex "target remote :1234" -ex "break main" -ex "continue"
```

## Notes

- Linux 0.01 expects a floppy-booted system with root FS on floppy
- The kernel boots into `task0` (idle), forks `init` which runs `/bin/sh`
- You need a minimal root filesystem (busybox or the original Minix tools) for a usable shell
- `make Image` creates a raw binary from the boot+system binary