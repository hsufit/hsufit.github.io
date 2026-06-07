---
title: Linux Kernel Module Hello World
description: Notes about a minimal loadable Linux kernel module, init and exit callbacks, kernel messages, and dmesg monitoring.
tags:
- linux
- kernel
- device-driver
- kernel-module
- dmesg
---

## Goal

- Build the smallest useful loadable kernel module.
- Observe when the kernel calls its initialization and cleanup functions.
- Learn the basic development loop:

```txt
write source
    -> build module
    -> insert module
    -> observe kernel log
    -> remove module
```

## Minimal Module

```c
#include <linux/init.h>
#include <linux/module.h>

static int __init hello_init(void)
{
    pr_info("hello_driver: module loaded\n");
    return 0;
}

static void __exit hello_exit(void)
{
    pr_info("hello_driver: module unloaded\n");
}

module_init(hello_init);
module_exit(hello_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("hsufit");
MODULE_DESCRIPTION("Minimal Linux kernel module");
```

## Initialization Function

- `module_init(hello_init)` registers `hello_init()` as the initialization entry point.
- For a loadable module, the kernel calls it when `insmod` or `modprobe` loads the module.
- The function returns:
  - `0` for success
  - a negative Linux error code if initialization fails

```c
static int __init hello_init(void)
```

- `__init` marks code that is only needed during initialization.
- The kernel can release this section after initialization completes.

## Exit Function

- `module_exit(hello_exit)` registers the cleanup function.
- The kernel calls it when the module is removed with `rmmod`.
- Cleanup code must release resources acquired during initialization.

```c
static void __exit hello_exit(void)
```

- `__exit` marks code used only when a module exits.
- An exit callback returns `void`; module cleanup cannot report a recoverable failure to `rmmod`.

## Kernel Messages

- Kernel code does not use the userspace `printf()` function.
- `pr_info()` writes an informational message to the kernel log:

```c
pr_info("hello_driver: module loaded\n");
```

- Related helpers include:

| Helper | Meaning |
|---|---|
| `pr_err()` | error |
| `pr_warn()` | warning |
| `pr_info()` | informational message |
| `pr_debug()` | debug message, subject to debug configuration |

- Prefixing messages with the module or driver name makes logs easier to filter.

## Makefile

```make
obj-m += hello_driver.o

KDIR ?= /lib/modules/$(shell uname -r)/build
PWD := $(shell pwd)

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

- `obj-m` declares a loadable module target.
- Building `hello_driver.c` produces `hello_driver.ko`.
- The external-module build uses the build directory for the running kernel.

## Build

```sh
make
```

- Confirm that the module was generated:

```sh
ls -l hello_driver.ko
```

- Kernel headers or the kernel development package for the running kernel must be installed.

## Observe New Kernel Messages

- Start a terminal that waits for only new kernel messages:

```sh
sudo dmesg -W
```

- `-W` means `--follow-new`.
- It waits and prints messages added after the command starts.
- This is convenient for watching module load and unload events without scrolling through the entire existing ring buffer.

## Insert The Module

```sh
sudo insmod hello_driver.ko
```

- The kernel calls `hello_init()`.
- The monitoring terminal should show:

```txt
hello_driver: module loaded
```

- Confirm that the module is loaded:

```sh
lsmod | grep hello_driver
```

## Remove The Module

```sh
sudo rmmod hello_driver
```

- `rmmod` uses the module name, normally without the `.ko` extension.
- The kernel calls `hello_exit()`.
- The monitoring terminal should show:

```txt
hello_driver: module unloaded
```

## Inspect Module Metadata

```sh
modinfo hello_driver.ko
```

- This can display:
  - license
  - author
  - description
  - filename
  - module dependencies
  - supported parameters

## `insmod` Versus `modprobe`

- `insmod` loads the exact `.ko` path supplied by the user.
- `modprobe` loads modules by name and handles declared dependencies.
- For a small module in the current directory, `insmod` is convenient.
- Installed production modules are normally managed with `modprobe`.

## Common Problems

- `Operation not permitted`:
  - use sufficient privileges
  - check whether module loading is restricted

- `Invalid module format`:
  - the module may have been built for a different kernel version or configuration
  - compare `uname -r` and `modinfo hello_driver.ko`

- Kernel log access denied:
  - use `sudo dmesg -W`
  - some systems restrict access through `kernel.dmesg_restrict`

- Module is in use:
  - another kernel component or open device may still hold a module reference
  - later driver notes explain why `.owner = THIS_MODULE` matters

## Key Lesson

- `module_init()` and `module_exit()` define a module's lifecycle.
- `pr_info()` records observable kernel messages.
- `dmesg -W` is useful for monitoring only new messages during driver development.
- A real driver extends this lifecycle by registering resources during init and unregistering them in reverse order during exit.

## References

- [Linux kernel driver entry and exit points](https://docs.kernel.org/driver-api/basics.html)
- [Linux kernel module initialization notes](https://docs.kernel.org/kernel-hacking/hacking.html)
- [`dmesg(1)` manual](https://man7.org/linux/man-pages/man1/dmesg.1.html)

## Previous Reading

- None. This note is the starting point.
