---
title: Linux Device Numbers, Nodes, Inodes, and TTY Modes
description: Notes about character and block device nodes, major and minor numbers, mknod, inodes, cdev registration, and TTY raw and canonical modes.
tags:
- linux
- kernel
- device-driver
- character-device
- inode
- tty
---

## Goal

- Understand how a pathname such as `/dev/mydev0` reaches a kernel driver.
- Learn the roles of:
  - device type
  - major number
  - minor number
  - device inode
  - `struct cdev`
  - `mknod`

## Device Access Path

```txt
userspace pathname
    -> directory entry
    -> device inode
    -> device type and dev_t
    -> major/minor lookup
    -> character or block driver
    -> device instance
```

- A device node is an access point to a driver.
- It does not contain the device's ordinary data.
- Creating another node with the same type, major, and minor numbers does not create another hardware device.

## Device Node Types

- `ls -l` shows a file-type character at the beginning of the mode string:

| First character | Type |
|---|---|
| `-` | regular file |
| `d` | directory |
| `c` | character device |
| `b` | block device |
| `p` | FIFO or named pipe |
| `s` | socket |
| `l` | symbolic link |

- Character devices provide stream- or byte-oriented interfaces.
- Block devices support block-addressed storage and the block I/O subsystem.

```txt
crw-rw---- 1 root dialout 188, 0 /dev/ttyUSB0
brw-rw---- 1 root disk      8, 0 /dev/sda
```

- `c` identifies a character device.
- `b` identifies a block device.
- `188, 0` and `8, 0` are major/minor pairs.

## Major And Minor Numbers

- The major number selects a driver or device class registered for that range.
- The minor number selects a device instance or subdevice interpreted by that driver.

```txt
major 188, minor 0 -> first USB serial device
major 188, minor 1 -> second USB serial device
```

- This is a conceptual rule; the exact interpretation of minor numbers belongs to each driver.
- The kernel stores the pair in `dev_t`.

## Working With `dev_t`

```c
dev_t dev;

ret = alloc_chrdev_region(&dev, 0, 1, "mydev");
if (ret)
    return ret;

pr_info("mydev: major=%u minor=%u\n",
        MAJOR(dev), MINOR(dev));
```

- `alloc_chrdev_region()` dynamically allocates a range of character-device numbers.
- Arguments:
  - output `dev_t`
  - first requested minor number
  - number of consecutive device numbers
  - registration name
- `MAJOR(dev)` extracts the major number.
- `MINOR(dev)` extracts the minor number.
- `MKDEV(major, minor)` constructs a `dev_t` when numbers are already known.

## Fixed Versus Dynamic Registration

- Dynamic registration:

```c
alloc_chrdev_region(&dev, first_minor, count, "mydev");
```

- Fixed registration:

```c
dev = MKDEV(requested_major, first_minor);
register_chrdev_region(dev, count, "mydev");
```

- Dynamic allocation avoids collisions and is generally preferred for new teaching or out-of-tree drivers.
- Cleanup must release the same range:

```c
unregister_chrdev_region(dev, count);
```

## Connecting A `cdev`

- A device number reserves an identity, but callbacks must also be connected to it.
- A character driver uses `struct cdev`:

```c
static struct cdev mydev_cdev;

cdev_init(&mydev_cdev, &mydev_fops);
mydev_cdev.owner = THIS_MODULE;

ret = cdev_add(&mydev_cdev, dev, 1);
```

- `cdev_init()` connects the character-device object to `file_operations`.
- `cdev_add()` makes the device live for the registered number range.
- Once `cdev_add()` succeeds, userspace may open the device at any time.
- Cleanup reverses registration:

```c
cdev_del(&mydev_cdev);
unregister_chrdev_region(dev, 1);
```

## Creating A Node With `mknod`

- Manual character-device node:

```sh
sudo mknod /dev/mydev0 c <major> 0
sudo chmod 666 /dev/mydev0
```

- Syntax:

```txt
mknod PATH TYPE MAJOR MINOR
```

- Common device types:
  - `c` or `u`: character device
  - `b`: block device
- A FIFO uses `p`, but it does not take major and minor numbers:

```sh
mkfifo /tmp/myfifo
```

- Inspect the node:

```sh
ls -l /dev/mydev0
stat /dev/mydev0
```

## Manual Nodes Versus `udev`

- `mknod` is useful for learning and debugging.
- Modern systems commonly create nodes automatically through:
  - the driver model
  - sysfs
  - `devtmpfs`
  - `udev`
- A driver can register a class and device so userspace creates the node:

```c
mydev_class = class_create("mydev");
device_create(mydev_class, NULL, dev, NULL, "mydev0");
```

- Cleanup occurs in reverse:

```c
device_destroy(mydev_class, dev);
class_destroy(mydev_class);
```

- Exact helper signatures can vary across kernel versions, so code should be checked against the target kernel headers.

## A Node Can Exist Without A Working Device

- `mknod` creates a filesystem object.
- It does not register or load a driver.
- If no driver owns the device number, opening the node fails even though `ls` shows it.

```txt
node exists
    + driver not registered
    -> open() fails
```

## Two Nodes With The Same Device Number

```sh
sudo mknod /tmp/mydev c <major> 0
```

- If `/dev/mydev0` and `/tmp/mydev` have the same:
  - node type
  - major number
  - minor number
- both paths normally reach the same registered device instance.

```txt
/dev/mydev0 --\
               -> same dev_t -> same cdev -> same driver instance
/tmp/mydev  ---/
```

- Per-open state can still differ because each `open()` creates a separate `struct file`.
- Shared hardware state normally remains shared.

## Inode Overview

- A directory maps a filename to an inode number.
- The inode represents the filesystem object:

```txt
filename
    -> directory entry
    -> inode number
    -> inode
```

- A regular-file inode commonly stores:
  - file type and permissions
  - owner and group
  - file size
  - timestamps
  - hard-link count
  - references to data blocks
- The filename itself is stored in the directory entry, not in the inode.

## Hard Links And File Lifetime

```sh
ln a.txt b.txt
ls -li a.txt b.txt
```

- Both names can point to the same inode.
- Removing one name decrements the inode's link count.
- File storage is reclaimed only when:
  - the hard-link count reaches zero
  - no open file description still references it
- This explains why deleting an open log file may not immediately release disk space.

## Device Inodes

- A device inode stores the special-file type and device identifier instead of ordinary file data blocks:

```txt
file type = character device
dev_t = major/minor
```

- Opening a device node follows:

```txt
open("/dev/mydev0")
    -> directory entry
    -> device inode
    -> inode's dev_t
    -> character-device lookup
    -> cdev and file_operations
```

- The VFS passes both an inode and a newly created file object to the driver's `.open` callback.
- The inode identifies the opened device.
- The file object represents this particular open instance.

## TTY Character Devices And Line Discipline

- Serial ports such as these are character devices:

```txt
/dev/ttyS0
/dev/ttyUSB0
/dev/ttyACM0
```

- A TTY line discipline can sit between userspace and the underlying character driver:

```txt
userspace
    -> TTY line discipline
    -> character driver
    -> UART or USB serial device
```

- It can:
  - buffer input
  - echo characters
  - process editing keys
  - translate control characters
- This explains why TTY reads can behave differently from a minimal custom character device.

## Canonical Mode

- Canonical mode is line-oriented.
- Input is normally held until Enter or another configured line delimiter is received.
- Editing controls such as Backspace can be processed before `read()` returns.

```txt
user types: h e l l o Enter
read() returns: "hello\n"
```

## Raw And Noncanonical Modes

- Noncanonical mode disables line-based input collection.
- Raw mode also disables most echo, signal, input, and output processing.
- These modes are useful for:
  - serial protocols
  - terminal emulators
  - full-screen terminal applications
  - programs that need individual key presses

```c
struct termios options;

tcgetattr(fd, &options);
cfmakeraw(&options);
cfsetispeed(&options, B115200);
cfsetospeed(&options, B115200);
tcsetattr(fd, TCSANOW, &options);
```

- `termios` functions commonly use TTY-specific `ioctl()` operations internally.
- Shell-side serial and Minicom usage is recorded separately in:
  - [Linux Terminal Screen, Shell History, and Minicom Notes](./linux-terminal-screen-shell-history-and-minicom-notes)

## Cleanup Order

- A simple character driver should release resources in reverse registration order:

```txt
device_destroy()
    -> class_destroy()
    -> cdev_del()
    -> unregister_chrdev_region()
```

- Only release resources that were successfully acquired.
- Failure paths during initialization must unwind earlier completed steps.

## Key Lesson

- Major/minor numbers identify the driver-facing device.
- A device node is a filesystem name and inode containing that identity.
- `struct cdev` connects the device number to `file_operations`.
- `mknod` creates a node but does not create or register the driver.
- TTY raw/canonical behavior is an additional processing layer over character-device I/O.

## References

- [Linux kernel character-device API](https://docs.kernel.org/core-api/kernel-api.html)
- [Linux VFS overview](https://docs.kernel.org/filesystems/vfs.html)
- [Kernel example of major/minor numbers and `mknod`](https://docs.kernel.org/powerpc/hvcs.html)

## Previous Reading

- [Linux Kernel Module Hello World](./linux-kernel-module-hello-world)
