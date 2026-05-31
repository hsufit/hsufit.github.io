---
title: Chisel FIRRTL CIRCT firtool and BOOM Flow Notes
description: Notes about emitCHIRRTL, Chisel elaboration, FIRRTL/CIRCT lowering, firtool, split Verilog, pass pipelines, and how BOOM is built through Chipyard.
tags:
- chisel
- firrtl
- circt
- firtool
- boom
---

## Question

- I wanted to understand what `emitCHIRRTL` does in Chisel.
- I also wanted to understand each compile/emit step:

```txt
Chisel
  |
  v
CHIRRTL
  |
  v
Low FIRRTL
  |
  v
Verilog / SystemVerilog
```

- Then I wanted the tool names and commands for each step.
- Finally, I wanted to know how BOOM does this flow.
- Related BOOM repository:
  - https://github.com/riscv-boom/riscv-boom

## Short Answer

- `emitCHIRRTL` converts a Chisel hardware design into CHIRRTL text.
- CHIRRTL is a high-level FIRRTL representation.
- The modern practical flow is often:

```txt
Scala / Chisel elaboration
    |
    v
FIRRTL / CHIRRTL (.fir)
    |
    v
firtool
    |
    v
SystemVerilog (.sv)
```

- `firtool` hides many internal lowering steps.
- BOOM usually uses Chipyard to run this kind of flow.
- BOOM is not usually built as a standalone single-module `firtool` project.

## What Is CHIRRTL?

- CHIRRTL is a high-level FIRRTL form produced from Chisel.
- It is closer to the original Chisel semantics than final Verilog.
- It may still preserve high-level constructs such as:
  - `Mem`
  - `Vec`
  - aggregate types
  - `Bundle`
  - high-level connects
  - width inference
  - `when` structure
  - `printf`
  - assertions
  - invalid values

- It is not fully lowered RTL yet.

```txt
Chisel
  |
  v
CHIRRTL
  |
  v
FIRRTL lowering
  |
  v
Low-level RTL IR
  |
  v
SystemVerilog
```

## What emitCHIRRTL Does

- `emitCHIRRTL` runs Chisel elaboration and emits CHIRRTL text.
- It is useful for:
  - debugging generated hardware
  - understanding Chisel lowering
  - checking how `Vec`, `Bundle`, `Mem`, and connects are represented
  - writing or debugging FIRRTL/CIRCT passes

## Simple Chisel Example

```scala
import chisel3._
import circt.stage.ChiselStage

class MyModule extends Module {
  val io = IO(new Bundle {
    val a = Input(UInt(8.W))
    val b = Output(UInt(8.W))
  })

  io.b := io.a + 1.U
}

object Main extends App {
  println(
    ChiselStage.emitCHIRRTL(new MyModule)
  )
}
```

- Example CHIRRTL-like output:

```firrtl
circuit MyModule :
  module MyModule :
    input clock : Clock
    input reset : UInt<1>
    input io_a : UInt<8>
    output io_b : UInt<8>

    node _T = add(io_a, UInt<1>("h1"))
    io_b <= _T
```

- This is not Verilog.
- It is an intermediate representation.

## emitCHIRRTL vs emitSystemVerilog

| API | Main output |
|---|---|
| `emitCHIRRTL` | high-level CHIRRTL / FIRRTL text |
| `emitFIRRTLDialect` | FIRRTL dialect IR |
| `emitSystemVerilog` | SystemVerilog |
| `emitVerilog` | Verilog/SystemVerilog-style RTL, depending on version/flow |

- In practice, RTL designers often care most about:
  - generated `.fir`
  - generated `.sv`
  - naming
  - memory lowering
  - synthesis/lint result

## Important Concept: Elaboration

- Chisel is not parsed like Verilog.
- Chisel is Scala code that runs on the JVM.
- When the Scala program runs, Chisel constructs a hardware graph.
- This step is called elaboration.

```txt
Scala program executes
    |
    v
Chisel objects are created
    |
    v
hardware graph is built
    |
    v
FIRRTL / CHIRRTL is emitted
```

- A Scala loop is executed during elaboration.
- Example:

```scala
for (i <- 0 until 8) {
  // generate hardware
}
```

- This does not become a hardware loop.
- It generates eight pieces of hardware.

## Chisel Is More Like an IR Builder

- Many people say "Chisel compiler".
- A more precise mental model:

```txt
Scala elaborator
    +
hardware IR generator
    +
FIRRTL/CIRCT compiler backend
```

- Chisel creates the hardware IR.
- FIRRTL/CIRCT performs the main lowering work.

## Step 1: Scala to Chisel Hardware Graph

- Tools:
  - Scala compiler
  - JVM
  - `sbt`
  - `mill`
  - `scala-cli`

- Common commands:

```sh
sbt run
sbt "runMain mypackage.Main"
```

- What happens:
  - Scala code executes.
  - `new MyModule` constructs the module.
  - Chisel records modules, wires, registers, memories, nodes, and connections.

## Step 2: Chisel to CHIRRTL

- API:

```scala
import circt.stage.ChiselStage

ChiselStage.emitCHIRRTL(new MyModule)
```

- Older code may use:

```scala
import chisel3.stage.ChiselStage

(new ChiselStage).emitCHIRRTL(new MyModule)
```

- The modern ecosystem is moving toward `circt.stage.ChiselStage`.
- Exact API style depends on the Chisel version.

## Emit CHIRRTL to a File

- A build can emit a `.fir` file.
- Example shape:

```scala
import circt.stage.ChiselStage
import chisel3.stage.ChiselGeneratorAnnotation

(new ChiselStage).execute(
  Array("--target", "chirrtl"),
  Seq(ChiselGeneratorAnnotation(() => new MyModule))
)
```

- Output is usually something like:

```txt
MyModule.fir
```

## Step 3: FIRRTL and CIRCT Lowering

- The `.fir` file is processed by `firtool`.
- `firtool` comes from CIRCT.
- CIRCT is based on LLVM/MLIR infrastructure.

- Internally, the flow is roughly:

```txt
CHIRRTL / FIRRTL text
    |
    v
CIRCT FIRRTL dialect
    |
    v
HW / Comb / Seq dialects
    |
    v
SV dialect
    |
    v
SystemVerilog text
```

- Most users do not manually handle each dialect.
- `firtool` wraps the lowering pipeline.

## FIRRTL Levels

- The older mental model is:

```txt
CHIRRTL
  |
  v
High FIRRTL
  |
  v
Middle FIRRTL
  |
  v
Low FIRRTL
```

- High FIRRTL may still preserve:
  - `Bundle`
  - `Vec`
  - memories
  - abstract connects
- Middle FIRRTL starts lowering:
  - aggregate access
  - width inference
  - connect expansion
- Low FIRRTL is closer to flat RTL:
  - explicit widths
  - fewer high-level types
  - less implicit inference

## Typical Lowering Passes

- Compiler lowering is not one big step.
- It is many passes.
- Example pass ideas:

| Pass idea | Purpose |
|---|---|
| `InferWidths` | infer bit widths |
| `ExpandWhens` | lower `when` structure |
| `RemoveAccesses` | flatten aggregate access |
| constant propagation | simplify constants |
| dead code elimination | remove unused logic |
| memory lowering | rewrite memory constructs |
| common subexpression elimination | share repeated logic |
| Verilog export | emit final RTL text |

- This is similar in spirit to LLVM compiler passes.

## Step 4: FIRRTL to SystemVerilog

- Basic `firtool` command:

```sh
firtool MyModule.fir -o MyModule.sv
```

- This produces a SystemVerilog output file.
- For small designs, one output file is simple.
- For large designs, one huge file is painful.

## split-verilog

- Without `--split-verilog`:

```sh
firtool input.fir -o out.sv
```

- The output may put many modules into one file:

```txt
out.sv
```

- With `--split-verilog`:

```sh
firtool input.fir --split-verilog -o generated/
```

- The output directory may contain one file per module:

```txt
generated/
|-- Top.sv
|-- ALU.sv
|-- Decoder.sv
`-- RegisterFile.sv
```

## Why split-verilog Matters

- Large hardware designs may have hundreds of modules.
- Splitting output helps:
  - git diff
  - review
  - simulator input lists
  - synthesis flows
  - project organization
- This is common in practical RTL flows.

## Useful firtool Commands

- Basic output:

```sh
firtool Top.fir -o Top.sv
```

- Split generated SystemVerilog:

```sh
firtool Top.fir --split-verilog -o build/
```

- Preserve more names for debug:

```sh
firtool Top.fir --preserve-values=all -o Top.sv
```

- Show pass execution:

```sh
firtool Top.fir --verbose-pass-executions
```

- Print IR after each pass:

```sh
firtool Top.fir --mlir-print-ir-after-all
```

- Warning:
  - firtool flags can change across versions.
  - always check the version-specific help:

```sh
firtool --help
```

## What Is --verbose-pass-executions?

- `--verbose-pass-executions` prints the compiler pass execution process.
- It is like a compiler construction log.
- It can show:
  - pass names
  - pass order
  - when a pass starts
  - when a pass ends

- Example idea:

```txt
Running pass: LowerCHIRRTLPass
Running pass: InferWidthsPass
Running pass: CanonicalizerPass
Running pass: CSEPass
Running pass: ExportVerilogPass
```

## Who Uses Pass Pipeline Debug?

- Usually:
  - compiler engineers
  - FIRRTL/CIRCT developers
  - EDA flow developers
  - framework developers
  - advanced RTL flow debugging

- Normal RTL designers usually do not need to read every pass.
- They care more about:
  - generated `.fir`
  - generated `.sv`
  - naming
  - lint
  - synthesis
  - timing

## RTL Designer Minimal Flow

- The simplified flow is:

```txt
1. ChiselStage.emitCHIRRTL -> Top.fir
2. firtool Top.fir --split-verilog -o output_dir/
```

- This is valid and common.
- A real project may wrap these steps inside `sbt`, `mill`, Makefiles, or a larger SoC build system.

## More Precise Modern View

- The two visible steps hide more internal work:

```txt
ChiselStage.emitCHIRRTL
    |
    v
Top.fir
    |
    v
firtool
    |
    +--> parse FIRRTL
    +--> lower FIRRTL dialect
    +--> optimize
    +--> lower to HW/Comb/Seq dialects
    +--> lower to SV dialect
    +--> emit SystemVerilog
```

- So the short workflow is correct.
- It just hides the MLIR/CIRCT backend details.

## Common Misunderstandings

- Misunderstanding:
  - FIRRTL to MLIR is always a manual step.
- Better view:
  - `firtool` usually handles this internally.

- Misunderstanding:
  - `emitCHIRRTL` directly emits MLIR.
- Better view:
  - `emitCHIRRTL` emits CHIRRTL/FIRRTL text.
  - `firtool` ingests that and uses MLIR internally.

- Misunderstanding:
  - `emitCHIRRTL` is the normal final RTL output.
- Better view:
  - it is mainly for debug or compiler flow input.
  - final RTL users usually want `.sv`.

## Production Flow Shape

- A practical project flow often looks like:

```txt
Chisel source
    |
    v
sbt / mill elaboration
    |
    v
.fir
    |
    v
firtool --split-verilog
    |
    v
generated RTL directory
    |
    v
Verilator / VCS / Xcelium / FPGA tools / synthesis
```

## Common Backend Tools

| Tool | Use |
|---|---|
| Verilator | open-source simulation |
| Synopsys VCS | commercial ASIC simulation |
| Cadence Xcelium | commercial ASIC simulation |
| AMD Vivado | FPGA synthesis/implementation |
| Intel Quartus | FPGA synthesis/implementation |

## BOOM Flow

- BOOM means Berkeley Out-of-Order Machine.
- BOOM is written in Chisel.
- BOOM is a parameterized RISC-V out-of-order core generator.
- BOOM is not best understood as one fixed RTL file.
- It is a hardware generator that creates a family of cores.

## Important BOOM Point

- The BOOM repository says it is not a self-running repository.
- To instantiate a BOOM core, use Chipyard.
- This is important because BOOM depends on:
  - Rocket Chip
  - Diplomacy
  - TileLink
  - Chipyard build infrastructure
  - SoC configuration

## BOOM Build Shape

- Typical BOOM build flow:

```txt
BOOM Scala / Chisel source
    |
    v
Chipyard configuration
    |
    v
Chisel elaboration
    |
    v
FIRRTL / CHIRRTL
    |
    v
FIRRTL/CIRCT/firtool backend
    |
    v
Verilog / SystemVerilog
    |
    v
Verilator / VCS / FPGA / FireSim / ASIC flow
```

## BOOM Through Chipyard

- Example shape from the BOOM/Chipyard flow:

```sh
git clone https://github.com/ucb-bar/chipyard.git
cd chipyard
./scripts/init-submodules-no-riscv-tools.sh
cd sims/verilator
make CONFIG=LargeBoomConfig
```

- The exact command can change with Chipyard versions.
- The important idea:
  - Chipyard elaborates BOOM as part of a complete SoC generator.
  - The generated FIRRTL/SV is part of the Chipyard build output.

## Why BOOM Is Not Just firtool input.fir

- BOOM is not usually used like:

```sh
firtool BoomCore.fir --split-verilog -o out/
```

- Reasons:
  - BOOM is part of a larger SoC generator.
  - Rocket Chip and Diplomacy build the system graph.
  - TileLink and memory system components are configured together.
  - The top-level design depends on a selected Chipyard config.
  - The backend compiler step is usually hidden inside the build system.

## BOOM Mental Model

- For a small Chisel module:

```txt
MyModule.scala -> MyModule.fir -> firtool -> MyModule.sv
```

- For BOOM:

```txt
Chipyard config
    |
    v
BOOM + Rocket Chip + Diplomacy + TileLink
    |
    v
SoC FIRRTL
    |
    v
generated RTL / simulator build
```

- BOOM is a generator inside a SoC generator.
- That is why the build feels different from a small Chisel example.

## RTL Designer Takeaway

- For a small project, this is a reasonable minimal flow:

```txt
emitCHIRRTL -> .fir
firtool .fir --split-verilog -o out/
```

- For BOOM, use Chipyard.
- In BOOM, `firtool` is usually a backend step controlled by the build system.
- The designer usually interacts with:
  - configs
  - generated source directories
  - simulator make targets
  - waveform/debug output
  - final Verilog/SystemVerilog

## Further Topics Mentioned

- How the firtool pass pipeline lowers FIRRTL dialect into HW/SV dialects.
- How Rocket Chip, Diplomacy, and TileLink influence the FIRRTL graph.
- Why BOOM's generated hardware depends heavily on configuration.
- How to inspect generated BOOM FIRRTL or generated SystemVerilog.

## References

- BOOM repository:
  - https://github.com/riscv-boom/riscv-boom
- BOOM documentation:
  - https://docs.boom-core.org/
- BOOM overview:
  - https://docs.boom-core.org/en/latest/sections/intro-overview/boom.html
- BOOM development ecosystem:
  - https://docs.boom-core.org/en/latest/sections/boom-ecosystem.html
- Chipyard BOOM getting-started reference:
  - https://docs.boom-core.org/
- Chipyard repository:
  - https://github.com/ucb-bar/chipyard
- Chipyard heterogeneous SoCs / BOOM config fragments:
  - https://chipyard.readthedocs.io/en/latest/Customization/Heterogeneous-SoCs.html
- Chisel testing docs with `emitCHIRRTL` examples:
  - https://www.chisel-lang.org/docs/explanations/testing
- Chisel `circt.stage` API:
  - https://www.chisel-lang.org/api/latest/circt/stage/index.html
- CIRCT FIRRTL dialect:
  - https://circt.llvm.org/docs/Dialects/FIRRTL/
- CIRCT FIRRTL dialect rationale:
  - https://circt.llvm.org/docs/Dialects/FIRRTL/RationaleFIRRTL/
- CIRCT dialect list:
  - https://circt.llvm.org/docs/Dialects/

## Final Summary

- `emitCHIRRTL` emits high-level FIRRTL/CHIRRTL, not final Verilog.
- Chisel first elaborates Scala code into a hardware graph.
- `firtool` lowers FIRRTL through CIRCT/MLIR internals and emits SystemVerilog.
- `--split-verilog` makes large generated RTL easier to manage.
- `--verbose-pass-executions` is mainly for compiler/debug work.
- RTL designers usually use a simple `.fir -> firtool -> .sv` flow.
- BOOM uses the same broad compiler ideas, but it is built through Chipyard as a configurable SoC generator.
