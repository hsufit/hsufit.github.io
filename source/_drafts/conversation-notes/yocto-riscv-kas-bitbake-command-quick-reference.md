---
title: Yocto Kas And Debug Commands Quick Reference
description: Compact commands for reproducing a kas build, iterating on persistent recipe source, forcing BitBake tasks, checking patches, and locating intermediate and final RISC-V artifacts.
tags:
- yocto
- riscv
- kas
- bitbake
- devtool
- debugging
- quick-reference
---

## Placeholders

- `<config>`:

```txt
kas/base.yml
```

- `<mode>`:

```txt
kas/modes/debug.yml
```

- `<recipe>`:

```txt
hello-yocto
```

- Combined kas configurations use a colon:

```txt
kas/base.yml:kas/modes/debug.yml
```

## Reproduce A Build

```sh
git clone <project-url>
cd <project-directory>
kas checkout <config>
kas build <config>
```

- Pin floating branches and tags:

```sh
kas lock --update <config>
git add kas/*.lock.yml
git commit -m "Pin kas repository revisions"
```

- kas automatically uses the matching lockfile on later builds.
- Commit the lockfile so another machine resolves the same repository commits.

- Inspect the merged configuration:

```sh
kas dump <config>
kas dump <config>:<mode>
```

- Show the checked-out revision of every kas-managed repository:

```sh
kas for-all-repos <config> 'git status --short --branch'
kas for-all-repos <config> 'git rev-parse HEAD'
```

## Enter The Build Environment

```sh
kas shell <config>
```

- Run one command without keeping an interactive shell:

```sh
kas shell <config> -c 'bitbake virtual/kernel'
```

## Common Build Targets

| Goal | Command |
|---|---|
| Build the configured kas target | `kas build <config>` |
| Build an image | `bitbake core-image-minimal` |
| Build the selected kernel | `bitbake virtual/kernel` |
| Build one recipe | `bitbake <recipe>` |
| Build an SDK installer | `bitbake core-image-minimal -c populate_sdk` |
| Boot QEMU | `runqemu qemuriscv64 core-image-minimal ext4 nographic` |

## Inspect Project Configuration

```sh
bitbake-getvar MACHINE
bitbake-getvar DISTRO
bitbake-getvar TARGET_ARCH
bitbake-getvar IMAGE_FSTYPES
bitbake-getvar PACKAGE_CLASSES
```

- Expected target values include:

```txt
MACHINE="qemuriscv64"
TARGET_ARCH="riscv64"
```

## Inspect Layers And Recipes

```sh
bitbake-layers show-layers
bitbake-layers show-recipes
bitbake-layers show-recipes <recipe>
bitbake-layers show-recipes virtual/kernel
bitbake-layers show-appends
```

- Find the file that defines a recipe:

```sh
bitbake-getvar -r <recipe> FILE
```

- List the tasks available for a recipe:

```sh
bitbake -c listtasks <recipe>
```

## Inspect Final Variable Values

```sh
bitbake-getvar -r <recipe> PV
bitbake-getvar -r <recipe> SRC_URI
bitbake-getvar -r <recipe> WORKDIR
bitbake-getvar -r <recipe> S
bitbake-getvar -r <recipe> B
bitbake-getvar -r <recipe> T
bitbake-getvar -r <recipe> D
```

- `bitbake-getvar` also shows how assignments, appends, overrides, and configuration files produced the final value.
- Dump the complete recipe environment:

```sh
bitbake -e <recipe> > <recipe>.env
```

- Search it:

```sh
grep '^SRC_URI=' <recipe>.env
grep '^PACKAGECONFIG=' <recipe>.env
grep '^CFLAGS=' <recipe>.env
```

## Check A Patch

- Confirm that the patch reaches the recipe:

```sh
bitbake -e <recipe> | grep '<patch-name>'
bitbake-getvar -r <recipe> SRC_URI
bitbake-layers show-appends
```

- Run only through patching:

```sh
bitbake -f -c patch <recipe>
```

- Find the patch log and patched source:

```sh
bitbake-getvar -r <recipe> T
bitbake-getvar -r <recipe> S
```

```txt
${T}/log.do_patch
${S}/
```

- For a kernel patch:

```sh
bitbake -e virtual/kernel | grep '<patch-name>'
bitbake -f -c patch virtual/kernel
```

## Normal Edit And Build Loop

- Edit source or metadata stored in a custom layer.
- Let BitBake detect the changed task signature:

```sh
bitbake <recipe>
bitbake core-image-minimal
```

- This should be the default loop for `file://` source stored beside a recipe.

## Persistent Git Source Loop With devtool

- Use `devtool` when a recipe fetches a larger source tree and you want a persistent Git checkout.

- Extract the recipe source into a Git working tree:

```sh
devtool modify <recipe> ~/src/<recipe>
cd ~/src/<recipe>
git status
```

- The workspace redirects the recipe to this persistent source tree.
- Edit files normally, then build:

```sh
devtool build <recipe>
```

- BitBake can also build the workspace recipe:

```sh
bitbake <recipe>
```

- After the first build, useful links can appear in the source tree:
  - `oe-logs`
  - `oe-workdir`

## Use An Existing Git Checkout

- If the source already exists as a local Git repository:

```sh
devtool modify -n <recipe> ~/src/<recipe>
```

- `-n` tells `devtool` not to extract another copy.
- This is the useful loop when you want to:
  - keep source history
  - edit repeatedly
  - force compilation without refetching source
  - inspect commits and diffs directly

## Force Compile Repeatedly

- After editing the persistent source:

```sh
bitbake -f -c compile <recipe>
bitbake <recipe>
```

- The first command forces `do_compile`.
- The second command completes install, packaging, and deployment tasks.
- A shorter task-invalidating build is:

```sh
bitbake -C compile <recipe>
```

- Forced tasks are marked as tainted.
- Use force during investigation; use correct source tracking and task signatures for stable production builds.

## Finish Or Leave A devtool Workspace

- Commit the source changes before finishing:

```sh
cd ~/src/<recipe>
git add .
git commit
```

- Export the committed changes into the custom layer and leave the workspace:

```sh
devtool finish <recipe> ../meta-lab
```

- Stop overriding the recipe but leave the source directory untouched:

```sh
devtool reset <recipe>
```

## Quick Temporary Source Inspection

- A Git recipe is commonly unpacked under `${S}`, often ending in:

```txt
${WORKDIR}/git
```

- Find the actual location:

```sh
bitbake-getvar -r <recipe> S
```

- Editing `${WORKDIR}/git` can be useful for a quick experiment.
- It is not a durable development workflow because clean, unpack, or patch tasks can replace it.
- Use `devtool modify` when changes must survive repeated builds.

## Task-Level Build Commands

| Task | Command |
|---|---|
| Fetch source | `bitbake -c fetch <recipe>` |
| Unpack source | `bitbake -c unpack <recipe>` |
| Apply patches | `bitbake -c patch <recipe>` |
| Configure | `bitbake -c configure <recipe>` |
| Compile | `bitbake -c compile <recipe>` |
| Install into `${D}` | `bitbake -c install <recipe>` |
| Create packages | `bitbake -c package <recipe>` |
| Open build shell | `bitbake -c devshell <recipe>` |
| Show tasks | `bitbake -c listtasks <recipe>` |

## Clean And Rebuild

| Strength | Commands | Meaning |
|---|---|---|
| Normal | `bitbake <recipe>` | Rebuild tasks whose signatures changed |
| Force one task | `bitbake -f -c compile <recipe>` | Run the selected task even when its stamp is valid |
| Invalidate from task | `bitbake -C compile <recipe>` | Rebuild the task and dependent work |
| Clean work output | `bitbake -c clean <recipe>` | Remove recipe work but retain reusable shared state |
| Remove local shared state | `bitbake -c cleansstate <recipe>` | Force a substantially cleaner rebuild |

- Start with the normal build.
- Use `cleansstate` only when incremental state is genuinely suspect.

## Locate Downloaded Source

- Show the download cache:

```sh
bitbake-getvar DL_DIR
```

- Common contents:

```txt
${DL_DIR}/git2/       cached Git repositories
${DL_DIR}/*.tar.*     downloaded source archives
```

- `DL_DIR` is a cache, not the patched working source tree.

## Locate Recipe Work And Logs

```sh
bitbake-getvar -r <recipe> WORKDIR
bitbake-getvar -r <recipe> S
bitbake-getvar -r <recipe> B
bitbake-getvar -r <recipe> T
bitbake-getvar -r <recipe> D
bitbake-getvar -r <recipe> PKGDEST
```

| Variable | Typical content |
|---|---|
| `WORKDIR` | All temporary work for one recipe version |
| `S` | Unpacked and patched source, often `${WORKDIR}/git` |
| `B` | Build directory and generated objects |
| `T` | Task logs and generated run scripts |
| `D` | Files installed by `do_install` before packaging |
| `PKGDEST` | Split package trees, commonly under `packages-split` |

- Important log files:

```txt
${T}/log.do_fetch
${T}/log.do_patch
${T}/log.do_configure
${T}/log.do_compile
${T}/log.do_install
${T}/log.do_package
```

## Locate RPM Or Other Binary Packages

- Confirm the selected package format:

```sh
bitbake-getvar PACKAGE_CLASSES
```

- RPM output:

```sh
bitbake-getvar DEPLOY_DIR_RPM
ls -R tmp/deploy/rpm/
```

- Other package formats use:

```txt
tmp/deploy/ipk/
tmp/deploy/deb/
```

- Inspect package ownership and contents:

```sh
oe-pkgdata-util list-pkgs
oe-pkgdata-util list-pkg-files <package>
oe-pkgdata-util find-pkg <recipe-or-runtime-name>
```

## Locate The Image Root Filesystem During Assembly

```sh
bitbake-getvar -r core-image-minimal WORKDIR
bitbake-getvar -r core-image-minimal IMAGE_ROOTFS
```

- The image rootfs staging tree is commonly below the image recipe's work directory as:

```txt
${WORKDIR}/rootfs/
```

- This is an intermediate tree.
- The deployable ext4 file is produced later.

## Locate Final Kernel And Image Outputs

```sh
bitbake-getvar DEPLOY_DIR_IMAGE
ls -lh tmp/deploy/images/qemuriscv64/
```

- Final RISC-V outputs include:

```txt
Image
core-image-minimal-qemuriscv64.rootfs.ext4
qemuboot.conf
```

- Confirm the kernel image type:

```sh
bitbake-getvar -r virtual/kernel KERNEL_IMAGETYPE
```

## Build And Boot After A Recipe Change

```sh
bitbake <recipe>
bitbake core-image-minimal
runqemu qemuriscv64 core-image-minimal ext4 nographic
```

- Rebuilding a package does not automatically rewrite an already completed ext4 image.
- Rebuild the image when the changed package must appear in the root filesystem.

## Dependency And Signature Debugging

- Generate dependency information:

```sh
bitbake -g <recipe>
```

- Common generated files:

```txt
pn-buildlist
task-depends.dot
recipe-depends.dot
```

- Show why a recipe will rebuild:

```sh
bitbake -S printdiff <recipe>
```

- Compare saved task signatures:

```sh
bitbake-diffsigs <sigdata-a> <sigdata-b>
```

## Short Development Loop

```sh
kas shell <config>
devtool modify -n <recipe> ~/src/<recipe>

# edit files in ~/src/<recipe>

bitbake -f -c compile <recipe>
bitbake <recipe>
bitbake core-image-minimal
runqemu qemuriscv64 core-image-minimal ext4 nographic
```

## Memory Rules

- `kas lock` pins repositories for reproduction.
- `kas dump` shows the merged project configuration.
- `bitbake-getvar` explains one final variable.
- `bitbake -e` dumps the complete recipe environment.
- `devtool modify` creates or connects a persistent editable source tree.
- `${WORKDIR}/git` is temporary; a `devtool` source tree is the better edit loop.
- `${D}` is installed recipe content before package splitting.
- `tmp/deploy/rpm` contains package files.
- `tmp/deploy/images/qemuriscv64` contains final boot artifacts.
- Rebuild the image after changing a package that must appear in ext4.

## References

- [kas commands and lockfiles](https://kas.readthedocs.io/en/latest/userguide/plugins.html)
- [Yocto Project devtool workflow](https://docs.yoctoproject.org/dev/dev-manual/devtool.html)
- [Yocto Project debugging tools](https://docs.yoctoproject.org/6.0/dev-manual/debugging.html)
- [Yocto Project task reference](https://docs.yoctoproject.org/6.0/ref-manual/tasks.html)

## Previous Reading

- [Build RISC-V Linux With Yocto And QEMU](./yocto-build-linux-and-boot-qemu)
- [Organize A Yocto Project With Layers And kas](./yocto-layers-machines-modes-and-kas-configuration)
