---
title: TLM Event-Driven Simulation and Its Differences from gem5
description: Notes about how SystemC TLM uses discrete-event simulation, how its abstraction differs from RTL, and how TLM compares with gem5.
tags:
- systemc
- tlm
- gem5
- simulation
- computer-architecture
- research-notes
---

## Question: Is TLM an Event-Driven Simulation?

- Yes, but the precise answer depends on which level of Transaction-Level Modeling (TLM) is being discussed.
- SystemC TLM models normally run on top of the SystemC simulation kernel.
- The SystemC kernel is fundamentally a discrete-event simulation (DES) kernel.
- Therefore:
  - the SystemC kernel is event-driven
  - TLM models normally execute on an event-driven kernel
  - TLM intentionally reduces low-level signal events to improve simulation speed

## RTL Simulation

- RTL simulation is a fine-grained form of event-driven simulation.
- A simplified execution flow is:

```txt
clock edge
  -> update registers
  -> evaluate combinational logic
  -> generate signal-change events
  -> evaluate affected processes again
```

- RTL simulation can generate many events, including:
  - clock edges
  - signal changes
  - process wakeups
  - handshake transitions
- The simulator must track behavior at the signal and often cycle level.

## TLM Simulation

- TLM raises the abstraction level from individual signals to transactions.
- A simplified flow is:

```txt
CPU sends a read transaction
  -> bus transfers the transaction
  -> memory returns a response
```

- A model may represent the operation directly with a call such as:

```cpp
socket->b_transport(trans, delay);
```

- One function call may represent:
  - an entire bus transaction
  - one memory access
  - a DMA transfer
- The model does not necessarily simulate:
  - every clock cycle
  - every bus signal
  - every handshake signal
- As a result, TLM can require far fewer simulation events than RTL.

## Temporal Decoupling

- Higher-level TLM models, especially TLM-2.0 Loosely Timed (LT) models, may use temporal decoupling.
- A CPU model may:

```txt
execute 1,000 instructions
  -> accumulate local simulated time
  -> synchronize with the SystemC kernel
```

- This can look different from traditional event-driven simulation because much of the work runs continuously inside a process between synchronization points.
- However:
  - the SystemC kernel remains event-driven
  - synchronization still depends on simulation time and events
  - low-level timing and signal activity have simply been abstracted away

## Event-Driven Behavior by Modeling Level

| Modeling level | Event-driven behavior |
| --- | --- |
| RTL | Yes; strongly dependent on fine-grained events |
| SystemC cycle-accurate model | Yes |
| TLM-2.0 Approximately Timed (AT) | Yes, usually with fewer and higher-level events |
| TLM-2.0 Loosely Timed (LT) | Yes, but event activity is greatly reduced |
| Temporally decoupled TLM | The kernel is still event-driven, but execution between synchronization points may not depend on kernel events |

## Short Answer

- TLM can usually be described as a high-level abstraction running on an event-driven simulator.
- More precisely:
  - SystemC TLM normally runs on an event-driven SystemC kernel.
  - It changes the simulation granularity from signals and low-level events to transactions.
  - Events still exist, but their role is much less visible than in RTL simulation.

## Question: How Is TLM Different from gem5?

- SystemC TLM and gem5 can both be used for system-level simulation.
- Their primary goals are different:
  - TLM focuses on hardware system modeling and communication between components.
  - gem5 focuses on computer architecture simulation and the performance behavior of CPUs, caches, and memory systems.

## Simulation Kernels

### SystemC TLM

- TLM normally runs on the SystemC kernel.
- A simplified flow is:

```txt
event
  -> process wakeup
  -> transaction
  -> next event
```

- Important SystemC concepts include:
  - `SC_THREAD`
  - `SC_METHOD`
  - `sc_event`
  - `sc_time`
- The SystemC kernel is a discrete-event simulator.

### gem5

- gem5 is also a discrete-event simulator.
- A simplified flow is:

```txt
event queue
  -> advance to an event tick
  -> execute the event
  -> schedule new events
```

- Depending on the selected gem5 models, events may represent:
  - CPU pipeline activity
  - cache requests and misses
  - memory-controller activity
  - DRAM responses
- From a simulation-theory perspective, both SystemC and gem5 use discrete-event simulation.
- The main difference is not whether they are event-driven.
- The main difference is what each event or transaction represents.

## Abstraction-Level Difference

### TLM Example

- A TLM model may represent a request as:

```txt
CPU reads address 0x1000
```

- The operation may be handled by one blocking transport call:

```cpp
socket->b_transport(trans, delay);
```

- The transaction can abstract the internal steps of the interconnect and memory system.

### gem5 Example

- A detailed gem5 configuration may break the same read into:

```txt
CPU sends request
  -> TLB lookup
  -> L1 cache access
  -> L1 miss
  -> L2 cache access
  -> L2 miss
  -> memory controller
  -> DRAM access
  -> response
```

- This can generate many simulator events.
- The exact granularity depends on the selected gem5 CPU, cache, and memory models.

| Characteristic | SystemC TLM | Detailed gem5 model |
| --- | --- | --- |
| Typical granularity | Transaction-level | Architectural or microarchitectural |
| Typical speed | Faster | Slower |
| Internal detail | Lower or abstracted | Higher |
| Main focus | Component interaction and system integration | Architecture behavior and performance |

- These are typical tendencies rather than absolute rules.
- TLM can include detailed timing, and gem5 can also use faster, less detailed models.

## Typical Uses of TLM

- TLM is often used during early SoC development.
- A model may include:

```txt
CPU
  -> AXI interconnect
  -> DMA
  -> DDR memory
```

- Common goals include:
  - evaluating whether the system architecture is reasonable
  - hardware and software co-design
  - virtual-platform development
  - IP integration
  - early software development
- A TLM platform can run before the RTL implementation is complete.

## Typical Uses of gem5

- gem5 is commonly used to study computer architecture.
- Research topics include:
  - cache size and organization
  - branch predictors
  - out-of-order CPU behavior
  - memory hierarchy design
  - interconnect behavior
- Example research question:

```txt
If the L2 cache grows from 1 MiB to 2 MiB,
how much does IPC improve?
```

- This type of performance study is one of gem5's main strengths.

## Timing Detail

### TLM-2.0 Loosely Timed Model

- A memory operation may be represented as:

```txt
read memory
  -> annotated delay = 100 ns
  -> response completes
```

- The model may care that the operation finishes after 100 ns without describing every internal step.

### Detailed gem5 Model

- A detailed gem5 model may track activity over many ticks or cycles:

```txt
cycle 101
cycle 102
cycle 103
...
cycle 350
```

- Depending on the chosen model, gem5 can expose detailed metrics such as:
  - instructions per cycle (IPC)
  - stall cycles
  - cache miss rate
  - branch mispredictions
  - memory latency

## Can TLM and gem5 Be Used Together?

- Yes.
- A mixed simulation environment may look like:

```txt
SystemC TLM SoC
  -> gem5 CPU model
  -> TLM bus
  -> TLM memory
```

- In this arrangement:
  - gem5 models the CPU or another architecture-sensitive component
  - SystemC TLM models the rest of the SoC at a higher abstraction level
- The combination can balance:
  - CPU and architecture detail
  - whole-system simulation speed
  - reuse of existing SystemC IP models

## Travel Analogy

- Imagine simulating a trip from Taipei to Kaohsiung.

### TLM View

- Record only:

```txt
08:00 depart
10:00 arrive
```

- The entire trip is treated as one transaction.

### Detailed gem5 View

- Record many intermediate steps:

```txt
08:00 board
08:01 pass location A
08:02 pass location B
...
09:59 pass location Z
10:00 arrive
```

- The trip is divided into many events so that the internal behavior can be studied.

## Final Takeaway

- Both SystemC TLM and gem5 are generally built around discrete-event simulation.
- TLM usually focuses on:
  - functional communication between system components
  - transaction-level behavior
  - coarse or abstracted timing
  - fast system integration
- gem5 usually focuses on:
  - architecture and microarchitecture
  - detailed CPU, cache, and memory behavior
  - performance analysis
- The event granularity in a detailed gem5 model is usually finer than in a high-level TLM model.
- The best one-sentence distinction is:
  - TLM asks how system components interact.
  - gem5 asks how architectural details affect behavior and performance.
