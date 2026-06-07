---
title: Organize A Yocto Project With Layers And kas
description: Notes on separating Yocto metadata into layers, selecting a RISC-V machine, composing kas YAML files, and enabling optional patches or build flags.
tags:
- yocto
- kas
- bitbake
- layer
- riscv
- kernel
---

## Goal

- Understand the roles of:
  - layers
  - machines
  - distros
  - images
  - project-defined build modes
- Create a custom layer.
- Add a small out-of-tree kernel-module recipe.
- Register the layer in a kas project.
- Select builds with or without patches and debug flags.

## Configuration Concepts

| Concept | Main question | Typical location |
|---|---|---|
| Layer | Which metadata is available? | `meta-*/conf/layer.conf` |
| Machine | Which hardware or emulator is targeted? | `conf/machine/*.conf` |
| Distro | Which distribution-wide policy is used? | `conf/distro/*.conf` |
| Image | Which packages form the root filesystem? | `recipes-core/images/*.bb` |
| Recipe | How is one component built and packaged? | `recipes-*/*/*.bb` |
| Append | How does another layer modify a recipe? | `*.bbappend` |
| Mode | Which optional project configuration is enabled? | kas overlay YAML |

## Layer

- A layer is an ordered collection of metadata.
- It can contain:
  - recipes
  - recipe append files
  - machine definitions
  - distro configuration
  - image recipes
  - classes
  - configuration fragments
- Layer priority and append matching determine how metadata is combined.
- A custom project should normally keep its changes outside Poky.
- Avoid directly editing Poky because:
  - upgrades become difficult
  - project changes are mixed with upstream code
  - reproducing the change requires preserving a modified Poky checkout

## Machine

- `MACHINE` selects hardware-specific metadata.
- It can control:
  - CPU architecture and tuning
  - kernel provider
  - bootloader and firmware
  - device trees
  - kernel image type
  - QEMU options
  - machine-specific package dependencies

- This project starts with:

```bitbake
MACHINE = "qemuriscv64"
```

- `qemuriscv64` is supplied by existing OpenEmbedded metadata.
- A custom physical board would normally add:

```txt
meta-my-board/
`-- conf/
    `-- machine/
        `-- my-riscv-board.conf
```

## Distro And Image

- `DISTRO = "poky"` selects the reference distribution policy.
- `core-image-minimal` is an image target.
- They solve different problems:
  - machine describes hardware
  - distro describes system-wide policy
  - image describes root-filesystem contents

## What A Build Mode Means

- BitBake does not define one universal object named a build mode.
- A project can model modes by composing kas YAML files.
- Typical modes include:
  - normal versus debug
  - patched versus unpatched
  - feature enabled versus disabled
  - development versus production
- Keep the base configuration small.
- Put optional differences into overlay YAML files.

## Suggested Project Layout

```txt
yocto-riscv/
|-- kas/
|   |-- base.yml
|   `-- modes/
|       |-- debug.yml
|       `-- with-kernel-patch.yml
|-- meta-lab/
|   |-- conf/
|   |   `-- layer.conf
|   `-- recipes-kernel/
|       `-- hello-yocto/
|           |-- files/
|           |   |-- hello_yocto.c
|           |   `-- Makefile
|           `-- hello-yocto_0.1.bb
`-- meta-kernel-patch/
    |-- conf/
    |   `-- layer.conf
    `-- recipes-kernel/
        `-- linux/
            |-- files/
            |   `-- 0001-example-kernel-change.patch
            `-- linux-yocto_%.bbappend
```

## Create The Base Custom Layer

- Enter the kas shell:

```sh
kas shell kas/base.yml
```

- From the generated build directory:

```sh
bitbake-layers create-layer ../meta-lab
bitbake-layers show-layers
```

- `create-layer` generates `conf/layer.conf` and an example recipe.
- Remove the generated example recipe when it is not useful.
- Add the layer to kas YAML instead of relying on a manual edit to `bblayers.conf`.

## Minimal Kernel Module Source

- Create `meta-lab/recipes-kernel/hello-yocto/files/hello_yocto.c`:

```c
#include <linux/init.h>
#include <linux/module.h>

static int __init hello_yocto_init(void)
{
    pr_info("hello_yocto: loaded\n");
    return 0;
}

static void __exit hello_yocto_exit(void)
{
    pr_info("hello_yocto: unloaded\n");
}

module_init(hello_yocto_init);
module_exit(hello_yocto_exit);

MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Small Yocto module example");
```

## Kernel Module Makefile

- Create `meta-lab/recipes-kernel/hello-yocto/files/Makefile`:

```make
obj-m := hello_yocto.o

SRC := $(shell pwd)

all:
	$(MAKE) -C $(KERNEL_SRC) M=$(SRC) modules

modules_install:
	$(MAKE) -C $(KERNEL_SRC) M=$(SRC) modules_install

clean:
	$(MAKE) -C $(KERNEL_SRC) M=$(SRC) clean
```

## Kernel Module Recipe

- Create `meta-lab/recipes-kernel/hello-yocto/hello-yocto_0.1.bb`:

```bitbake
SUMMARY = "Small out-of-tree kernel module"
LICENSE = "CLOSED"

inherit module

SRC_URI = " \
    file://hello_yocto.c \
    file://Makefile \
"

S = "${UNPACKDIR}"
```

- `inherit module` supplies the standard out-of-tree module workflow.
- The resulting module package is normally named from `hello_yocto.ko`:

```txt
kernel-module-hello-yocto
```

## Install The Module In The Image

- Add `meta-lab/recipes-core/images/core-image-minimal.bbappend`:

```bitbake
IMAGE_INSTALL:append = " kernel-module-hello-yocto"
```

- Optional automatic loading:

```bitbake
KERNEL_MODULE_AUTOLOAD += "hello_yocto"
```

- Manual loading is clearer while testing:

```sh
modprobe hello_yocto
dmesg | tail
```

## Base kas Configuration

- Create `kas/base.yml`:

```yaml
header:
  version: 19

machine: qemuriscv64
distro: poky
target:
  - core-image-minimal

repos:
  poky:
    url: https://git.yoctoproject.org/poky.git
    tag: yocto-6.0
    layers:
      meta:
      meta-poky:

  project:
    layers:
      meta-lab:

local_conf_header:
  base-image: |
    IMAGE_FSTYPES:append = " ext4"
```

- The `project` repository has no URL because the kas file and layers live in the current Git repository.
- The base build includes `meta-lab` but does not include the optional kernel-patch layer.

## Debug Mode

- Create `kas/modes/debug.yml`:

```yaml
header:
  version: 19

local_conf_header:
  debug-mode: |
    EXTRA_IMAGE_FEATURES += "debug-tweaks tools-debug"
    DEBUG_BUILD = "1"
```

- Build the base configuration with debug settings:

```sh
kas build kas/base.yml:kas/modes/debug.yml
```

- A recipe-specific feature can be enabled when that recipe defines a matching `PACKAGECONFIG` option:

```bitbake
PACKAGECONFIG:append:pn-<recipe> = " trace"
```

## Optional Kernel-Patch Layer

- Create the second layer:

```sh
bitbake-layers create-layer ../meta-kernel-patch
```

- Add:

```txt
meta-kernel-patch/
`-- recipes-kernel/
    `-- linux/
        |-- files/
        |   `-- 0001-example-kernel-change.patch
        `-- linux-yocto_%.bbappend
```

- Example `linux-yocto_%.bbappend`:

```bitbake
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"
SRC_URI += "file://0001-example-kernel-change.patch"
```

- Confirm the selected kernel recipe before naming the append:

```sh
bitbake-layers show-recipes virtual/kernel
bitbake-getvar PREFERRED_PROVIDER_virtual/kernel
```

- If the provider is not `linux-yocto`, the append filename must match the real recipe.

## With-Patch Mode

- Create `kas/modes/with-kernel-patch.yml`:

```yaml
header:
  version: 19

repos:
  project:
    layers:
      meta-kernel-patch:
```

- Build without the optional patch:

```sh
kas build kas/base.yml
```

- Build with the optional patch:

```sh
kas build kas/base.yml:kas/modes/with-kernel-patch.yml
```

- Combine patch and debug modes:

```sh
kas build \
    kas/base.yml:kas/modes/with-kernel-patch.yml:kas/modes/debug.yml
```

- kas merges command-line configurations from left to right.
- Later configuration can add to or override earlier configuration.

## Recipe Patch Versus kas Repository Patch

- A recipe patch:
  - is listed in `SRC_URI`
  - is applied during the recipe's `do_patch`
  - changes the software being built

- A kas repository patch:
  - is declared under a kas repository's `patches` entry
  - changes the checked-out metadata repository itself
  - is useful for temporarily patching a layer before BitBake parses it

- Prefer recipe patches for kernel and application source changes.
- Use kas repository patches when the change must modify an external layer repository before parsing.

## Confirm A Mode

```sh
kas dump kas/base.yml:kas/modes/debug.yml
kas shell kas/base.yml:kas/modes/with-kernel-patch.yml
```

- Inside the shell:

```sh
bitbake-layers show-layers
bitbake-layers show-appends
bitbake-getvar EXTRA_IMAGE_FEATURES
bitbake -e virtual/kernel | grep '0001-example-kernel-change.patch'
```

## Key Lessons

- Put project changes in custom layers, not Poky.
- `MACHINE`, `DISTRO`, and image targets describe different dimensions.
- A build mode is a project convention implemented cleanly with kas overlays.
- The base YAML should describe the normal build.
- Optional YAML files should add patches, flags, or development behavior.
- An optional patch layer gives the clearest patched versus unpatched build boundary.

## References

- [kas project configuration and includes](https://kas.readthedocs.io/en/latest/userguide/project-configuration.html)
- [Yocto Project layer documentation](https://docs.yoctoproject.org/6.0/dev-manual/layers.html)
- [Yocto Project out-of-tree module documentation](https://docs.yoctoproject.org/6.0/kernel-dev/common.html#working-with-out-of-tree-modules)

## Previous Reading

- [Build RISC-V Linux With Yocto And QEMU](./yocto-build-linux-and-boot-qemu)
