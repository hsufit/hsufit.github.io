---
title: RISC-V Page Table PTE Bits A D U G V Notes
description: Notes about RISC-V page table entry bits, A/D update behavior, permission faults, and related Sv32 virtual memory details.
tags:
- riscv
- virtual-memory
- page-table
- operating-system
- cpu
---

## Question

- I was reading the RISC-V page table specification.
- I did not fully understand the `A`, `D`, `U`, `G`, and `V` bits.
- I wanted to summarize:
  - what these bits mean
  - what behavior the spec defines
  - what OS uses them for
  - what related topics I did not ask about but should notice
- The official RISC-V privileged architecture links are collected in the References section.

## Sv32 PTE Shape

- An Sv32 PTE roughly contains:

```txt
| PPN | RSW | D | A | G | U | X | W | R | V |
```

- The bits discussed here:

| Bit | Name | Meaning |
|---|---|---|
| `V` | Valid | whether this PTE is valid |
| `U` | User | whether user mode can access this page |
| `G` | Global | whether this mapping is shared across address spaces |
| `A` | Accessed | whether the page has been read, written, or fetched |
| `D` | Dirty | whether the page has been written |

## V Bit

- `V` means valid.
- `V=1` means the PTE is a valid mapping.
- `V=0` means the PTE is invalid.
- If a page table walk sees an invalid PTE, it raises a page fault.

## Common OS Uses for V=0

- `V=0` may mean:
  - page is not allocated
  - page is swapped out
  - lazy allocation has not happened yet
  - guard page
  - unmapped memory

- When `V=0`, the mapping is not valid.
- OS software may sometimes use ignored bits for metadata, depending on the architecture rules.

## U Bit

- `U` means user accessible.
- `U=1` means U-mode software may access the page.
- `U=0` means user mode may not access the page.

## U Bit and SUM

- Supervisor mode normally does not access user pages.
- The `SUM` bit in `sstatus` changes this behavior.
- If `sstatus.SUM=1`, supervisor mode can access pages with `U=1`.
- This is useful for kernel routines such as copying data from user memory.
- Example concept:
  - `copy_from_user()`
  - temporarily allow kernel access to user pages

## U Bit Security Detail

- Even if `SUM=1`, supervisor mode cannot execute code from pages with `U=1`.
- This prevents the kernel from accidentally executing user memory.
- This is a security feature.

## G Bit

- `G` means global mapping.
- `G=1` means the mapping exists in all address spaces.
- Typical global mappings:
  - kernel text
  - kernel direct map
  - trampoline code

## Why G Bit Exists

- The `G` bit is a TLB optimization.
- On context switch, non-global TLB entries may need to be flushed or separated by ASID.
- Global mappings do not need to be duplicated for every ASID.
- They also do not need to be flushed by some `SFENCE.VMA` cases.

## G Bit Pitfall

- Not marking a global mapping as global only hurts performance.
- Marking a non-global mapping as global is a software bug.
- If a non-global page is incorrectly marked global, different address spaces may use the wrong mapping.
- That can cause unpredictable behavior.

## A Bit

- `A` means accessed.
- `A=1` means the virtual page has been:
  - read
  - written
  - instruction-fetched
- since the last time software cleared the `A` bit.

## OS Use of A Bit

- The OS can clear `A`.
- Later, the OS checks whether hardware or traps set it again.
- This helps approximate:
  - LRU behavior
  - page replacement
  - working set estimation

## D Bit

- `D` means dirty.
- `D=1` means the virtual page has been written.
- It tells the OS whether memory differs from backing storage.

## OS Use of D Bit

- During page eviction:
  - if `D=0`, the page may not need writeback
  - if `D=1`, the page may need to be written back to disk or swap
- `D` affects correctness.
- Losing dirty information can lose data.

## A/D Bit Management Modes

- The spec defines two schemes for managing `A` and `D`.
- These schemes define the contract between hardware page walkers and OS virtual memory.
- The Svade/Svadu references at the end are useful for checking current naming and behavior.

## Scheme 1: Svade Extension

- With the Svade extension:
  - access with `A=0` raises a page fault
  - write with `D=0` raises a page fault
- The OS handles the fault.
- The OS updates the PTE bits itself.

## Scheme 2: Hardware Auto-Update

- If Svade is not implemented, hardware updates the bits.
- On access:
  - if `A=0`, hardware sets `A=1`
- On write:
  - if `D=0`, hardware sets `D=1`
- This is similar in spirit to how many systems use hardware-maintained accessed/dirty bits.

## Why the Spec Spends So Much Text on A/D

- PTEs are shared memory data structures.
- Multiple harts may access or update page tables.
- The spec must define:
  - atomicity
  - ordering
  - speculation
  - two-stage translation behavior

## A Bit Can Be Speculative

- The spec allows `A` bit updates to happen speculatively.
- This means `A` may be set even if the memory access is not finally performed architecturally.
- This helps implementations such as:
  - speculative page table walks
  - address translation prefetchers
  - speculative TLB fills

## Why Speculative A Is OK

- The OS usually treats `A` as a hint.
- It is useful for page replacement.
- It does not need to be perfectly exact for functional correctness.

## D Bit Must Be Exact

- `D` bit updates must be exact.
- `D` cannot be speculative for explicit stores.
- `D` updates must be observed in program order by the local hart.
- This is because `D` affects correctness during page eviction and writeback.

## Atomic PTE Update

- Hardware PTE updates must be atomic with respect to other accesses to the PTE.
- This prevents races with OS page table updates.
- Example risk:
  - hardware is setting `A`
  - OS reuses or recycles the PTE
  - a non-atomic update could set bits on the wrong mapping

## PTE Update Ordering

- A PTE update must become globally visible before the memory access that caused it.
- The hart must not perform the memory access before the PTE update is globally visible.
- A trap may happen after the PTE update but before the actual memory access.
- Therefore:
  - `A` or `D` may be updated
  - but the memory access may not have completed

## A/D Bits Are Not Cleared by Hardware

- Hardware may set `A` and `D`.
- Hardware never clears them.
- Supervisor software clears them when it wants to track page activity again.
- If the OS does not care about `A` or `D`, it should set them to `1` to improve performance.

## Non-Leaf PTE Bits

- For non-leaf PTEs, `D`, `A`, and `U` are reserved for future standard use.
- Software must clear them for forward compatibility.
- Leaf PTEs use these bits for page permissions and status.

## Permission Fault Types

- The spec also defines which page-fault type is raised.

| Access | Missing permission | Fault |
|---|---|---|
| instruction fetch | `X=0` | instruction page fault |
| load | `R=0` | load page fault |
| store | `W=0` | store page fault |

## AMO Special Case

- AMO means Atomic Memory Operation.
- AMOs never raise load page-fault exceptions.
- AMOs include a write side.
- If an AMO targets an unreadable page, the page is also unwritable.
- Therefore the fault is a store page fault.

## Megapages

- Sv32 supports normal 4 KiB pages.
- Sv32 also supports 4 MiB megapages.
- Any level of PTE may be a leaf PTE.
- A megapage must be aligned to a 4 MiB boundary.
- If the physical address is not aligned correctly, a page fault is raised.

## Misaligned Access Partial Success

- Misaligned loads, stores, and instruction fetches may be decomposed into multiple accesses.
- Some parts may succeed before another part raises a page fault.
- A misaligned store may partially modify memory before another part faults.
- This is important because page faults do not always imply full rollback of a misaligned store.

## Two-Stage Translation

- Two-stage translation is used for virtualization.
- The flow is roughly:

```txt
guest virtual address
    |
    v
guest physical address
    |
    v
host physical address
```

- The stages are often called:
  - VS-stage
  - G-stage
- A single explicit memory access may cause A/D updates in both stages.

## RSW Field

- `RSW` is reserved for supervisor software.
- Hardware ignores this field.
- The OS can use it for its own metadata.

## Svade vs Svadu Naming Note

- In the quoted spec text, `Svade` is the extension name used for the scheme where clear `A` or `D` causes a page fault.
- There is also RISC-V virtual memory discussion around `Svadu`, which is related to hardware updating A/D bits.
- The important practical distinction:
  - Svade style: trap when `A` or `D` is not set
  - hardware-update style: hardware sets `A` or `D` automatically
- The official Svadu page in the References section is the link to revisit when checking this naming.

## Compact Bit Summary

| Bit | Meaning | Main purpose |
|---|---|---|
| `V` | valid mapping | mapping exists |
| `U` | user accessible | protection and security |
| `G` | global mapping | TLB optimization |
| `A` | accessed | page replacement hint |
| `D` | dirty | writeback correctness |

## Most Important Rules

- `V=0` means the PTE is invalid.
- `U=1` allows user-mode access.
- `G=1` means the mapping is global across address spaces.
- `A=1` means the page was accessed.
- `D=1` means the page was written.
- `A` may be speculative.
- `D` must be exact.
- PTE updates must be atomic.
- Hardware never clears `A` or `D`.
- Non-leaf PTEs must clear `D`, `A`, and `U`.

## References

- RISC-V privileged architecture manual, HTML snapshot:
  - https://riscv.github.io/riscv-isa-manual/snapshot/privileged
- RISC-V privileged architecture manual, PDF:
  - https://docs.riscv.org/reference/isa/_attachments/riscv-privileged.pdf
- RISC-V Svadu extension page:
  - https://docs.riscv.org/reference/isa/v20240411/priv/svadu.html

## One-Sentence Summary

- The `A`, `D`, `U`, `G`, and `V` bits define the contract between RISC-V hardware page walking and the OS virtual memory system: validity, protection, TLB sharing, replacement hints, and dirty-page correctness.
