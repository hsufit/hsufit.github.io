---
title: Linux Userspace, Kernel Space, Physical Memory, and mmap
description: Notes about copy_to_user, copy_from_user, ioremap, mmap, and how userspace, kernel virtual addresses, physical memory, and device memory relate.
tags:
- linux
- device-driver
- mmap
- kernel
- memory
---

## Goal

- Understand how a Linux driver moves or maps data across:
  - userspace virtual memory
  - kernel virtual memory
  - physical RAM
  - device MMIO address ranges
- Connect these memory-related APIs to the earlier file-operation callbacks:
  - `copy_to_user()`
  - `copy_from_user()`
  - `ioremap()`
  - `iounmap()`
  - `mmap()`
- Explain:
  - what `mmap()` means in a userspace program
  - whether `mmap()` runs in userspace or kernel space
  - when copying is more appropriate than sharing a mapping

## Address-Space Big Picture

- A pointer is interpreted in an address space.
- A userspace pointer normally contains a process virtual address.
- A normal kernel pointer contains a kernel virtual address.
- A physical address identifies RAM or a device bus address from the hardware's perspective.
- Virtual addresses are translated through page tables before reaching physical memory.

```txt
Process virtual address
    -> process page tables
    -> physical RAM page

Kernel virtual address
    -> kernel mapping
    -> physical RAM page or mapped device resource
```

- User and kernel virtual addresses can refer to the same physical page through different mappings.
- A driver must not treat userspace, kernel virtual, and physical addresses as interchangeable numbers.

## API Relationship

```txt
User space
    |
    | copy_to_user() / copy_from_user()
    v
Kernel space
    |
    | ioremap() / iounmap()
    v
Hardware MMIO registers / physical memory
```

- `copy_to_user()` and `copy_from_user()` move data between user and kernel.
- `ioremap()` maps hardware physical addresses into kernel virtual addresses.
- `mmap()` maps files, device memory, or buffers into a process virtual address space.

| Operation | Source | Destination | Main Result |
|---|---|---|---|
| `copy_to_user()` | kernel buffer | userspace buffer | bytes are copied |
| `copy_from_user()` | userspace buffer | kernel buffer | bytes are copied |
| `ioremap()` | MMIO physical range | kernel virtual range | kernel mapping is created |
| driver `.mmap` | driver-selected pages or PFNs | process virtual range | userspace mapping is created |

## copy_to_user and copy_from_user

- These APIs safely copy data between kernel space and userspace.
- Direction:
  - `copy_to_user()` means kernel to user.
  - `copy_from_user()` means user to kernel.
- The Linux kernel hacking reference in the References section documents the important return-value and sleep caveats.

```c
if (copy_to_user(user_buf, kernel_buf, len))
    return -EFAULT;

if (copy_from_user(kernel_buf, user_buf, len))
    return -EFAULT;
```

- The kernel should not directly trust a user pointer.
- A user pointer may:
  - point to an invalid address
  - trigger a page fault
  - be controlled by a malicious program
- These APIs are common in:
  - `read()`
  - `write()`
  - `ioctl()`
- Their return value is the number of bytes that could not be copied.
- Therefore, `0` means the complete copy succeeded.
- These helpers may fault and sleep, so they must only be used from a context where sleeping is permitted.

## ioremap

- `ioremap()` maps a hardware physical address into kernel virtual address space.
- It is commonly used for MMIO.
- MMIO means Memory-Mapped I/O.
- The Linux device I/O reference in the References section is the main link for `ioremap()`, `iounmap()`, `readl()`, and `writel()`.
- Examples:
  - PCI device registers
  - SoC peripheral registers
  - device control/status registers

```c
void __iomem *reg_base;

reg_base = ioremap(0x10000000, 0x1000);
```

- After mapping, the driver can access registers through special MMIO helpers:

```c
value = readl(reg_base + offset);
writel(value, reg_base + offset);
```

- The important mapping is:

```txt
hardware physical address -> kernel virtual address
```

- The returned pointer should usually be marked as `__iomem`.
- Drivers should use MMIO accessors like `readl()` and `writel()`, not normal pointer dereference.
- `ioremap()` is intended for I/O resources; it is not a general replacement for mapping ordinary RAM.

## iounmap

- `iounmap()` releases a mapping created by `ioremap()`.

```c
iounmap(reg_base);
```

- Simple memory:
  - `ioremap()` creates the mapping.
  - `iounmap()` destroys the mapping.
- A driver should unmap MMIO regions during cleanup or error handling.

## Driver mmap

- A device driver can implement a `.mmap` file operation.
- This lets a userspace process map device memory or driver-managed memory.
- Common use cases:
  - GPU buffers
  - camera frame buffers
  - DMA buffers
  - high-performance I/O
  - zero-copy style data sharing

```c
static int my_mmap(struct file *file, struct vm_area_struct *vma)
{
    unsigned long requested_size = vma->vm_end - vma->vm_start;

    if (requested_size > size)
        return -EINVAL;

    return remap_pfn_range(vma,
                           vma->vm_start,
                           phys_addr >> PAGE_SHIFT,
                           requested_size,
                           vma->vm_page_prot);
}
```

- Driver-side `mmap` is not the same as copying data.
- It creates a mapping into the process address space.
- Userspace can then access the mapped region with normal memory loads and stores.
- The kernel memory-management API reference below is the place to check `remap_pfn_range()`.
- The driver must validate:
  - requested length
  - offset
  - page alignment
  - access permissions
  - whether the physical range is safe to expose
- `remap_pfn_range()` maps page frame numbers into the userspace VMA.
- It must not be used blindly for arbitrary kernel virtual memory.

## Userspace mmap

- In a userspace program, `mmap()` means memory map.
- It maps a file, device, or anonymous memory into the process virtual address space.
- The program receives a pointer and can access the mapped region like memory.
- The `mmap(2)` manual page in the References section is the userspace API reference.

```c
void *addr = mmap(NULL, length,
                  PROT_READ | PROT_WRITE,
                  MAP_SHARED,
                  fd, 0);
```

- For a file mapping:
  - the file is represented as a memory range
  - accessing the memory reads file contents
  - writing the memory may update the file, depending on mapping flags
- For a device mapping:
  - the file descriptor may be `/dev/mydev`
  - the driver's `.mmap` handler decides what memory is mapped

## mmap vs read and write

- Traditional file I/O:

```c
read(fd, buf, size);
write(fd, buf, size);
```

- `mmap()` style:

```c
char *p = mmap(...);
printf("%c", p[10]);
```

- Comparison:

| Method | Style | Performance Pattern | Common Use |
|---|---|---|---|
| `read()` / `write()` | explicit I/O copy | simple but may copy more | normal I/O |
| `mmap()` | memory access | good for large or random access | files, buffers, devices |
| `copy_to_user()` | kernel-to-user copy | good for small data | driver `read()` or `ioctl()` |

## Copying Versus Mapping

- Copy-based path:

```txt
device or kernel buffer
    -> driver read()
    -> copy_to_user()
    -> userspace buffer
```

- Mapping-based path:

```txt
driver-managed pages or device range
    -> driver .mmap()
    -> process virtual address
    -> userspace loads and stores
```

- Copying is often preferable when:
  - transfers are small
  - the interface is naturally message- or stream-oriented
  - the driver must validate or transform each transfer
  - simple ownership and lifetime rules matter most
- Mapping is often preferable when:
  - buffers are large
  - access is frequent or random
  - repeated copies are expensive
  - the memory can be exposed safely for the mapping lifetime

## Userspace mmap Use Cases

- File processing:
  - large file reading
  - random access
  - database storage engines
- Memory management:
  - anonymous mappings
  - large memory allocation
  - some `malloc()` implementations may use `mmap()` for large allocations
- IPC:
  - shared memory between processes
  - shared file-backed mappings
- Device access:
  - frame buffers
  - DMA buffers
  - device memory exposed through a driver `.mmap`

## mmap Is Lazy

- `mmap()` usually does not load all content immediately.
- The kernel creates a virtual memory area first.
- Real data is often loaded on demand.
- This is called lazy loading.
- The key event is a page fault.

## Is mmap Userspace or Kernel Space?

- The `mmap()` function call starts in userspace.
- The real mapping work is done in kernel space.
- A good summary:

| Part | Where It Happens |
|---|---|
| calling `mmap()` | userspace |
| entering the system call | kernel entry |
| checking permissions | kernel space |
| creating the VMA | kernel space |
| returning a pointer | back to userspace |
| later memory access | userspace instruction |
| page fault handling | kernel space |
| page table update | kernel space / MMU support |

## mmap Flow

- Step 1:
  - userspace calls `mmap()`

```c
void *p = mmap(...);
```

- Step 2:
  - libc enters the `mmap` system call
  - control moves from userspace to kernel space
- Step 3:
  - the kernel checks permissions
  - the kernel creates a `vm_area_struct`
  - the kernel records the virtual memory range
- Step 4:
  - the kernel returns a userspace virtual address
- Step 5:
  - userspace accesses the mapped pointer

```c
printf("%c", ((char *)p)[0]);
```

- Step 6:
  - if the page is not present, the CPU raises a page fault
  - the kernel handles the fault
  - the kernel loads or allocates the page
  - the kernel updates the page table
- Step 7:
  - execution returns to userspace
  - the same memory access can now continue

## mmap Flow Diagram

```txt
Userspace:
    mmap()

        |
        | syscall
        v

Kernel:
    create VMA
    register mapping
    return virtual address

        |
        v

Userspace:
    access p[]

        |
        | page fault if page is missing
        v

Kernel:
    find VMA
    load file page or allocate memory
    update page table

        |
        v

Userspace:
    continue normal memory access
```

## mmap in One Sentence

- `mmap()` does not simply read a file.
- It creates a relationship between:

```txt
process virtual memory <-> file / device / anonymous memory
```

- The kernel manages that relationship.
- The CPU and MMU make later memory access fast.

## mmap and Device Drivers

- From userspace:
  - the program calls `mmap()` on a device file descriptor
- From kernel space:
  - the driver implements `.mmap`
  - the driver maps the right memory into the process
- After the mapping:
  - userspace can access the memory directly
  - the driver avoids repeated `copy_to_user()` calls

```txt
Userspace process
    |
    | mmap("/dev/mydev")
    v
Driver .mmap()
    |
    | remap_pfn_range() or fault-based mapping
    v
Mapped device / buffer pages
```

## mmap vs copy_to_user

| API | Main Idea | Best Fit |
|---|---|---|
| `copy_to_user()` | copy data from kernel to user | small or simple transfers |
| `copy_from_user()` | copy data from user to kernel | commands, writes, ioctl input |
| `mmap()` | share a memory mapping | large buffers or high-performance access |

- `copy_to_user()` moves bytes each time.
- `mmap()` sets up a mapping once.
- For large buffers, `mmap()` can avoid repeated copies.

## Important Notes

- `munmap()` removes a userspace mapping.
- `msync()` can force changes back to a mapped file.
- Memory mappings are page-based.
- Wrong pointer access can still cause a segmentation fault.
- Driver `mmap` code must be careful about:
  - permissions
  - page alignment
  - cache attributes
  - lifetime of mapped memory
  - security of exposing device memory to userspace
- A mapping is not automatically safe or truly zero-cost:
  - page faults and page-table setup still have costs
  - synchronization may still be required
  - the driver must not free mapped backing memory while userspace can access it

## References

- Linux kernel device I/O documentation:
  - https://www.kernel.org/doc/html/latest/driver-api/device-io.html
- Linux kernel memory-management API documentation:
  - https://www.kernel.org/doc/html/latest/core-api/mm-api.html
- Linux `copy_to_user()` / `copy_from_user()` notes:
  - https://www.kernel.org/doc/html/latest/kernel-hacking/hacking.html#copy-to-user-copy-from-user-get-user-put-user
- Linux `mmap(2)` manual page:
  - https://man7.org/linux/man-pages/man2/mmap.2.html

## One-Line Memory Aids

- `copy_to_user()`:
  - safe data copy from kernel to user
- `copy_from_user()`:
  - safe data copy from user to kernel
- `ioremap()`:
  - map hardware physical address to kernel virtual address
- `iounmap()`:
  - remove an `ioremap()` mapping
- `mmap()`:
  - map file, device, or memory into a process virtual address space

## Final Summary

- `copy_to_user()` and `copy_from_user()` are safe copy tools.
- `ioremap()` and `iounmap()` are MMIO mapping tools for drivers.
- Userspace `mmap()` starts as a function call but becomes a kernel system call.
- The kernel creates the mapping and handles page faults.
- Userspace later accesses the mapped region like normal memory.
- In device drivers, `mmap()` is useful when repeated copying is too expensive.
- User virtual, kernel virtual, and physical addresses belong to different address spaces and cannot be used interchangeably.

## Previous Reading

- [Linux Kernel Module Hello World](./linux-kernel-module-hello-world)
- [Linux Device Numbers, Nodes, Inodes, and TTY Modes](./linux-device-numbers-nodes-inodes-and-tty-modes)
- [Linux File Operations and ioctl](./linux-file-operations-and-ioctl)
