---
title: Cache Coherence Snooping and RISC-V TileLink Notes
description: Notes about how snooping protocols work on a bus, and how RISC-V TileLink-style coherence differs from classic bus snooping.
tags:
- cpu
- cache
- coherence
- riscv
- tilelink
---

## Question

- I wanted to understand how a snooping protocol really works on a bus.
- I also wanted to know how this maps to RISC-V systems.
- A specific question was:
  - In a system like RISC-V TileLink, is it really just caches snooping a bus?
  - Or is there hardware in the middle coordinating coherence?

## Short Answer

- In a small system, pure bus snooping can be real.
- In modern SoCs, coherence is usually not pure broadcast snooping.
- TileLink coherence is closer to manager-directed probe coherence.
- It often behaves more like directory-style coherence than classic shared-bus snooping.
- So yes, there is usually hardware in the middle.

## Classic Snooping Protocol

- A snooping protocol is a cache coherence protocol.
- Each cache watches memory transactions on a shared interconnect.
- If another core touches a cache line, each cache checks whether it has that line.
- The cache then updates, downgrades, or invalidates its own copy.

## Classic Bus-Based Snooping Diagram

```txt
                +----------------------+
                |      Main Memory     |
                |        (DRAM)        |
                +----------+-----------+
                           |
                    System Bus / Interconnect
================================================================================
        |                        |                        |
        | snoop bus traffic      | snoop bus traffic      | snoop bus traffic
        v                        v                        v

 +---------------+      +---------------+      +---------------+
 |     CPU 0     |      |     CPU 1     |      |     CPU 2     |
 |               |      |               |      |               |
 | +-----------+ |      | +-----------+ |      | +-----------+ |
 | |   Cache   | |      | |   Cache   | |      | |   Cache   | |
 | | (MESI...) | |      | | (MESI...) | |      | | (MESI...) | |
 | +-----------+ |      | +-----------+ |      | +-----------+ |
 +---------------+      +---------------+      +---------------+
```

## Classic Snooping Flow

- Example:
  - Core 0 wants to write cache line `X`.
  - Core 1 may already have `X`.
  - Core 2 may also have `X`.

```txt
 CPU0 wants to write X
        |
        v
 +------------------+
 | Cache0 sends     |
 | BusRdX(X)        |
 +------------------+
        |
        v
 ================================================== BUS
        |
        +--> Cache1 snoops request
        |      if it has X:
        |         invalidate X
        |
        +--> Cache2 snoops request
               if it has X:
                  invalidate X
```

- `BusRdX(X)` means:
  - I want to read line `X`.
  - I also want exclusive ownership so I can write it.
- Other caches snoop the bus.
- If they have `X`, they invalidate or downgrade their copy.
- Core 0 can then own the line in a writable state.

## MESI View

- Common snooping protocols use states such as:
  - Modified
  - Exclusive
  - Shared
  - Invalid

```txt
                +-----------+
                | Modified  |
                +-----------+
                 ^         |
        write hit|         | flush
                 |         v
+-----------+ <-----> +-----------+
| Exclusive |         |  Shared   |
+-----------+ <-----> +-----------+
                 read miss |
                           v
                     +-----------+
                     | Invalid   |
                     +-----------+
```

## Why Classic Snooping Works

- Classic snooping works naturally when the interconnect is a shared broadcast medium.
- Every cache can see every transaction.
- Every cache can react immediately.
- This is simple and elegant for small systems.

## Why Pure Snooping Does Not Scale

- Broadcast is expensive.
- In a large system, every transaction cannot be sent to every cache.
- Problems include:
  - high wire power
  - high traffic
  - long timing paths
  - difficult arbitration
  - poor scalability with many cores
- With 32 cores or a mesh NoC, pure broadcast snooping becomes too expensive.

## Directory-Based Coherence

- Directory coherence adds a record of who has each cache line.
- The directory may know:

```txt
line A -> Core1, Core7
```

- If Core 0 wants to write `A`, the directory only needs to contact Core 1 and Core 7.
- It does not need to broadcast to all cores.

## TileLink Is Not Classic Bus Snooping

- TileLink is common in Rocket Chip-style RISC-V systems.
- TileLink coherence is message-based.
- It is not simply every cache listening to every bus transaction.
- It is closer to:
  - coherence manager
  - probes
  - permission transfer
  - directory-style metadata
- The Chipyard TileLink/Diplomacy reference in the References section is useful for mapping this idea to Rocket Chip systems.

## TileLink Roles

- TileLink has clients and managers.
- A client may be:
  - L1 cache
  - core-side cache agent
- A manager may be:
  - L2 cache bank
  - memory controller
  - coherence manager
  - LLC bank
- The Chipyard node-type documentation in the References section shows the client/manager terminology.

## TileLink Coherence Messages

- Important TileLink coherence concepts include:
  - `Acquire`
  - `AcquirePerm`
  - `Probe`
  - `ProbeAck`
  - `Grant`
  - `Release`
- These messages transfer cache line permissions.
- The TileLink specification link in the References section is the place to check exact message definitions.

## TileLink-Style Write Flow

- Example:
  - Core 0 wants to write address `A`.
  - Core 1 has cached `A`.

```txt
Core0
  |
  | AcquirePerm(A)
  v
+----------------------+
| L2 / Coherence       |
| Manager              |
+----------------------+
  |
  | Probe(A)
  v
Core1
  |
  | ProbeAck(A)
  v
+----------------------+
| L2 / Coherence       |
| Manager              |
+----------------------+
  |
  | Grant(exclusive)
  v
Core0
```

- Core 0 does not directly snoop Core 1.
- The manager coordinates the permission change.
- The manager checks coherence metadata.
- The manager sends targeted probes.
- Core 0 writes only after it receives the grant.

## Why TileLink Looks Like Snooping

- `Probe` messages look like snoop requests.
- A probe asks another cache to downgrade, invalidate, or report a line.
- The difference is:
  - classic snooping broadcasts on a shared bus
  - TileLink managers send targeted probe messages

## Strict vs Broad Definition

- Strict snooping:
  - every cache monitors a shared bus
  - transactions are broadcast
  - caches independently react

- Broad coherence family usage:
  - any probe or invalidate protocol may be casually called snoop-like

- TileLink is not classic strict snooping.
- TileLink is better described as manager-directed probe coherence.

## RISC-V and Cache Coherence

- RISC-V ISA does not mandate one cache coherence protocol.
- A RISC-V SoC may use:
  - no coherence
  - bus snooping
  - directory coherence
  - TileLink coherence
  - AXI ACE-like coherence
  - custom NoC coherence
- The coherence protocol is a platform and SoC design choice.

## RISC-V Small Snooping System

```txt
                    +----------------------+
                    |      DDR / DRAM      |
                    |   Memory Controller  |
                    +----------+-----------+
                               |
                     +---------+---------+
                     |  AXI / TileLink   |
                     |  Coherent Interconnect
                     +---------+---------+
                               |
        =====================================================
             |                     |                    |
             |                     |                    |
     +-------+------+      +-------+------+     +-------+------+
     |    RISC-V    |      |    RISC-V    |     |    RISC-V    |
     |    Core 0    |      |    Core 1    |     |    Core 2    |
     +--------------+      +--------------+     +--------------+
     |   L1 I$      |      |   L1 I$      |     |   L1 I$      |
     |   L1 D$      |      |   L1 D$      |     |   L1 D$      |
     | MESI Agent   |      | MESI Agent   |     | MESI Agent   |
     +------+-------+      +------+-------+     +------+-------+
            \                     |                    /
             \____________________|___________________/
                          snooping / coherence
```

## RISC-V TileLink-Style System

```txt
              +----------------------------------+
              |          L2 Cache /              |
              |      Coherence Manager           |
              +----------------+-----------------+
                               |
                    TileLink Coherent Interconnect
=================================================================
          |                        |                       |
          |                        |                       |

   +------+-------+        +------+-------+       +------+-------+
   |  Rocket Core |        |  Rocket Core |       |  Rocket Core |
   |      0       |        |      1       |       |      2       |
   +--------------+        +--------------+       +--------------+
   |   L1 ICache  |        |   L1 ICache  |       |   L1 ICache  |
   |   L1 DCache  |        |   L1 DCache  |       |   L1 DCache  |
   +--------------+        +--------------+       +--------------+
```

- This is not pure bus snooping.
- The coherence manager coordinates probes and permissions.

## Textbook-Style RISC-V Coherent SoC

```txt
                  +----------------------+
                  |     Main Memory      |
                  +----------+-----------+
                             |
                    +--------+--------+
                    |   Coherence     |
                    |    Manager      |
                    +--------+--------+
                             |
                 Coherent Interconnect
================================================================================
        |                           |                           |
        |                           |                           |

 +------+-------+           +------+-------+           +------+-------+
 | RISC-V Core0 |           | RISC-V Core1 |           | RISC-V Core2 |
 +--------------+           +--------------+           +--------------+
 |   L1 I$      |           |   L1 I$      |           |   L1 I$      |
 |   L1 D$      |           |   L1 D$      |           |   L1 D$      |
 | MESI FSM     |           | MESI FSM     |           | MESI FSM     |
 +--------------+           +--------------+           +--------------+
```

## Rocket Chip Has Real Managers

- In Rocket Chip and TileLink systems, manager nodes are not just abstract words.
- The diplomacy graph includes:
  - `TLClientNode`
  - `TLManagerNode`
- These nodes correspond to real hardware responsibilities.
- An L2 cache bank or coherence manager may act as a TileLink manager.
- The Rocket Chip repository and Chipyard references are useful follow-up links for this section.

## Modern CPU Coherence

- Modern systems are more complex than old SMP buses.
- Examples:
  - Intel MESIF
  - ARM CMN
  - AMD Infinity Fabric
- These systems may include:
  - directory
  - home agent
  - snoop filter
  - distributed coherence point
  - NoC routing
- The principle is still coherence.
- The implementation is no longer simple broadcast snooping.

## References

- Chipyard TileLink and Diplomacy reference:
  - https://chipyard.readthedocs.io/en/1.12.2/TileLink-Diplomacy-Reference/
- Chipyard TileLink node types:
  - https://chipyard.readthedocs.io/en/1.12.2/TileLink-Diplomacy-Reference/NodeTypes.html
- TileLink specification PDF:
  - https://www.starfivetech.com/uploads/tilelink_spec_1.8.1.pdf
- Rocket Chip repository:
  - https://github.com/chipsalliance/rocket-chip
- Diplomacy paper:
  - https://carrv.github.io/2017/papers/cook-diplomacy-carrv2017.pdf

## Final Summary

- Classic snooping means:
  - shared bus
  - broadcast transaction
  - every cache snoops
- TileLink coherence means:
  - manager-directed messages
  - targeted probes
  - permission transfer
  - directory-like coordination
- A small RISC-V system may use snooping.
- A larger RISC-V SoC usually uses a manager or directory-style coherence mechanism.
- TileLink should be understood as message-based coherent interconnect, not a pure snooping bus.
