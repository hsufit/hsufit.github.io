---
title: Linux zram and zswap Notes
description: Notes about zram, how it works as compressed RAM swap, and how it differs from zswap.
tags:
- linux
- memory
- zram
- zswap
- swap
---

## Question

- I wanted to understand what `zram` is.
- I also wanted to know whether zram is like page swap.
- My first mental model was:
  - compress before swap out
  - keep it in RAM first
  - write it to disk only if it is really unused
- That mental model is close to `zswap`.
- Pure `zram` works differently.

## What Is zram?

- `zram` is a Linux kernel feature.
- It creates a compressed RAM block device.
- The device usually appears as:

```txt
/dev/zram0
```

- Linux can use this device as swap.
- The important idea:
  - normal swap writes pages to SSD or HDD
  - zram compresses pages and keeps them in RAM
- The Linux kernel zram documentation is linked in the References section.

## Simple Comparison

- Traditional swap:
  - RAM is not enough.
  - Pages are written to SSD or HDD.
  - It is slower.

- zram:
  - RAM is not enough.
  - Pages are compressed.
  - Compressed pages stay in RAM.
  - It is usually much faster than disk swap.

## zram Mental Model

- zram is like a compressed swap device inside RAM.
- The kernel treats it as a swap device.
- But the backing storage is compressed memory, not disk.

```txt
memory pressure
    |
    v
kernel decides to swap out a page
    |
    v
page is written to /dev/zram0
    |
    v
zram driver compresses the page
    |
    v
compressed object stays in RAM
```

## Important Point

- zram does not later write those pages to disk by itself.
- zram itself is the swap device.
- The flow ends after the page enters zram.

```txt
anonymous page
    |
    v
swap out
    |
    v
compressed in zram
    |
    v
still in RAM
```

- There is no built-in:

```txt
zram -> disk swap
```

## Why zram Saves Memory

- A normal Linux page is often 4 KB.
- Some pages compress very well.
- Example:
  - original page: 4 KB
  - compressed page: 300 bytes
- zram can free the original 4 KB page frame.
- It stores a much smaller compressed object instead.
- The tradeoff:
  - spend CPU
  - gain effective memory capacity

## Example

- Suppose the system has:
  - 8 GB RAM
  - 4 GB zram swap
- If the compression ratio is around 2:1:
  - zram may physically use about 2 GB RAM
  - but it can hold about 4 GB of swapped data
- The system may feel like it has more usable memory.
- The exact result depends on workload.

## Why zram Is Fast

- Compression in RAM is usually much faster than disk swap.
- Algorithms like `lz4` are very fast.
- Even with compression and decompression cost, zram is often much faster than SSD or HDD swap.

## Advantages

- zram reduces disk I/O.
- zram reduces SSD write wear.
- zram improves low-memory behavior.
- zram can make small RAM systems feel smoother.
- It is useful for:
  - Raspberry Pi
  - old laptops
  - small VMs
  - Docker hosts
  - Kubernetes nodes
  - Chromebooks
  - Android devices

## Disadvantages

- zram uses CPU for compression and decompression.
- zram does not create real RAM.
- zram only improves memory efficiency.
- If the working set is much larger than physical RAM, the system can still become slow.
- Heavy workloads may still need real RAM or disk-backed swap.

## What Happens on Page Fault?

- If a program accesses a page that was swapped into zram:

```txt
access swapped page
    |
    v
page fault
    |
    v
read compressed object from zram
    |
    v
decompress page
    |
    v
restore normal page
```

- This adds CPU work and some latency.
- But it is usually still much better than reading from disk swap.

## Linux VM Subsystem View

- The Linux VM subsystem sees swap as an abstraction.
- It does not need to care whether the swap device is:
  - disk
  - SSD
  - zram
- The VM subsystem only needs:
  - a page was evicted
  - the page has a swap slot
- The zram driver handles:
  - receiving writes
  - compressing pages
  - storing metadata
  - managing compressed memory

## What Is Stored in zram?

- zram stores compressed swapped pages.
- A rough flow:

```txt
struct page
    |
    v
compress
    |
    v
zsmalloc allocator
    |
    v
compressed object
```

- Common compression algorithms include:
  - `lz4`
  - `lzo`
  - `zstd`

## Why Compression Works

- Many memory pages contain compressible data.
- Examples:
  - many zeros
  - repeated strings
  - sparse data
  - idle application memory
  - cache-like data
- Compression ratios may be around:
  - 1.5x
  - 2x
  - 3x
- It depends heavily on workload.

## zram vs zswap

- zram and zswap are easy to confuse.
- They both compress memory pages.
- But their roles are different.
- The Linux kernel zswap documentation in the References section is the source for the compressed swap-cache model.

| Feature | zram | zswap |
|---|---|---|
| Basic idea | compressed swap device in RAM | compressed cache before disk swap |
| Disk backing | no built-in disk backing | yes, disk swap can be backing store |
| Where pages go | compressed RAM device | compressed RAM cache first |
| If full | swap pressure stays in zram limits | can write back to disk swap |
| Mental model | compressed RAM disk | compressed write-back cache |

## zswap Mental Model

- zswap is closer to this idea:
  - before writing to disk swap, compress the page
  - keep it in RAM first
  - if the compressed cache is full, write back to disk

```txt
memory pressure
    |
    v
page prepared for swap out
    |
    v
compressed into zswap cache
    |
    v
if zswap is full
    |
    v
write to disk swap
```

- This is different from pure zram.

## zram Memory Layout

```txt
RAM
|-- normal pages
`-- compressed swapped pages in zram
```

- Everything is still in RAM.
- Some pages are just compressed.

## zswap Memory Layout

```txt
RAM
|-- normal pages
|-- compressed zswap cache
`-- disk swap backing store exists outside RAM
```

- zswap may eventually write pages to disk swap.

## How to Check zram on Linux

- Check zram devices:

```bash
ls /dev/zram*
```

- Check active swap:

```bash
swapon --show
```

- Check zram status:

```bash
zramctl
```

## Enable zram with systemd

- Some Linux distributions support zram through `systemd-zram-generator`.
- The `systemd-zram-generator` project link is in the References section.
- Example install command on Debian or Ubuntu style systems:

```bash
sudo apt install systemd-zram-generator
```

- Example config:

```ini
# /etc/systemd/zram-generator.conf

[zram0]
zram-size = ram / 2
compression-algorithm = lz4
```

- Reload and reboot:

```bash
sudo systemctl daemon-reload
sudo reboot
```

## Android Uses zram Heavily

- Modern Android phones commonly use zram.
- Android is a good fit because:
  - RAM is limited
  - flash write should be reduced
  - many apps stay in the background
  - app switching benefits from keeping cold pages compressed
- zram helps Android multitasking feel smoother.

## Good Use Cases

- zram is useful for:
  - 4 GB to 16 GB RAM Linux desktops
  - laptops
  - Raspberry Pi
  - small VMs
  - Docker hosts
  - Kubernetes nodes
  - NAS boxes
  - embedded Linux
  - Android devices

- Common starting settings:
  - zram size: 25% to 100% of RAM
  - compression algorithm: `lz4`

## When zram May Not Help Much

- zram may not help much if:
  - the system has 64 GB or more RAM
  - the workload almost never swaps
  - the workload is extremely CPU-sensitive
  - the working set is far larger than RAM
- In these cases, real RAM or workload tuning may matter more.

## One-Sentence Summary

- zram is a compressed swap device that lives in RAM, while zswap is a compressed cache in front of disk swap.

## Mental Shortcut

- zram:
  - compressed RAM swap device
  - no automatic writeback to disk

- zswap:
  - compressed swap cache
  - can eventually write back to disk swap

## References

- Linux kernel zram documentation:
  - https://www.kernel.org/doc/html/latest/admin-guide/blockdev/zram.html
- Linux kernel zswap documentation:
  - https://www.kernel.org/doc/html/latest/admin-guide/mm/zswap.html
- systemd zram generator:
  - https://github.com/systemd/zram-generator
