---
title: Build RISC-V Linux With Yocto And QEMU
description: Relationship between Yocto, OpenEmbedded, BitBake, Poky, and kas, followed by a minimal Poky-only qemuriscv64 image build and QEMU boot flow.
tags:
- yocto
- poky
- riscv
- qemu
- linux
---

## Goal

- Understand which tool owns each part of a Yocto build.
- Choose the correct RISC-V toolchain model.
- Use Poky directly without kas or a custom layer.
- Build a Linux kernel and minimal root filesystem for `qemuriscv64`.
- Boot the result with `runqemu`.

## Relationship Between Yocto, OpenEmbedded, BitBake, Poky, And kas

- The Yocto Project is the wider project and ecosystem.
  - It provides documentation, reference metadata, tools, releases, and development conventions.

- OpenEmbedded is the build-system foundation.
  - OE-Core supplies core metadata, classes, recipes, and machine support.

- BitBake is the task executor.
  - It parses recipes and configuration.
  - It resolves dependencies.
  - It runs tasks such as fetch, patch, configure, compile, package, and image creation.

- Poky is the Yocto Project reference distribution and integration repository.
  - It contains BitBake.
  - It contains OE-Core's `meta` layer.
  - It contains `meta-poky` and reference BSP metadata.
  - It provides a known-compatible starting point, not a separate build engine.

- kas is a project orchestration tool around BitBake-based builds.
  - It reads one or more YAML configuration files.
  - It clones and pins repositories.
  - It selects layers, machine, distro, and build targets.
  - It generates the BitBake build configuration.
  - It then invokes BitBake or opens a configured shell.

- kas does not replace Poky or BitBake.
- kas is useful for larger reproducible projects, but it is not required for this minimal build.

```txt
Minimal flow:

Poky
  -> oe-init-build-env
  -> BitBake
  -> kernel + packages + root filesystem

Larger project flow:

kas YAML
  -> Poky and other repositories
  -> generated BitBake configuration
  -> BitBake
```

## RISC-V Toolchain Choice

- Yocto normally builds and uses its own cross-toolchain from metadata.
  - A separately installed RISC-V cross-compiler is usually not required for a normal image build.
  - The build host still needs its native compiler and the standard Yocto host dependencies.

- If an external toolchain is used manually, distinguish Linux from bare metal:

| Toolchain kind | Typical prefix | Runtime model | Appropriate use |
|---|---|---|---|
| Linux | `riscv64-unknown-linux-gnu-` | Linux ABI with glibc or another Linux C library | Linux kernel, kernel modules, and Linux userspace |
| Yocto SDK | `riscv64-poky-linux-` or tune-specific equivalent | Matches the Yocto image and sysroot | Applications and libraries for the generated image |
| Bare metal | `riscv64-unknown-elf-` | ELF with Newlib or no operating-system ABI | Firmware, boot code, and standalone programs |

- A Linux kernel itself does not link against userspace glibc.
- However, a complete Linux system also needs Linux userspace binaries and a matching sysroot.
- For this workflow, use Yocto's generated Linux toolchain or a Linux-targeting RISC-V toolchain.
- Do not use a bare-metal `unknown-elf` compiler for normal Linux userspace applications.

## Clone Poky

```sh
mkdir -p ~/yocto-lab
cd ~/yocto-lab
git clone -b yocto-6.0 https://git.yoctoproject.org/poky.git
cd poky
```

- `yocto-6.0` selects the Yocto 6.0 Wrynose release.
- This note assumes the build host already has the normal Yocto host tools and packages.
- No kas configuration or custom layer is required.

## Initialize The Poky Build Environment

```sh
source oe-init-build-env build-riscv
```

- This creates and enters:

```txt
~/yocto-lab/poky/build-riscv/
```

- Run this command again whenever starting a new shell.

## Select The RISC-V QEMU Machine

- Add these settings to `conf/local.conf`:

```bitbake
MACHINE = "qemuriscv64"
IMAGE_FSTYPES:append = " ext4"
EXTRA_IMAGE_FEATURES += "debug-tweaks"
```

- `debug-tweaks` is useful for a local QEMU lab.
- Do not enable it in a production image.

## Confirm The Configuration

```sh
bitbake-getvar MACHINE
bitbake-getvar TARGET_ARCH
```

- Expected values:

```txt
MACHINE="qemuriscv64"
TARGET_ARCH="riscv64"
```

## Build The Complete Minimal Image

```sh
bitbake core-image-minimal
```

- The image build also builds:
  - Yocto's internal RISC-V cross-toolchain
  - the selected Linux kernel
  - target packages
  - the root filesystem
  - QEMU boot metadata

## Build Only The Kernel

```sh
bitbake virtual/kernel
```

- `virtual/kernel` resolves to the kernel provider selected for `qemuriscv64`.
- A kernel alone is not a complete Linux boot test.
- Build `core-image-minimal` before the first boot so a root filesystem is available.

## Confirm The Kernel Image Type

```sh
bitbake-getvar -r virtual/kernel KERNEL_IMAGETYPE
```

- It should normally resolve to:

```txt
Image
```

## Find The Final Outputs

```sh
ls -lh tmp/deploy/images/qemuriscv64/
```

- Important outputs include:
  - `Image`
  - `core-image-minimal-qemuriscv64.rootfs.ext4`
  - `qemuboot.conf` or a machine-specific equivalent
  - versioned files and stable symbolic links

## Boot The RISC-V Image

```sh
runqemu qemuriscv64 core-image-minimal ext4 nographic
```

- `runqemu` selects the matching kernel, root filesystem, firmware, and QEMU arguments from the deploy directory.
- `nographic` uses the current terminal as the serial console.

## Verify The Guest

```sh
uname -m
uname -a
cat /etc/os-release
cat /proc/cmdline
```

- `uname -m` should report:

```txt
riscv64
```

## Minimal Build Flow

```txt
Poky checkout
    -> oe-init-build-env
    -> conf/local.conf
    -> Yocto cross-toolchain
    -> Linux Image
    -> core-image-minimal ext4
    -> runqemu
    -> RISC-V Linux guest
```

## Key Lessons

- Poky supplies a compatible reference set of BitBake and OpenEmbedded metadata.
- BitBake performs the actual build tasks.
- kas is optional and is intentionally not used in this minimal build.
- Yocto normally builds its own target cross-toolchain.
- Linux userspace needs a Linux ABI toolchain and matching sysroot.
- Bare-metal `unknown-elf` toolchains are for firmware and standalone programs.
- A bootable Linux test needs both the kernel and a root filesystem.

## References

- [Yocto Project quick build](https://docs.yoctoproject.org/6.0/brief-yoctoprojectqs/index.html)
- [Yocto Project 6.0 release information](https://docs.yoctoproject.org/6.0/migration-guides/release-6.0.html)
- [Yocto Project QEMU documentation](https://docs.yoctoproject.org/6.0/dev-manual/qemu.html)
- [RISC-V GNU toolchain](https://github.com/riscv-collab/riscv-gnu-toolchain)
- [Yocto SDK toolchain overview](https://docs.yoctoproject.org/6.0/sdk-manual/intro.html)

## Previous Reading

- None required. This is the foundation note for the Yocto RISC-V workflow.
