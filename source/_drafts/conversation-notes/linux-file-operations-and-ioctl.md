---
title: Linux File Operations and ioctl
description: Notes about struct file_operations, VFS callback dispatch, read and write data transfer, ioctl command design, and callback implementation.
tags:
- linux
- kernel
- device-driver
- file-operations
- ioctl
---

## Goal

- Understand how userspace file operations reach driver code.
- Learn how to implement:
  - `.open`
  - `.release`
  - `.read`
  - `.write`
  - `.unlocked_ioctl`
- Understand the important fields in `struct file_operations`.

## Callback Registration, Not Function Overwriting

- A driver does not overwrite the kernel's `read()`, `write()`, or `ioctl()` system calls.
- It implements callbacks and registers their addresses in `struct file_operations`.
- The VFS dispatches an operation through the `struct file` associated with the file descriptor:

```txt
userspace system call
    -> VFS
    -> file->f_op
    -> driver callback
```

## Userspace To Driver Mapping

| Userspace operation | Common `file_operations` callback |
|---|---|
| `open()` | `.open` |
| `close()` | `.release` when the last file reference closes |
| `read()` | `.read` or `.read_iter` |
| `write()` | `.write` or `.write_iter` |
| `ioctl()` | `.unlocked_ioctl` |
| 32-bit `ioctl()` on a 64-bit kernel | `.compat_ioctl` |
| `lseek()` | `.llseek` |
| `poll()` or `select()` | `.poll` |
| `mmap()` | `.mmap` |

## Important `struct file_operations` Fields

```c
static const struct file_operations mydev_fops = {
    .owner          = THIS_MODULE,
    .open           = mydev_open,
    .release        = mydev_release,
    .read           = mydev_read,
    .write          = mydev_write,
    .unlocked_ioctl = mydev_ioctl,
    .llseek         = no_llseek,
};
```

- `.owner`:
  - normally set to `THIS_MODULE`
  - helps prevent removal while operations still refer to module code

- `.open`:
  - called when the VFS opens the device inode
  - receives `struct inode *` and `struct file *`
  - can initialize `file->private_data`

- `.release`:
  - called when the final reference to an open file is released
  - cleans up per-open state

- `.read` and `.write`:
  - transfer byte-oriented data
  - use user-copy helpers instead of directly dereferencing userspace pointers

- `.unlocked_ioctl`:
  - handles device-specific control commands
  - the name distinguishes it from the historical Big Kernel Lock implementation

- `.llseek`:
  - controls file-position changes
  - `no_llseek` is appropriate when seeking has no meaning

## `struct inode` Versus `struct file`

- `struct inode` represents the underlying filesystem object.
- `struct file` represents one open file description.

```txt
device inode
    -> identifies major/minor device

open file object
    -> current position
    -> access flags
    -> private_data
    -> file_operations
```

- Multiple calls to `open()` on one device node can share the same inode but have separate file objects.
- Per-device state belongs in the device structure.
- Per-open state can be attached to `file->private_data`.

## Open And Release Callbacks

```c
static int mydev_open(struct inode *inode, struct file *file)
{
    file->private_data = &mydev;
    pr_info("mydev: opened\n");
    return 0;
}

static int mydev_release(struct inode *inode, struct file *file)
{
    pr_info("mydev: closed\n");
    return 0;
}
```

- `.open` can reject access with a negative error code.
- `.release` normally performs cleanup and returns `0`.
- These callbacks should not be confused with module init and exit:
  - module init/exit happen when the driver module loads or unloads
  - open/release happen for userspace file access

## Read Callback

```c
static ssize_t mydev_read(
    struct file *file,
    char __user *buf,
    size_t count,
    loff_t *offset)
{
    size_t available;
    size_t length;

    if (*offset >= mydev.data_len)
        return 0;

    available = mydev.data_len - *offset;
    length = min(count, available);

    if (copy_to_user(buf, mydev.data + *offset, length))
        return -EFAULT;

    *offset += length;
    return length;
}
```

- `buf` is a userspace pointer marked with `__user`.
- `copy_to_user()` copies kernel data to userspace.
- Return values:
  - positive byte count for successful transfer
  - `0` for end-of-file
  - negative error code on failure
- Updating `*offset` gives regular file-like sequential-read behavior.

## Write Callback

```c
static ssize_t mydev_write(
    struct file *file,
    const char __user *buf,
    size_t count,
    loff_t *offset)
{
    size_t length = min(count, sizeof(mydev.data));

    if (copy_from_user(mydev.data, buf, length))
        return -EFAULT;

    mydev.data_len = length;
    return length;
}
```

- `copy_from_user()` copies userspace data into kernel memory.
- Never directly trust or dereference a userspace pointer.
- Real drivers must also consider:
  - synchronization
  - partial transfers
  - blocking and nonblocking behavior
  - device capacity
  - concurrent opens

## `.read` Versus `.read_iter`

- `struct file_operations` supports:
  - `.read` and `.write`
  - `.read_iter` and `.write_iter`
- The iterator variants use:
  - `struct kiocb`
  - `struct iov_iter`
- They support generalized vectored and potentially asynchronous I/O paths.
- A beginner character driver can start with `.read` and `.write`.
- A driver should normally select one read interface and one write interface rather than implementing both with inconsistent behavior.

## What `ioctl()` Is

```txt
read() / write() -> transfer data
ioctl()          -> control a device or query its state
```

- `ioctl()` handles operations that do not naturally fit byte-stream transfer:
  - reset a device
  - change a device mode
  - query capacity or status
  - configure UART or modem controls

```c
ioctl(fd, command, argument);
```

- The command identifies the operation.
- The optional argument may be:
  - an integer
  - a pointer to input data
  - a pointer to output data
  - a pointer used for both input and output

## Defining `ioctl` Commands

- Put shared command definitions in a UAPI header used by both driver and userspace code:

```c
#ifndef MYDEV_IOCTL_H
#define MYDEV_IOCTL_H

#include <linux/ioctl.h>

#define MYDEV_IOC_MAGIC 'M'

struct mydev_status {
    unsigned int value;
    unsigned int ready;
};

#define MYDEV_RESET      _IO(MYDEV_IOC_MAGIC, 0)
#define MYDEV_SET_VALUE  _IOW(MYDEV_IOC_MAGIC, 1, unsigned int)
#define MYDEV_GET_STATUS _IOR(MYDEV_IOC_MAGIC, 2, struct mydev_status)

#endif
```

| Macro | Direction from the userspace perspective |
|---|---|
| `_IO` | no data payload |
| `_IOW` | userspace writes data to the kernel |
| `_IOR` | userspace reads data from the kernel |
| `_IOWR` | data moves in both directions |

- The encoded command includes information such as:
  - command family or magic value
  - command number
  - transfer direction
  - payload type size
- Do not use unexplained raw integers as a public command interface.

## Implementing `.unlocked_ioctl`

```c
static long mydev_ioctl(
    struct file *file,
    unsigned int cmd,
    unsigned long arg)
{
    unsigned int value;
    struct mydev_status status;

    switch (cmd) {
    case MYDEV_RESET:
        mydev.value = 0;
        return 0;

    case MYDEV_SET_VALUE:
        if (copy_from_user(
                &value,
                (void __user *)arg,
                sizeof(value)))
            return -EFAULT;

        mydev.value = value;
        return 0;

    case MYDEV_GET_STATUS:
        status.value = mydev.value;
        status.ready = 1;

        if (copy_to_user(
                (void __user *)arg,
                &status,
                sizeof(status)))
            return -EFAULT;

        return 0;

    default:
        return -ENOTTY;
    }
}
```

- Unsupported commands should normally return `-ENOTTY`.
- Validate:
  - command identity
  - argument values
  - payload sizes
  - device state
- User pointers still require `copy_to_user()` or `copy_from_user()`.

## Userspace `ioctl` Example

```c
#include <fcntl.h>
#include <stdio.h>
#include <sys/ioctl.h>
#include <unistd.h>

#include "mydev_ioctl.h"

int main(void)
{
    struct mydev_status status;
    unsigned int value = 42;
    int fd;

    fd = open("/dev/mydev0", O_RDWR);
    if (fd < 0)
        return 1;

    if (ioctl(fd, MYDEV_SET_VALUE, &value) < 0)
        perror("MYDEV_SET_VALUE");

    if (ioctl(fd, MYDEV_GET_STATUS, &status) == 0)
        printf("value=%u ready=%u\n",
               status.value, status.ready);

    close(fd);
    return 0;
}
```

## Simplified Dispatch Paths

```txt
read(fd, ...)
    -> VFS
    -> file->f_op->read()
    -> mydev_read()
```

```txt
write(fd, ...)
    -> VFS
    -> file->f_op->write()
    -> mydev_write()
```

```txt
ioctl(fd, ...)
    -> VFS
    -> file->f_op->unlocked_ioctl()
    -> mydev_ioctl()
```

## Compatibility Callback

- `.compat_ioctl` supports compatibility system calls, commonly 32-bit userspace on a 64-bit kernel.
- It matters when payload layouts contain values whose size or alignment differs between ABIs.
- A stable UAPI should prefer fixed-width integer types and carefully defined layouts.
- A simple teaching driver may omit `.compat_ioctl`, but production ABI design must consider it.

## Why `ioctl()` Requires Care

- `ioctl()` is a flexible command entry point and can become an undocumented collection of unrelated operations.
- Keep the ABI maintainable:
  - use named command macros
  - document every payload
  - validate all input
  - preserve compatibility after release
  - avoid embedding userspace pointers inside structures unless the ABI explicitly handles them
- Other Linux interfaces may fit some control planes better:
  - sysfs
  - configfs
  - Netlink
  - dedicated system calls or subsystem APIs
- These do not universally replace `ioctl()`; the correct interface depends on the operation and subsystem.

## Error And Return Conventions

- Kernel callbacks return negative error codes directly:

```c
return -EINVAL;
```

- The system-call layer converts this into userspace `-1` and sets `errno`.
- Common errors:

| Kernel return | Meaning |
|---|---|
| `-EFAULT` | invalid userspace memory access |
| `-EINVAL` | invalid argument |
| `-ENODEV` | device unavailable |
| `-EBUSY` | resource busy |
| `-ENOTTY` | unsupported `ioctl` command |

## Concurrency And Lifetime

- Driver callbacks may run concurrently from multiple processes or threads.
- Shared state can require:
  - mutexes
  - spinlocks
  - atomic variables
  - wait queues
- The correct primitive depends on whether the path can sleep and whether it runs in process or interrupt context.
- `.owner = THIS_MODULE` protects module-code lifetime, but it does not synchronize device data.

## Key Lesson

- Drivers register callbacks; they do not replace global system-call implementations.
- The VFS reaches callbacks through `file->f_op`.
- `struct inode` identifies the underlying device node, while `struct file` describes one open instance.
- `read()` and `write()` transfer data.
- `ioctl()` handles well-defined control and query commands.
- Userspace pointers must be accessed with kernel user-copy helpers.
- A released `ioctl` interface is a userspace ABI and should be designed for long-term compatibility.

## References

- [Let's code a Linux Driver YouTube playlist by Johannes 4GNU_Linux](https://www.youtube.com/watch?v=DZrb9oSEzlU&list=PLCGpd0Do5-I3b5TtyqeF1UdyD4C-S-dMa&index=2)
- [`struct file_operations` in Linux `fs.h`](https://elixir.bootlin.com/linux/v7.0.11/source/include/linux/fs.h#L1926)
- [Linux VFS documentation](https://docs.kernel.org/filesystems/vfs.html)
- [Linux ioctl design documentation](https://docs.kernel.org/driver-api/ioctl.html)
- [Linux read and write VFS implementation](https://github.com/torvalds/linux/blob/v7.0/fs/read_write.c)

## Previous Reading

- [Linux Kernel Module Hello World](./linux-kernel-module-hello-world)
- [Linux Device Numbers, Nodes, Inodes, and TTY Modes](./linux-device-numbers-nodes-inodes-and-tty-modes)
