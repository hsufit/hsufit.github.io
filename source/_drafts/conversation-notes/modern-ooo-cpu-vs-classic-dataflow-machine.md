---
title: Modern OoO CPU vs Classic Dataflow Machine
description: Notes about why modern out-of-order CPUs feel dataflow-like, and why they are still not classic dataflow machines.
tags:
- cpu
- microarchitecture
- ooo
- dataflow
- tomasulo
---

## Question

- I am doing CPU research.
- I saw a statement about modern OoO CPUs and classic dataflow machines.
- The statement was:
  - 1980s dataflow machines use token-driven execution.
  - Modern OoO CPUs use tag-driven speculative execution.
  - The concepts are very similar.
- I wanted a clearer explanation.

## Core Idea

- Modern superscalar OoO CPUs are locally similar to dataflow machines.
- The backend schedules instructions based on operand readiness.
- That feels very dataflow-like.
- But the control model is very different.
- A modern OoO CPU is still built around sequential ISA semantics.
- It only uses dataflow-like scheduling inside a bounded backend window.

## Classic von Neumann Execution

- Traditional execution is control-flow driven.
- The Program Counter decides instruction order.
- The flow is:

```txt
PC -> fetch -> decode -> execute
```

- Example:

```asm
1: R1 = R2 + R3
2: R4 = R1 * 5
3: R8 = R9 + R10
```

- In a simple pipeline, instruction 1 comes before 2.
- Instruction 2 comes before 3.
- Even if instruction 3 is independent, the instruction stream is still PC ordered.
- Execution order is mainly controlled by the Program Counter.

## Classic Dataflow Machine

- Classic dataflow machines took a radical idea.
- Instructions should not execute just because the PC reaches them.
- Instructions should execute when their operands are ready.
- The program is represented as a dataflow graph.

```txt
 b ----\
        (+) ---- a ----\
 c ----/                (*) ---> d
                         /
 e ---------------------/

 g ----\
        (+) ---> f
 h ----/
```

- Execution is driven by token arrival.
- A token carries data to a graph node.
- When a node receives all required input tokens, it fires.
- There is no normal PC-driven sequential issue.
- There is no reorder buffer.
- The machine computes where the data flows.

## Token-Driven Execution

- A token is like a small data packet.
- It usually contains:
  - operand value
  - destination information
  - sometimes a tag or context identifier
- A node fires when all needed tokens arrive.
- The key idea:
  - data movement triggers execution
  - not program counter order

## Modern OoO CPU

- Modern OoO CPUs still execute sequential ISAs.
- Examples:
  - x86
  - ARM
  - RISC-V
- The frontend still fetches instructions by PC.
- The frontend still decodes an instruction stream.
- But after rename, the backend builds a dynamic dependency graph.

## Rename Changes the View

- Before rename, instructions use architectural registers.
- After rename, instructions use physical registers.
- Example:

```asm
1: R1 = R2 + R3
2: R4 = R1 * 5
3: R8 = R9 + R10
```

- After rename, it may become:

```asm
1: P1 = P2 + P3
2: P4 = P1 * 5
3: P8 = P9 + P10
```

- The scheduler can see:
  - instruction 2 depends on `P1`
  - instruction 3 is independent
- Instruction 3 can execute before instruction 2.
- This is the point where OoO starts to feel dataflow-like.

## Why OoO Looks Like Dataflow

- The scheduler waits for operands to become ready.
- A reservation station or issue queue tracks source readiness.
- Example:

```txt
src1 ready | src2 ready | op
0          | 1          | ADD
```

- When `src1` becomes ready:

```txt
src1 ready | src2 ready | op
1          | 1          | ADD
```

- The instruction can issue.
- This is similar to a small dataflow node.
- The instruction fires when inputs are ready.

## Tomasulo Is Dataflow-Like

- Tomasulo's algorithm already had many dataflow-like ideas.
- It introduced:
  - dynamic scheduling
  - register renaming
  - dependency wakeup
  - tag broadcast
  - reservation stations
- The Common Data Bus broadcasts results.
- Example broadcast:

```txt
(P17, 42)
```

- Waiting instructions compare their source tags.
- If `src_tag == P17`, the operand is ready.
- This looks similar to token matching.
- The original Tomasulo paper is linked in the References section.

## Difference 1: Who Controls Execution

- In a classic dataflow machine, data dependency is the main controller.
- There is no global sequential instruction order.
- In a modern OoO CPU, the architectural program order still matters.
- OoO execution is only temporary internal reordering.
- The CPU must eventually retire instructions in order.

## Difference 2: OoO Has a ROB

- ROB means Reorder Buffer.
- Modern OoO flow is roughly:

```txt
fetch -> rename -> issue -> execute -> ROB -> commit
```

- The ROB provides:
  - precise exceptions
  - branch recovery
  - interrupt handling
  - in-order commit
  - architectural state recovery
- Classic dataflow machines do not have this same commit model.
- They are graph reduction systems, not speculative sequential machines.

## Difference 3: Speculation

- Modern OoO CPUs depend heavily on speculation.
- Branch prediction guesses the future PC.
- The backend executes down predicted paths.
- If the prediction is wrong, the CPU flushes and recovers.
- Classic dataflow machines are usually not built around global control-flow speculation.
- They execute based on token availability.

## Difference 4: Memory Model

- Memory is one of the biggest differences.
- Ideal dataflow works best with explicit dependencies.
- Real programs use memory heavily.
- Example:

```c
*p = 5;
x = *q;
```

- Can these operations reorder?
- It depends on whether `p` and `q` alias.
- Modern OoO CPUs need:
  - Load/Store Queue
  - memory disambiguation
  - store-to-load forwarding
  - speculation
  - replay
- Memory aliasing is one reason pure dataflow machines struggled with real programs.

## Difference 5: Granularity

- A classic dataflow machine tries to execute a whole program as a dataflow graph.
- A modern OoO CPU only creates a local dynamic dataflow window.
- Example window sizes may be:
  - 128 entries
  - 256 entries
  - 512 entries
- The CPU sees only a bounded part of the program at a time.
- That makes OoO a bounded dynamic dataflow engine, not a whole-program dataflow machine.

## Why Classic Dataflow Did Not Win

- Dataflow was once seen as a possible future of computing.
- It did not become the mainstream CPU model.
- Main problems:
  - token matching was expensive
  - communication traffic was huge
  - memory aliasing was hard
  - precise exceptions were hard
  - OS support was difficult
- The bottleneck became communication, not only computation.

## Why OoO Won

- OoO keeps the sequential ISA model.
- That makes it compatible with existing software.
- OoO adds local dataflow scheduling inside the backend.
- The ROB restores precise architectural state.
- This is a successful compromise:
  - sequential ISA outside
  - speculative dataflow-like backend inside
  - in-order retirement at the end

## Best Mental Model

- Modern OoO CPU:
  - frontend is mostly in-order
  - rename builds dependencies
  - backend is dataflow-like
  - retirement is in-order

```txt
Frontend     Rename        OoO Backend       Retirement
PC-driven -> deps made -> dataflow-like -> in-order state
```

## One-Sentence Summary

- A modern OoO CPU is a speculative machine with von Neumann sequential semantics on the outside and a bounded dynamic dataflow engine inside the backend.

## Research Note

- A good way to describe this:
  - Tomasulo plus reservation stations is a bounded dynamic dataflow engine.
- But modern CPUs are still not pure dataflow computers because they have:
  - ROB
  - speculation
  - precise state
  - sequential commit
  - memory ordering constraints
- BOOM and XiangShan are useful open-source references for seeing modern OoO ideas in RISC-V implementations.

## References to Read

- Tomasulo, [*An Efficient Algorithm for Exploiting Multiple Arithmetic Units*](https://courses.cs.washington.edu/courses/cse548/05wi/files/inclass/Jan24_Tomasulo.pdf)
- Hennessy and Patterson, *Computer Architecture: A Quantitative Approach*
- IBM System/360 Model 91 materials
- Alpha 21264, [*The Alpha 21264 Microprocessor Architecture*](https://cseweb.ucsd.edu/classes/fa07/cse240a/Papers/alpha21264.pdf)
- Intel P6 / Pentium Pro microarchitecture overview
- [BOOM documentation](https://docs.boom-core.org/)
- [XiangShan project](https://xiangshan.cc/)
