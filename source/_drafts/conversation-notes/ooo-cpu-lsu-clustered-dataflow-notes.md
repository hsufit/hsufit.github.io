---
title: OoO CPU LSU, Clustered Scheduler, and Dataflow Notes
description: Discussion notes about multiple LSU design, clustered OoO synchronization, and the evolution from Tomasulo to modern tag-driven execution.
tags:
- cpu
- microarchitecture
- ooo
- lsu
- dataflow
---

## Story

- I wanted to understand whether multiple LSU design is still an active CPU design topic.
- The answer is yes.
- Modern CPUs already use multiple load/store resources.
- The hard part is no longer just adding more LSU pipes.
- The hard part is scaling the memory subsystem without exploding timing, power, and replay complexity.

## Multiple LSU Design Issues

- Multiple LSU means multiple Load/Store Units.
- It is common in modern superscalar and OoO CPUs.
- The difficult topics are:
  - memory ordering complexity
  - load queue scalability
  - store queue scalability
  - replay storms
  - AGU bottlenecks
  - L1D cache bandwidth
  - coherence traffic
  - energy efficiency

## Load Queue and Store Queue Scalability

- The LSQ is one of the most painful parts of an OoO CPU.
- Going from one LSU to two LSU is already hard.
- Going from two to three or four is much harder.
- The complexity grows quickly because memory operations compare against each other.
- A load must check older stores.
- A store must check younger loads for ordering violations.
- The hardware must detect speculation failures.
- The hardware must find store-to-load forwarding candidates.
- This creates large associative searches.
- Large CAM searches burn power.
- Large CAM searches also hurt timing closure.
- LSQ scaling is one of the reasons memory backend design is so expensive.

## Memory Dependence Speculation

- Modern CPUs cannot wait for every older store address before issuing a load.
- Loads often execute speculatively.
- If an older store later resolves to the same address, the load must replay.
- With more LSU resources, more speculative memory operations can be in flight.
- This can create replay storms.
- Replay storms are worse with:
  - SMT
  - server workloads
  - pointer chasing
  - cache misses
  - unresolved store addresses
- The CPU needs memory dependence prediction to reduce useless speculation.

## AGU Bottleneck

- LSU is not only about cache access.
- AGU means Address Generation Unit.
- AGU calculates effective addresses.
- Address generation may include:
  - base register
  - index register
  - scale
  - offset
  - TLB access
  - alias prediction
- AGU timing can become a bottleneck.
- More LSU pipes do not always mean more effective memory throughput.
- AMD Zen 2 is often described as having:
  - three AGUs
  - two load pipes
  - one store pipe
- Many Intel cores also limit memory address generation per cycle.
- The Zen 2 references in the References section are useful for this AGU/LSU bandwidth example.

## LSQ and MSHR Interaction

- MSHR means Miss Status Holding Register.
- It tracks outstanding cache misses.
- A load miss may occupy:
  - a Load Queue entry
  - an MSHR entry
  - speculative state
- The CPU must coordinate them.
- Important questions:
  - What happens if a cache miss is pending while older store addresses are unresolved?
  - When should the CPU detect a memory ordering violation?
  - How does replay interact with pending cache misses?
- This is why LSQ and MSHR interaction is a real design topic.
- The Stack Overflow LSQ/MSHR discussion in the References section is the discussion link that matches this topic.

## Store-to-Load Forwarding

- Store-to-load forwarding becomes harder with more LSU bandwidth.
- Example case:
  - four loads
  - two stores
  - same cycle
  - partial overlap
  - unaligned access
- Forwarding logic becomes an N-by-M compare fabric.
- It must consider:
  - address match
  - byte mask
  - alignment
  - partial forwarding
- This is a critical path risk in high-frequency CPUs.

## Cache Bandwidth Wall

- More LSU pipes are useless if the L1D cache cannot provide enough bandwidth.
- L1D ports are expensive.
- More ports mean:
  - more area
  - harder routing
  - higher power
  - harder timing
- Common techniques:
  - banked L1D cache
  - pipelined cache access
  - clustered LSU
  - load/store bandwidth balancing
- Apple cores are often aggressive in memory backend width.
- AMD and Intel are usually careful about power and area tradeoffs.
- WikiChip and AnandTech Zen 2 references are useful examples for load/store bandwidth descriptions.

## Replay System

- Replay is a huge part of modern OoO CPU backend complexity.
- Replay may happen because of:
  - speculative load violation
  - cache bank conflict
  - TLB miss
  - line fill collision
  - failed forwarding
- A replay system can behave like a small scheduler.
- It must avoid flushing too much work.
- Selective replay is much better than replaying everything.

## Energy Efficiency

- LSU logic consumes a lot of power.
- Expensive parts include:
  - associative searches
  - forwarding compares
  - coherence snoops
  - wakeup and replay logic
- Modern design is not about blindly adding LSU pipes.
- Better options may include:
  - smarter memory dependence prediction
  - better prefetching
  - better cache banking
  - better replay control

## SMT Makes LSU Harder

- SMT means Simultaneous Multithreading.
- Multiple threads may share:
  - LSQ resources
  - cache ports
  - MSHRs
  - replay bandwidth
- Problems include:
  - starvation
  - fairness
  - replay amplification
  - port contention
- SMT plus multiple LSU makes arbitration much harder.

## Modern Trend

- Modern CPUs do not simply increase LSU count without limit.
- Intel and AMD often improve:
  - OoO window size
  - prefetching
  - memory dependence prediction
  - cache behavior
- Apple tends to build very wide and aggressive memory backends.
- The tradeoff is area and power.

## What Is the Load/Store Pipeline?

- The load/store pipeline is not only used for cache misses.
- It is used for normal memory operations too.
- It usually includes:
  - address generation
  - address translation
  - memory ordering checks
  - store-to-load forwarding
  - cache access
  - miss handling
  - replay handling
- Cache miss is only one possible event in the load/store pipeline.

## Clustered Scheduler Problem

- Clustered scheduling is one of the hardest parts of modern OoO CPUs.
- The goal is to split the scheduler to reduce timing and power.
- The problem is synchronization.
- A cluster cannot be fully independent.
- Real designs are closer to:
  - shared dependency universe
  - partitioned scheduling domains
- The CPU wants local scheduling without breaking the global OoO illusion.

## Wakeup Synchronization

- Wakeup is the core synchronization problem.
- Example:

```asm
MUL x1, x2, x3
ADD x4, x1, x5
```

- The multiply may execute in one cluster.
- The add may wait in another cluster.
- The add must know when `x1` is ready.
- Modern CPUs solve this with tag broadcast.

```txt
          result tag broadcast
                   |
   +---------------+---------------+
   v               v               v
 ALU queue      AGU queue       FPU queue
```

- Scheduling can be local.
- Dependency wakeup is still global or semi-global.
- This global wakeup network is expensive.

## Load/Store Ordering Synchronization

- Memory ordering is harder than normal ALU dependency.
- A store may be in one memory cluster.
- A load may be in another memory cluster.
- The ordering must still be globally correct.
- The LSQ often remains the global memory ordering authority.
- Even with clustered schedulers, the memory ordering domain is usually centralized or semi-centralized.

## Replay Across Clusters

- Replay is difficult when dependent instructions have already issued in different clusters.
- A failed load may affect:
  - ALU cluster instructions
  - AGU cluster instructions
  - FPU cluster instructions
- Modern CPUs try to use selective replay.
- The CPU marks dependent chains invalid.
- Then it re-wakes and re-schedules them.
- This avoids flushing the entire backend.

## Tag-Based Synchronization

- Modern OoO CPUs rely on physical register tags.
- After register renaming, instructions stop depending on architectural registers.
- They depend on physical registers.
- Example:
  - architectural register: `x1`
  - physical register: `P37`
- The wakeup network broadcasts that `P37` is ready.
- All clusters compare local waiting operands against the broadcast tag.

## Cluster Synchronization Summary

- Cluster synchronization is usually not direct queue-to-queue communication.
- It is based on:
  - shared physical register namespace
  - tag broadcast
  - shared ROB
  - shared or semi-shared LSQ
- The key challenge is maintaining global dependency consistency.

## Hierarchical Wakeup

- Global wakeup is expensive.
- Modern designs may use hierarchical wakeup.
- Local wakeup can be fast.
- Remote wakeup can be slower.
- Example:
  - same cluster wakeup: 1 cycle
  - cross cluster wakeup: 2 cycles or more
- This creates non-uniform scheduling latency.
- Modern schedulers start to look a little like NUMA systems.

## Frontend, Backend, and Retirement

- Modern OoO CPU can be viewed as three worlds:

```txt
Frontend   ->   OoO Backend   ->   Retirement
in-order        dataflow-ish       in-order
```

- Frontend is mostly in-order.
- Backend is tag-driven and dataflow-like.
- Retirement restores the in-order architectural illusion.

## Why Frontend Is Mostly In-Order

- Frontend work is PC-driven.
- It includes:
  - fetch
  - branch prediction
  - decode
  - rename
  - dispatch
- Instruction order matters before rename.
- Fetch depends on predicted control flow.
- OoO decode is usually not worth the cost.

## Rename Is the Boundary

- Rename is the important boundary.
- Before rename:
  - instructions use architectural registers
  - the machine looks sequential
- After rename:
  - instructions use physical registers
  - dependencies become explicit
  - WAR and WAW hazards are removed
- After rename, the backend behaves like a dataflow engine.

## OoO Backend as Dataflow-Like Machine

- Backend does not mainly ask: what is next in program order?
- Backend asks:
  - are operands ready?
  - are resources ready?
  - are memory dependencies satisfied?
- Instructions wait in queues.
- When dependencies are ready, instructions issue.
- This is why OoO backend feels like event-driven graph execution.

## ROB Restores Sequential Illusion

- Pure dataflow would complete instructions out of order.
- ISA requires precise architectural state.
- ROB means Reorder Buffer.
- The ROB forces in-order retirement.
- The outside world sees a sequential CPU.
- Internally, the backend may have executed instructions out of order.

## Modern CPU Conceptual Model

```txt
             +-------------+
             | Frontend    |
             | sequential  |
             +------+------+
                    |
                    v
             +-------------+
             | Rename      |
             | dependency  |
             | graph gen   |
             +------+------+
                    |
                    v
      +-----------------------------+
      | OoO Backend                 |
      | distributed dataflow-ish    |
      | schedulers + queues         |
      +-------------+---------------+
                    |
                    v
              +-----------+
              | ROB       |
              | reorder   |
              +-----+-----+
                    |
                    v
          architectural state
```

## Token vs Tag

- Classic dataflow machines are token-driven.
- Tomasulo and modern OoO CPUs are tag-driven.
- They look similar, but the mechanism is different.

## Classic Dataflow Token

- A token carries actual data.
- A token also carries routing information.
- Example:
  - value: `5`
  - destination: `node42.inputA`
- When all input tokens arrive, the node fires.
- The program is a graph.
- Execution is driven by data movement.

## Tomasulo Tag

- A tag is not the data.
- A tag identifies who will produce the data.
- Example:
  - operand not ready
  - wait for producer `RS3`
- The reservation station waits until the tag is resolved.
- When the producer broadcasts the result, waiting instructions wake up.

## Token vs Tag Difference

| Topic | Dataflow token | Tomasulo / OoO tag |
|---|---|---|
| What it is | actual data packet | producer identifier |
| Contains value | yes | no |
| Contains dependency info | implicit | explicit |
| Execution trigger | token arrival | tag readiness |
| Storage model | distributed graph | RS, ROB, PRF, queues |

## Tomasulo and Modern OoO

- Tomasulo looks dataflow-like because instructions wait for operands.
- The reservation station behaves like a small dataflow node.
- But modern CPUs are not pure Tomasulo machines.
- They are Tomasulo-like in dependency tracking.
- They also use:
  - physical register files
  - reorder buffers
  - speculative execution
  - clustered schedulers

## Reservation Station-Centric Tomasulo

- Early Tomasulo stores operands inside reservation stations.
- A reservation station stores:
  - instruction
  - operand values
  - dependency tags
- When operands are ready, the instruction issues.
- The Common Data Bus broadcasts results to waiting reservation stations.

## Why RS-Centric Design Does Not Scale

- It creates large broadcast networks.
- It duplicates operand storage.
- It makes every reservation station compare incoming tags.
- It increases CAM power.
- It creates long wires and difficult timing.

## Modern PRF-Centric OoO

- Modern CPUs usually use a Physical Register File.
- PRF stores the actual operand values.
- Issue queues store mostly tags and scheduling metadata.
- When an instruction issues, it reads operands from the PRF.
- This is more scalable than storing operand values in every reservation station.

```txt
Tomasulo RS-centric

        RESULT BUS
            |
   +--------+--------+
   v        v        v
  RS1      RS2      RS3
 [val]    [val]    [val]

Modern PRF-centric

        wakeup tags
            |
   +--------+--------+
   v        v        v
  IQ1      IQ2      IQ3
 [tag]    [tag]    [tag]

            |
            v
   +------------------+
   | Physical RF      |
   | actual operands  |
   +------------------+
```

## Real Hardware and References to Read

- IBM System/360 Model 91 / Tomasulo original paper
  - [An Efficient Algorithm for Exploiting Multiple Arithmetic Units](https://courses.cs.washington.edu/courses/cse548/05wi/files/inclass/Jan24_Tomasulo.pdf)
- Intel P6 / Pentium Pro
  - important OoO milestone
- Alpha 21264
  - [The Alpha 21264 Microprocessor Architecture](https://cseweb.ucsd.edu/classes/fa07/cse240a/Papers/alpha21264.pdf)
- Intel Core / Skylake
  - modern PRF-centric OoO style
- Apple Firestorm
  - aggressive wide OoO backend
- BOOM
  - [RISC-V BOOM documentation](https://docs.boom-core.org/)
- XiangShan
  - [XiangShan project](https://xiangshan.cc/)
- AMD Zen 2 LSU / AGU examples
  - [WikiChip Zen 2](https://en.wikichip.org/wiki/amd/microarchitectures/zen_2)
  - [AnandTech Zen 2 microarchitecture analysis](https://www.anandtech.com/show/14525/amd-zen-2-microarchitecture-analysis-ryzen-3000-and-epyc-rome)
- LSQ and MSHR discussion
  - [How does Load Store Queue work in the presence of MSHR?](https://stackoverflow.com/questions/65851556/how-does-load-store-queue-work-in-the-presence-of-mshr)
- Hennessy and Patterson
  - *Computer Architecture: A Quantitative Approach*

## Further Topics Mentioned

- Intel Sunny Cove / Golden Cove LSU details.
- Apple M1 / M2 memory subsystem.
- Load Queue CAM RTL structure.
- Store forwarding hardware.
- Replay engine architecture.
- Memory disambiguation predictors.
- GPU LSU versus CPU LSU differences.
- Clustered LSU architecture.
- Banked L1D cache design.
- Multi-LSU timing closure.
- Why Apple can build very wide memory backends.
- How BOOM and XiangShan implement LSU and replay.

## Final Summary

- Multiple LSU design is a real modern CPU problem.
- The hard part is memory ordering, LSQ scaling, forwarding, cache bandwidth, replay, and power.
- Clustered schedulers reduce local timing pressure but still need global dependency visibility.
- Modern OoO backend is best understood as a speculative, tag-driven, dataflow-like engine.
- The frontend is mostly in-order.
- The backend is event-driven.
- The ROB restores in-order architectural state.
- Classic dataflow machines move tokens carrying data.
- Tomasulo and modern OoO CPUs use tags to track future producers.
- Modern CPUs evolved from RS-centric Tomasulo toward PRF-centric OoO designs.
