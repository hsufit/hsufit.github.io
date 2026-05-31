---
title: RISC-V ASID, VMID, and Two-Stage Translation Notes
description: Notes about ASID, VMID, RISC-V satp/vsatp/hgatp, and how guest physical addresses are translated to machine physical addresses.
tags:
- riscv
- virtualization
- mmu
- tlb
- operating-system
---

## Question

- I wanted to understand ASID and VMID.
- I also wanted to know how they map to RISC-V registers.
- The confusing part was:
  - Does `satp` record ASID?
  - Does `vsatp` record VMID?
  - Why does `hgatp` translate directly to a machine physical address?
  - Where is the host OS in this translation flow?

## What Problem ASID and VMID Solve

- ASID and VMID are identifiers used by address translation hardware.
- Their main purpose is to avoid flushing the TLB too often.
- They tag TLB entries so translations from different contexts do not conflict.
- In short:
  - ASID separates process address spaces.
  - VMID separates virtual machines.

## TLB Background

- TLB means Translation Lookaside Buffer.
- It caches virtual-to-physical address translations.
- Page table walks are expensive.
- TLB hits make virtual memory fast.
- Without context tags, old TLB entries can become ambiguous after a context switch.

## Problem Without ASID

- Process A may use virtual address `0x1000`.
- Process B may also use virtual address `0x1000`.
- They map to different physical pages.

```txt
Process A: VA 0x1000 -> PA 0xAAA000
Process B: VA 0x1000 -> PA 0xBBB000
```

- Without ASID, the TLB cannot tell which process owns the cached translation.
- On context switch, the OS may need to flush the TLB.
- TLB flush is expensive.
- ASID lets the CPU keep translations from multiple processes at the same time.

## ASID

- ASID means Address Space Identifier.
- It identifies a process address space.
- A TLB entry can be tagged with ASID.

```txt
ASID | Virtual Address | Physical Address
-----+-----------------+-----------------
1    | 0x1000          | 0xAAA000
2    | 0x1000          | 0xBBB000
```

- With ASID, two processes can use the same virtual address without TLB conflict.
- Context switches do not always need a full TLB flush.
- One-line meaning:
  - Which process address space is this translation for?

## VMID

- VMID means Virtual Machine Identifier.
- It identifies a virtual machine.
- It is used by virtualization hardware and hypervisors.

## Problem Without VMID

- VM1 may use guest virtual address `0x1000`.
- VM2 may also use guest virtual address `0x1000`.
- They must not share the same translation context.
- If every VM switch flushed the TLB, virtualization would be slow.
- VMID lets the CPU keep translations from multiple VMs at the same time.

## VMID in TLB

- A TLB entry can be tagged with VMID.

```txt
VMID | Guest VA | Host/Machine PA
-----+----------+----------------
3    | 0x1000   | 0xAAA000
4    | 0x1000   | 0xBBB000
```

- With VMID, translations from different VMs can coexist in the TLB.
- VM switches can avoid full TLB flushes.
- One-line meaning:
  - Which virtual machine is this translation for?

## ASID vs VMID

| Item | ASID | VMID |
|---|---|---|
| Identifies | process address space | virtual machine |
| Managed by | OS | hypervisor |
| Avoids | process-switch TLB flush | VM-switch TLB flush |
| Translation level | process address translation | virtualization translation |

## ASID and VMID Together

- In virtualized systems, both may exist at the same time.
- A VM contains its own OS.
- That guest OS contains its own processes.

```txt
VM
|-- Process A (ASID 1)
`-- Process B (ASID 2)
```

- A full TLB translation context may include:
  - VMID
  - ASID
  - virtual address

```txt
(VMID, ASID, VA)
```

## ARM Comparison

- ARM virtualization is a useful mental model.
- It commonly has:
  - VMID to distinguish virtual machines
  - ASID to distinguish processes inside a VM
- A TLB translation may be conceptually tagged by:

```txt
(VMID, ASID, VA)
```

- This is not the RISC-V register layout.
- It is only a useful comparison:
  - ARM and RISC-V both need VM-level and process-level tagging.
  - RISC-V places ASID in `satp` / `vsatp`.
  - RISC-V places VMID in `hgatp`.

## RISC-V Register Summary

- In RISC-V:
  - `satp` contains ASID
  - `vsatp` contains ASID
  - `hgatp` contains VMID
- So `vsatp` is not the VMID register.
- The VMID is part of the hypervisor stage.
- The official RISC-V privileged architecture references at the end are the sources to check for exact register fields.

| Register | Main translation role | Identifier |
|---|---|---|
| `satp` | supervisor address translation | ASID |
| `vsatp` | guest supervisor address translation | ASID |
| `hgatp` | guest physical to machine physical translation | VMID |

- VMID is not stored in `satp`.
- VMID is not stored in `vsatp`.
- VMID is stored in `hgatp`.

## satp

- `satp` means Supervisor Address Translation and Protection.
- It is used by:
  - a normal OS without virtualization
  - the host supervisor context when running outside guest mode

```txt
satp
+------+--------+---------+
| MODE |  ASID  |   PPN   |
+------+--------+---------+
```

- `MODE` selects translation mode.
- `ASID` identifies an address space.
- `PPN` points to the root page table.
- `satp` is about normal supervisor virtual memory.

## vsatp

- `vsatp` means Virtual Supervisor Address Translation and Protection.
- It is used by the guest OS inside a VM.
- The guest OS still needs process isolation.
- Therefore `vsatp` also contains ASID.

```txt
vsatp
+------+--------+---------+
| MODE |  ASID  |   PPN   |
+------+--------+---------+
```

- This ASID belongs to the guest virtual address space.
- It helps distinguish processes inside the guest OS.
- `vsatp` is about guest OS process address spaces, not VM identity.

## hgatp

- `hgatp` means Hypervisor Guest Address Translation and Protection.
- It is used by the hypervisor.
- It controls guest physical to machine physical translation.
- This is where VMID lives.
- The `hgatp` and VMID details belong to the RISC-V hypervisor extension material referenced below.

```txt
hgatp
+------+--------+---------+
| MODE |  VMID  |   PPN   |
+------+--------+---------+
```

- `VMID` identifies the virtual machine.
- `PPN` points to the stage-2 translation root.
- `hgatp` is about VM memory ownership and isolation.

## Why satp and vsatp Both Have ASID

- Both host OS and guest OS need process address spaces.
- A guest OS also runs multiple processes.
- So the guest OS also needs ASIDs.
- That is why `vsatp` contains ASID, not VMID.

## Two-Stage Translation

- RISC-V virtualization uses two-stage translation.
- Stage 1:
  - Guest Virtual Address to Guest Physical Address
  - controlled by `vsatp`
  - uses guest ASID
- Stage 2:
  - Guest Physical Address to Machine Physical Address
  - controlled by `hgatp`
  - uses VMID
- This is why both ASID and VMID can matter for one guest memory access.

```txt
Guest Process VA
    |
    |  vsatp + guest ASID
    v
Guest Physical Address
    |
    |  hgatp + VMID
    v
Machine Physical Address
```

- A compact way to remember it:

```txt
GVA --guest page table--> GPA --hypervisor page table--> MPA
```

## Guest Physical Address Is Not Real Physical Address

- The guest OS thinks it owns physical memory.
- But the guest physical address is not the real DRAM address.
- It is a fake physical address from the guest OS point of view.
- The hypervisor maps it to real machine memory.
- This is the trick that lets each VM believe it has its own physical machine.

## Why hgatp Ends at Machine Physical Address

- `hgatp` is the second-stage translation root.
- It translates guest physical addresses to real machine physical addresses.
- The result is the real hardware DRAM address.
- Some documents call this:
  - Host Physical Address
  - Machine Physical Address
- `Machine Physical Address` is often clearer.
- `Host Physical Address` is common wording, but it can sound like the address belongs to the host OS virtual memory system.
- In this note, `Machine Physical Address` means the real hardware physical address.

## Where Is the Host OS?

- The host OS does not participate in every address translation.
- That would be too slow.
- Runtime translation is done by MMU hardware and TLB walkers.
- The host OS or hypervisor is on the management path:
  - allocates memory
  - creates mappings
  - owns page tables
  - updates stage-2 mappings
- The MMU is on the fast path:
  - looks up TLB entries
  - walks page tables on TLB miss
  - performs `GVA -> GPA -> MPA`
- The host OS is not called on every memory access.

## Type-1 Hypervisor

- Type-1 hypervisor runs directly on hardware.
- Examples:
  - VMware ESXi
  - Xen
  - Hyper-V in some configurations

```txt
Hardware
   |
   v
Hypervisor
   |
   v
Guest OS
```

- In this model, `GPA -> Machine PA` is direct and intuitive.
- The hypervisor manages the real machine memory.
- There may be no separate host OS.

## Type-2 Hypervisor

- Type-2 hypervisor runs with a host OS.
- Examples:
  - QEMU + KVM
  - VMware Workstation
  - VirtualBox

```txt
Hardware
   |
   v
Host OS
   |
   v
Hypervisor / VMM
   |
   v
Guest OS
```

- It may look like translation should go through host OS virtual memory.
- But that is not the runtime hardware translation path.
- The host OS helps set up and manage the memory, but the MMU still performs the fast translation.

## KVM Example

- KVM is part of the Linux kernel.
- Linux controls machine memory.
- KVM can build stage-2 page tables.
- MMU hardware uses those tables directly.
- Runtime address translation is:

```txt
Guest VA
    |
    v
Guest PA
    |
    v
Machine PA
```

- It is not:

```txt
Guest VA
    |
    v
Host OS VA
    |
    v
Machine PA
```

- Host OS virtual addresses are used by host processes.
- Guest memory translation uses guest and hypervisor page tables instead.

## Host OS Responsibility

- The host OS is responsible for management, not every translation.
- It handles:
  - memory allocation
  - page ownership
  - stage-2 page table construction
  - invalidation when mappings change
  - VM memory accounting
- The MMU handles the fast path.
- This split is important:
  - software builds and changes mappings
  - hardware uses mappings for normal memory accesses

## Complete Translation Diagram

```txt
Guest Process VA
    |
    v
Guest page table
(vsatp + ASID)
    |
    v
Guest Physical Address
(fake physical address)
    |
    v
Hypervisor page table
(hgatp + VMID)
    |
    v
Machine Physical Address
(real DRAM)
```

- If the translation is already cached, the TLB can skip most of the page-table walking.
- The ASID and VMID tags help decide whether the cached translation belongs to the current process and VM.

## Key Correction

- The wrong mental model is:
  - `satp` has ASID
  - `vsatp` has VMID
- The correct RISC-V mapping is:
  - `satp` has ASID
  - `vsatp` has ASID
  - `hgatp` has VMID
- This makes sense because the guest OS still needs ASIDs for its own processes.

## References

- RISC-V privileged architecture manual, HTML snapshot:
  - https://riscv.github.io/riscv-isa-manual/snapshot/privileged
- RISC-V privileged architecture manual, PDF:
  - https://docs.riscv.org/reference/isa/_attachments/riscv-privileged.pdf

## Final Summary

- ASID identifies a process address space.
- VMID identifies a virtual machine.
- Both help avoid unnecessary TLB flushes.
- RISC-V stores ASID in `satp` and `vsatp`.
- RISC-V stores VMID in `hgatp`.
- Guest physical address is not a real physical address.
- Stage-2 translation maps guest physical address to machine physical address.
- The host OS or hypervisor builds the mappings.
- The MMU hardware performs the translation fast path.
