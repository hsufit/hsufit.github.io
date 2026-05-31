---
title: RTL assign vs always_ff Pipeline Timing Notes
description: Notes about combinational outputs, registered outputs, FSM timing semantics, pipeline cuts, bypass paths, and the latency versus frequency tradeoff.
tags:
- rtl
- systemverilog
- pipeline
- timing
- digital-design
---

## Question

- I wanted to understand the difference between calculating an output directly in a flip-flop and using a combinational assignment.
- Example with registered output:

```systemverilog
always_ff @(posedge clk)
  out_state <= (in_a & in_b);
```

- Example with combinational output:

```systemverilog
assign out_state = in_a & in_b;
```

- The real question is:
  - should an FSM or RTL output be registered?
  - or should it stay combinational?

## Main Idea

- Both styles can be correct.
- The difference is not just coding style.
- The real difference is the timing meaning of the signal.

## assign Means Combinational Output

```systemverilog
assign out_state = in_a & in_b;
```

- Meaning:
  - the output reflects the current input value.
  - if `in_a` or `in_b` changes, `out_state` can change in the same cycle.
- Hardware meaning:
  - combinational logic
  - no memory
  - zero-cycle response

```txt
cycle N:
  in_a, in_b -> out_state
```

## always_ff Means Registered Output

```systemverilog
always_ff @(posedge clk)
  out_state <= (in_a & in_b);
```

- Meaning:
  - sample the result at the clock edge.
  - output the sampled result after the clock edge.
- Hardware meaning:
  - flip-flop
  - stored value
  - one-cycle pipeline behavior

```txt
cycle N:
  in_a, in_b -> sampled

cycle N+1:
  out_state reflects sampled result
```

## Timing Model Difference

| Feature | `assign` | `always_ff` |
|---|---|---|
| Latency | 0 cycle | +1 cycle |
| Hardware | combinational logic | flip-flop |
| Memory | no | yes |
| Timing path | longer | shorter after the FF |
| Output meaning | current value | sampled value |
| Area | smaller | larger |
| Fmax | limited by combinational delay | can be higher |

## FSM Output Default

- FSM outputs are often written as combinational decode.
- Examples:

```systemverilog
assign busy  = (state == RUN);
assign ready = (state == IDLE);
```

- This means:
  - `busy` is true when the current state is `RUN`.
  - `ready` is true when the current state is `IDLE`.
- This is pure state decoding.

## When to Register FSM Output

- Register an FSM output when the output needs a clocked timing boundary.
- Common reasons:
  - cut a long timing path
  - stabilize an external interface signal
  - intentionally add a pipeline stage
  - align the output with other pipelined data

## Case 1: Cut Timing Path

- Suppose the output logic is complex:

```systemverilog
assign out_state = complex_logic(in_a, in_b, in_c, in_d);
```

- If the combinational path is too long, timing may fail.
- Registering the output cuts the timing path:

```systemverilog
always_ff @(posedge clk)
  out_state <= complex_logic(in_a, in_b, in_c, in_d);
```

- This trades one cycle of latency for a shorter timing path.
- It can improve maximum clock frequency.

## Case 2: Interface Output

- Some outputs should be stable across a whole cycle.
- Examples:
  - AXI valid signal
  - NoC packet control signal
  - handshake signal
  - external module control signal
- These are often registered.
- The goal is not style.
- The goal is stable timing at the interface boundary.

## Case 3: Pipeline FSM

- Some designs intentionally pipeline FSM outputs.
- Example:

```txt
stage 1: compute condition
stage 2: latch result
stage 3: drive output
```

- This is common in CPU, GPU, NoC, and high-frequency designs.

## Simple Rule

- Use combinational output when you need current-cycle response.
- Use registered output when you need a pipeline boundary.
- Practical guideline:
  - combinational output: `assign` or `always_comb`
  - registered output: `always_ff`
  - register only when timing, pipeline, or interface semantics require it

## Intuition

- `assign` means:
  - I output what I see now.
- `always_ff` means:
  - I remember it now and tell you next cycle.

## Not an FSM Special Case

- This code:

```systemverilog
out_state <= in_a & in_b;
```

- is not really an FSM-specific issue.
- It asks a more general RTL question:
  - should this combinational result become a pipeline stage?

## Core Tradeoff

- RTL design often balances:

| Goal | Common Method |
|---|---|
| reduce latency | use combinational path |
| improve Fmax | add flip-flops |
| reduce timing risk | add pipeline stages |
| reduce pipeline penalty | add bypass paths |

- There is no single best answer.
- The right choice depends on the timing and latency requirement.

## Why Convert assign to FF?

- The most common reason is a critical path.
- Example:

```systemverilog
assign y = a + b + c + d + e;
```

- Hardware meaning:
  - several operations happen in one cycle.
  - the combinational delay may be too long.
- If timing fails, add a pipeline boundary:

```systemverilog
always_ff @(posedge clk)
  y <= a + b + c + d + e;
```

- The tradeoff:
  - better timing
  - higher possible clock rate
  - one extra cycle of latency

## Why Convert FF Back to assign?

- Sometimes the pipeline becomes too deep.
- Too many FF stages can increase latency.
- CPU and GPU designs often add bypass paths to reduce waiting.

```systemverilog
assign out = bypass_en ? direct_path : reg_out;
```

- This makes the path more combinational.
- It may reduce latency by one or more cycles.
- But it also increases timing pressure.

## Real CPU Style

- Real high-performance RTL often has all three:
  - combinational datapath
  - registered pipeline boundary
  - bypass path

```systemverilog
assign alu_comb = a + b;

always_ff @(posedge clk)
  alu_reg <= alu_comb;

assign alu_out = bypass ? alu_comb : alu_reg;
```

- `alu_comb` gives low-latency direct result.
- `alu_reg` gives a pipeline boundary.
- `bypass` chooses whether to use current-cycle or registered data.

## Unified View

- `FF` cuts the timing graph.
- `assign` exposes the timing graph.

| Operation | Result |
|---|---|
| add FF | shorter path, more latency |
| remove FF | lower latency, more timing risk |
| add bypass | reduce pipeline penalty, increase mux/path complexity |

## Simple Pipeline Cut Example

- Before cutting:

```systemverilog
assign y = (a + b) * (c + d);
```

- Hardware path:

```txt
a + b --\
         * -> y
c + d --/
```

- One cycle must finish:
  - adder
  - adder
  - multiplier
- This can be too long.

## Pipeline Cut Step 1

- First split the logic into intermediate results:

```systemverilog
wire [31:0] ab = a + b;
wire [31:0] cd = c + d;

assign y = ab * cd;
```

## Pipeline Cut Step 2

- Insert FFs at the cut point:

```systemverilog
logic [31:0] ab_r;
logic [31:0] cd_r;

always_ff @(posedge clk) begin
  ab_r <= a + b;
  cd_r <= c + d;
end

assign y = ab_r * cd_r;
```

- Now the design has two pipeline stages.

```txt
cycle N:
  a,b,c,d -> adders -> ab_r, cd_r

cycle N+1:
  ab_r, cd_r -> multiplier -> y
```

## Pipeline Cut Effect

| Item | Before | After |
|---|---|---|
| Latency | shorter | longer |
| Combinational depth | adder + adder + multiplier | split across cycles |
| Timing closure | harder | easier |
| Max frequency | lower | higher possible |

- The design does not become faster for one item of data.
- It may become faster in throughput because the clock can run faster.

## FIFO or Memory-Style Example

- Before cutting:

```systemverilog
assign out = mem[rd_ptr] + bias;
```

- The path includes:
  - memory read
  - adder
  - output
- Cut with a register:

```systemverilog
logic [31:0] mem_r;

always_ff @(posedge clk)
  mem_r <= mem[rd_ptr];

assign out = mem_r + bias;
```

- Pipeline view:

```txt
cycle N:
  read memory -> mem_r

cycle N+1:
  mem_r + bias -> out
```

## Three-Step Pipeline Method

- Step 1:
  - find the longest combinational path.
- Step 2:
  - choose a cut point and insert FFs.
- Step 3:
  - update the design so the work happens across multiple cycles.

```txt
Before:
  A -> B -> C -> output

After:
  A -> FF -> C -> output
```

## Most Important Intuition

- A flip-flop is not only for storing data.
- In high-speed RTL, a flip-flop is also a timing cut.
- It divides one long combinational path into shorter paths.

## Further Topics Mentioned

- How to decide the best place to cut a timing path.
- Why bypass networks can become one of the most complex RTL blocks in CPU/GPU designs.
- How modern CPU designs sometimes add FFs for timing and add bypass paths to recover latency.

## Final Summary

- `assign` means zero-cycle combinational response.
- `always_ff` means registered, sampled, next-cycle behavior.
- FSM outputs are often combinational, but they can be registered for timing or interface reasons.
- Adding FFs improves timing but adds latency.
- Removing FFs or adding bypass paths can reduce latency but increases timing pressure.
- Modern CPU and GPU RTL is a constant tradeoff between timing closure and latency.
