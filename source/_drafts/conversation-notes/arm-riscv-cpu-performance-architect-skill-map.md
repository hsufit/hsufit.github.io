---
title: ARM RISC-V CPU Performance Architect Skill Map
description: 從 ARM/RISC-V 系統角度整理 CPU Performance Architect 需要具備的能力、分析方法與設計取捨。
tags:
- cpu
- microarchitecture
- performance
- architect
- arm
- riscv
- research-notes
---

## 目的

- 這篇 note 把原本偏題庫分析的討論，整理成一份通用的 CPU Performance Architect 能力地圖。
- 不綁定特定公司、產品線或招募描述。
- 目標系統假設是 ARM/RISC-V SoC。
- 重點不是背 CPU 名詞，而是回答：
  - architect 具體要會什麼？
  - 拿到 performance data 要怎麼分析？
  - 看到 IPC drop、cache miss、branch miss、model mismatch，要如何一步一步找 root cause？
  - 做 CPU subsystem 定義時，要如何把 performance、power、area、thermal 放在一起取捨？

## CPU Performance Architect 的定位

- CPU Performance Architect 的工作不是單純寫 RTL。
- 也不是只會說出 pipeline stage 或 cache hierarchy。
- 核心工作是：
  - 定義 CPU 或 CPU subsystem 的 performance target
  - 建立 performance model
  - 分析 workload 的特性
  - 找出 bottleneck
  - 預測 microarchitecture feature 的收益
  - 做 pre-silicon / post-silicon correlation
  - 和 RTL、DV、compiler、OS、firmware、physical design、power/thermal 團隊一起收斂設計

- 一句話：

```txt
CPU Designer 可能實作一個 feature。
CPU Performance Architect 要回答這個 feature 值不值得做。
```

## 能力層級

- Junior engineer 常停在：
  - cache 是什麼
  - branch predictor 是什麼
  - OoO pipeline 有哪些 stage

- Senior engineer 應該能回答：
  - 為什麼這個 workload 慢
  - bottleneck 在 frontend、backend、memory，還是 branch speculation
  - 哪些 counter 可以支持這個判斷

- Architect 要再往前一步：
  - 如果把 L2 從 2 MiB 改成 4 MiB，IPC 會提升多少？
  - latency、power、area 會增加多少？
  - 如果 branch predictor accuracy 提升 2%，值不值得增加 predictor storage？
  - 如果 model IPC 是 3.2，但 RTL IPC 是 2.6，差異要怎麼定位？
  - 如果 workload 是 memory bandwidth bound，增加 issue width 還有沒有用？

- 準備或訓練時，每個主題都要提升到這個層級：

```txt
知道原理
  -> 知道如何量測
  -> 知道如何分析
  -> 知道如何建模
  -> 知道如何做設計取捨
```

## 1. Microarchitecture

### Architect 要會什麼

- 要理解現代 OoO CPU pipeline：
  - fetch
  - decode
  - rename
  - dispatch
  - issue
  - execute
  - writeback
  - retire
- 要理解每個 stage 的限制如何影響 IPC：
  - fetch width
  - decode width
  - rename bandwidth
  - issue width
  - execution port
  - load/store bandwidth
  - ROB size
  - physical register file size
  - branch recovery latency
  - cache and TLB latency

- 不只要知道 8-wide CPU 代表什麼。
- 還要知道 8-wide 不代表 IPC 一定可以到 8。
- 真正限制可能來自：
  - workload ILP 不足
  - branch misprediction
  - I-cache miss
  - load miss
  - dependency chain
  - execution port contention
  - ROB 滿
  - load queue / store queue 滿
  - memory ordering replay

### 對應要怎麼做

- 給定一個低 IPC case，先不要直接猜原因。
- 先建立 bottleneck map：

```txt
low IPC
  -> frontend bound?
  -> bad speculation?
  -> backend core bound?
  -> backend memory bound?
  -> retire or commit limitation?
```

- 先看 high-level counter：
  - instructions retired
  - cycles
  - IPC
  - branch MPKI
  - I-cache MPKI
  - D-cache MPKI
  - L2 / LLC MPKI
  - ITLB / DTLB miss
  - frontend stall cycles
  - backend stall cycles
  - ROB full cycles
  - issue queue full cycles
  - load queue / store queue full cycles

- 再把現象映射到可能原因：

| 現象 | 可能原因 | 下一步 |
| --- | --- | --- |
| frontend stall 高 | fetch bandwidth、I-cache、ITLB、branch redirect | 看 I-cache MPKI、ITLB miss、BTB miss、taken branch density |
| branch MPKI 高 | predictor 不準、BTB/RAS/indirect target 問題 | 分 branch type、看 hot PC、做 predictor sensitivity |
| backend stall 高但 port utilization 低 | memory latency 或 dependency chain | 看 ROB occupancy、load miss、critical path |
| backend stall 高且 port utilization 高 | execution resource 不夠 | 看 ALU/FPU/vector/load/store port 使用率 |
| ROB 常滿 | long-latency load、retire 卡住、speculation 卡住 | 看 oldest instruction、load miss、exception/replay |
| LSQ 滿或 replay 多 | memory dependence、store forwarding、ordering | 做 memory microbenchmark 和 replay 分類 |

## 2. IPC Drop Mapping

### Architect 要會什麼

- IPC drop mapping 是 performance architect 很核心的能力。
- 目標是把「IPC 變低」轉換成「哪一類硬體事件造成多少 CPI 增加」。
- 不是只說：

```txt
IPC dropped, so performance is bad.
```

- 而是要能說：

```txt
IPC 從 3.0 掉到 1.2。
CPI 從 0.333 增加到 0.833。
多出來的 0.5 cycles/instruction
主要來自 L2 miss latency 和 branch recovery。
```

### 對應要怎麼做

- Step 1: 確認比較條件一致。
  - same binary
  - same input
  - same compiler flags
  - same CPU frequency
  - same DVFS / thermal state
  - same core affinity
  - same OS noise level
  - same benchmark phase

- Step 2: 把 IPC 轉成 CPI。

```txt
CPI = cycles / instructions = 1 / IPC
```

- 例如：

```txt
baseline IPC = 3.0 -> CPI = 0.333
new IPC      = 1.2 -> CPI = 0.833
extra CPI    = 0.500
```

- Step 3: 建 CPI stack。

```txt
CPI
  = base execution
  + frontend stalls
  + bad speculation penalty
  + backend core stalls
  + memory stalls
  + synchronization / OS noise
```

- Step 4: 用 top-down 分類。

```txt
Retiring
Bad Speculation
Frontend Bound
Backend Bound
```

- Step 5: 繼續往下切。

```txt
Frontend Bound
  -> I-cache miss
  -> ITLB miss
  -> branch redirect bubble
  -> decode bandwidth

Bad Speculation
  -> branch misprediction
  -> machine clear / pipeline flush

Backend Bound
  -> Core Bound
      -> execution port contention
      -> scheduler / ROB / physical register limit
      -> dependency chain
  -> Memory Bound
      -> L1D miss
      -> L2 miss
      -> LLC miss
      -> DRAM latency
      -> bandwidth saturation
      -> TLB page walk
```

- Step 6: 用實驗驗證假設。
  - 如果懷疑 branch：改成 branch-friendly microbenchmark 或看 hot branch PC。
  - 如果懷疑 cache：做 cache size sweep、working set sweep、prefetch on/off。
  - 如果懷疑 memory latency：做 pointer chasing。
  - 如果懷疑 bandwidth：做 streaming bandwidth test。
  - 如果懷疑 TLB：改 page size 或做 stride/page footprint sweep。
  - 如果懷疑 dependency chain：看 data dependency depth 或做 instruction scheduling experiment。

### IPC Drop Mapping 範例表

| IPC drop 線索 | 可能 root cause | 需要看的資料 | 可做的驗證 |
| --- | --- | --- | --- |
| branch MPKI 上升 | predictor accuracy 下降 | branch type、hot branch PC、BTB/RAS/indirect miss | branch trace、predictor size sensitivity |
| I-cache MPKI 上升 | code footprint 或 layout 問題 | I-cache miss PC、ITLB miss、binary layout | function layout、code size、LTO/PGO |
| L2 MPKI 上升 | working set 超過 L2 | miss curve、reuse distance | L2 size sensitivity、cache simulator |
| LLC miss 多但 bandwidth 不滿 | latency bound | MLP、average miss latency、ROB full | pointer chasing、增加 outstanding miss |
| bandwidth 接近上限 | bandwidth bound | DRAM bandwidth、queue occupancy | streaming test、prefetch throttling |
| port utilization 高 | execution resource bottleneck | per-port utilization、instruction mix | 改 instruction mix 或增加 execution unit model |
| ROB full 高 | long-latency operation 卡住 | oldest instruction、load miss、replay | trace oldest blocking instruction |
| DTLB miss 高 | page walk cost | page walk cycles、page size、working set | huge page、TLB reach sweep |

## 3. Branch Prediction

### Architect 要會什麼

- 不只要知道 predictor 名稱：
  - bimodal
  - gshare
  - TAGE
  - perceptron
  - indirect predictor
  - RAS
- 還要知道 branch prediction 對 performance 的量化影響。
- Branch predictor 不是越大越好。
- 它會影響：
  - area
  - power
  - frontend timing
  - access latency
  - recovery behavior

### 對應要怎麼做

- 基本估算公式：

```txt
branch penalty CPI
  = branch frequency
  * mispredict rate
  * mispredict penalty
```

- 例子：

```txt
branch frequency = 20%
mispredict rate  = 5%
penalty          = 20 cycles

branch penalty CPI = 0.20 * 0.05 * 20 = 0.20
```

- 如果 accuracy 從 95% 提升到 97%：

```txt
old penalty = 0.20 * 0.05 * 20 = 0.20 CPI
new penalty = 0.20 * 0.03 * 20 = 0.12 CPI
gain        = 0.08 CPI
```

- Architect 要能接著問：
  - 這 0.08 CPI 對目標 workload 代表多少 performance gain？
  - predictor storage 要增加多少？
  - power 增加多少？
  - timing 會不會讓 frontend cycle time 變差？
  - 哪些 branch 類型真的受益？
  - 是 direction miss、target miss、indirect miss，還是 RAS miss？

- 實作分析時要看：
  - branch frequency
  - branch MPKI
  - taken / not-taken ratio
  - conditional / indirect / return branch 分布
  - mispredict penalty
  - hot branch PC
  - predictor table aliasing
  - BTB miss
  - RAS overflow / underflow

## 4. Cache and TLB

### Architect 要會什麼

- 不只要會背 AMAT：

```txt
AMAT = hit time + miss rate * miss penalty
```

- Architect 要能回答：
  - L1D 要多大？
  - L2 要選 2 MiB 還是 4 MiB？
  - LLC 要不要加大？
  - associativity 增加是否值得？
  - line size 對 bandwidth 和 pollution 的影響是什麼？
  - latency 增加會不會抵消 miss rate 改善？
  - prefetcher 會不會造成 cache pollution？

### 對應要怎麼做

- 做 cache tradeoff 時，不只看 miss rate。
- 要看：
  - hit latency
  - miss rate
  - miss penalty
  - MLP
  - area
  - dynamic power
  - leakage power
  - timing closure risk
  - physical distance and wire delay

- 粗略估算可以從這個式子開始：

```txt
Delta CPI
  ~= access_per_inst * Delta hit_latency
   + (Delta MPKI / 1000) * effective_miss_penalty
```

- 其中：

```txt
effective_miss_penalty = raw_miss_latency / MLP
```

- 例子：

```txt
L2 size: 2 MiB -> 4 MiB
L2 MPKI: 10 -> 8
L2 hit latency: 12 cycles -> 15 cycles
L2 access per instruction: 0.30
effective miss penalty: 80 cycles
```

- Miss 改善收益：

```txt
(10 - 8) / 1000 * 80 = 0.16 CPI saved
```

- Hit latency 成本：

```txt
0.30 * (15 - 12) = 0.90 CPI cost
```

- 在這個簡化例子中，加大 L2 可能不值得。
- 實際模型要更細，因為 L2 latency 不一定完全暴露在 CPI 上，也可能被 OoO hide。
- 但 architect 必須有這種 tradeoff 直覺。

### TLB 和 Page Walk

- TLB 不是 OS 細節而已。
- TLB miss 會造成 page walk，可能帶來很高 latency。
- 要看：
  - ITLB MPKI
  - DTLB MPKI
  - page walk cycles
  - page size
  - working set
  - ASID / context switch effect

- 在 ARM/RISC-V 系統中，還要理解：
  - virtual memory translation
  - page table level
  - ASID
  - memory attribute
  - cacheability
  - privilege level
  - TLB shootdown

## 5. Memory System

### Architect 要會什麼

- 要能判斷 workload 是：
  - latency bound
  - bandwidth bound
  - MLP bound
  - prefetch bound
  - TLB bound
  - synchronization bound

- 兩個 workload 就算 L2 miss rate 一樣，performance 也可能差很多。
- 常見原因是 MLP 不同。

```txt
workload A:
  one miss blocks the critical path
  MLP low

workload B:
  many misses overlap
  MLP high
```

### 對應要怎麼做

- 要看的資料：
  - L1D / L2 / LLC MPKI
  - average memory latency
  - outstanding misses
  - MSHR occupancy
  - load queue occupancy
  - store queue occupancy
  - DRAM bandwidth
  - memory controller queue occupancy
  - row buffer hit rate
  - prefetch accuracy
  - prefetch coverage
  - prefetch timeliness
  - useless prefetch traffic

- 判斷 latency bound：
  - bandwidth 沒滿
  - ROB 常被 load miss 卡住
  - MLP 低
  - pointer chasing 類 workload 慢

- 判斷 bandwidth bound：
  - DRAM bandwidth 接近平台上限
  - memory queue 滿
  - prefetch 和 demand request 競爭頻寬
  - 增加 cache 或 MLP 不再改善

- 判斷 prefetch 是否有用：
  - coverage 高不一定好
  - accuracy 低會浪費 bandwidth
  - timeliness 差可能太早或太晚
  - cache pollution 可能讓 useful data 被趕走

## 6. Core Backend and OoO Limits

### Architect 要會什麼

- OoO 的價值是：
  - hide memory latency
  - exploit ILP
  - improve resource utilization
- 但 OoO 不是魔法。
- 無法被有效 hide 的情況包括：
  - critical dependency chain
  - serial code
  - pointer chasing
  - branch-dependent control flow
  - memory ordering restriction
  - atomic and lock-heavy code

### 對應要怎麼做

- 分析 backend 時，要把 core bound 和 memory bound 分開。
- Core bound 常見線索：
  - ALU/FPU/vector port utilization 高
  - issue queue 滿
  - scheduler wakeup/select 壓力高
  - physical register pressure 高
  - rename / dispatch stall

- Memory bound 常見線索：
  - load miss 多
  - ROB 滿但 execution port 不忙
  - LSQ 滿
  - replay 多
  - store-to-load forwarding fail
  - memory dependence violation

- 如果考慮加大 ROB 或 scheduler：
  - 要先看有沒有足夠 ILP 或 MLP 可以利用
  - 要看 wakeup/select timing 是否可收斂
  - 要看 CAM/search power 是否可接受
  - 要看 physical register file 和 bypass network 成本

## 7. Performance Analysis Workflow

### Architect 要會什麼

- 要把 performance analysis 做成可重複的流程。
- 不是每次都憑直覺猜。

### 對應要怎麼做

- 一個實用流程：

```txt
1. Define the question
2. Reproduce the workload
3. Lock down measurement conditions
4. Collect high-level counters
5. Build CPI stack
6. Classify top-down bottleneck
7. Attribute to PC / function / phase
8. Build hypotheses
9. Run microbench or sensitivity experiment
10. Decide whether it is model issue, RTL issue, software issue, or workload property
```

- 需要輸出的不是一堆 counter。
- 需要輸出的是這種結論：

```txt
Root cause:
  L2 miss latency dominates the extra CPI.

Evidence:
  L2 MPKI increased from 4 to 13.
  ROB full cycles increased by 28%.
  DRAM bandwidth is only 45% of peak, so it is latency bound.
  Hot miss PCs are in pointer-chasing code.

Next action:
  Evaluate data prefetcher improvement.
  Try larger L2 only after checking miss curve.
  Build pointer-chasing microbenchmark for model/RTL correlation.
```

## 8. Performance Modeling

### Architect 要會什麼

- Performance model 是在 RTL 還沒有完成，甚至還不存在時，用來預測設計方向的工具。
- 它不是只有一種形式。

- 常見模型：
  - analytical model
  - trace-driven model
  - execution-driven simulator
  - cycle-level model
  - hybrid model

- 不同模型適合不同階段：

| 模型 | 適合用途 | 風險 |
| --- | --- | --- |
| analytical model | 快速估算 feature ROI | 過度簡化 |
| trace-driven model | cache、branch、memory sensitivity | trace 可能缺少 timing feedback |
| cycle-level model | pipeline 和 resource contention | 建模成本高 |
| execution-driven simulator | 軟體行為更完整 | 速度慢、維護成本高 |

### 對應要怎麼做

- 建 model 時要先定義：
  - target workload
  - target metric
  - baseline CPU config
  - design knobs
  - event taxonomy
  - calibration data
  - acceptable error range

- 常見 design knobs：
  - fetch/decode/issue/retire width
  - ROB size
  - scheduler size
  - branch predictor size
  - load/store queue size
  - L1/L2/LLC size
  - cache latency
  - prefetcher aggressiveness
  - memory bandwidth
  - vector width

- 一個 model study 應該輸出：
  - baseline IPC
  - feature-on IPC
  - delta CPI breakdown
  - sensitivity curve
  - workload coverage
  - model assumptions
  - risk and unknowns

- 例子：

```txt
Question:
  Should we increase ROB from 192 to 256 entries?

Need to measure:
  ROB full cycles
  long-latency miss overlap
  branch misprediction recovery cost
  physical register pressure
  scheduler pressure

Decision:
  If workload is mostly frontend bound, larger ROB may not help.
  If workload has high MLP and frequent long-latency loads, larger ROB may help.
```

## 9. Correlation: Model vs RTL vs Silicon

### Architect 要會什麼

- Correlation 是 performance architect 的核心工作。
- Model、RTL、FPGA/emulation、silicon 之間一定會有差異。
- 重要的是能定位差異來源。

### 對應要怎麼做

- Step 1: 先確認比較條件一致。
  - same benchmark
  - same input
  - same binary
  - same compiler
  - same ISA extension setting
  - same cache/memory config
  - same warm-up policy
  - same simulation window
  - same counter definition

- Step 2: 從 IPC 開始，但不要停在 IPC。

```txt
Model IPC = 3.2
RTL IPC   = 2.6
```

- Step 3: 分解成 CPI stack。

```txt
Model extra CPI:
  frontend 0.05
  branch   0.10
  memory   0.25

RTL extra CPI:
  frontend 0.08
  branch   0.12
  memory   0.70
```

- Step 4: 定位 mismatch。

| Mismatch | 可能原因 |
| --- | --- |
| RTL branch penalty 比 model 高 | recovery path 建模太樂觀、frontend refill latency 被低估 |
| RTL L2 latency 比 model 高 | interconnect、bank conflict、physical timing、queueing 沒建進去 |
| RTL rename stall 多 | model 沒有 physical register 或 rename bandwidth 限制 |
| RTL LSQ full 多 | model 沒有 store/load ordering、replay、MSHR 限制 |
| RTL IPC phase 差異大 | simulation window 沒對齊或 workload phase 不同 |

- Step 5: 建 microbenchmark 隔離。
  - branch microbenchmark
  - I-cache footprint microbenchmark
  - pointer chasing
  - streaming bandwidth
  - load/store forwarding test
  - atomic/lock contention test

- Step 6: 判斷是哪一種問題：
  - model assumption wrong
  - RTL implementation bug
  - counter definition mismatch
  - workload setup mismatch
  - real hardware limitation
  - software/compiler behavior different

## 10. Workload Characterization

### Architect 要會什麼

- 不只是會跑 benchmark。
- 要能把 workload 變成 profile。

- 一個有用的 workload profile 應包含：
  - IPC / CPI
  - instruction mix
  - load/store ratio
  - branch frequency
  - branch MPKI
  - I-cache / D-cache / L2 / LLC MPKI
  - ITLB / DTLB MPKI
  - MLP
  - memory bandwidth
  - working set size
  - vector utilization
  - atomic / lock frequency
  - kernel time vs user time
  - phase behavior

### 對應要怎麼做

- 先做分類：

| Workload 類型 | 常見特徵 | 常見瓶頸 |
| --- | --- | --- |
| compute bound | port utilization 高、cache miss 低 | execution unit、dependency chain |
| memory latency bound | bandwidth 不滿、ROB 被 load miss 卡住 | DRAM latency、MLP 低 |
| memory bandwidth bound | bandwidth 接近上限 | DRAM/channel/interconnect bandwidth |
| branch bound | branch MPKI 高 | predictor、BTB、frontend recovery |
| frontend bound | I-cache/ITLB miss、decode 壓力 | code footprint、instruction fetch |
| synchronization bound | atomic/lock 多 | coherence、memory ordering、OS scheduling |
| vector bound | vector utilization 高 | vector width、load/store bandwidth、alignment |

- 再做 hot spot attribution：
  - 找 hot function
  - 找 hot loop
  - 找 hot PC
  - 分 workload phase
  - 把 counter 對到 code region

- 對 ARM/RISC-V 系統，還要看：
  - 是否使用 vector extension
  - compiler 是否產生預期的 instruction
  - atomic 是否造成 coherence 壓力
  - page size 是否影響 TLB
  - Linux scheduler 是否造成 migration
  - driver/runtime 是否進入 kernel hot path

## 11. CPU Subsystem Definition

### Architect 要會什麼

- CPU subsystem definition 是 architect 真正的產出之一。
- 需要決定：
  - core count
  - cluster topology
  - private L2 size
  - shared LLC size
  - coherence fabric
  - interconnect bandwidth
  - memory bandwidth target
  - prefetcher policy
  - branch predictor class
  - PMU event definition
  - power state and DVFS behavior
  - security and isolation feature interaction

### 對應要怎麼做

- 每個決策都要能用資料支撐。
- 例如要選 4 MiB LLC 還是 8 MiB LLC，不應只說「8 MiB 比較快」。
- 應該整理成：

```txt
Option A: 4 MiB LLC
  performance: baseline
  area: lower
  leakage: lower
  workload risk: memory-footprint-heavy workload loses performance

Option B: 8 MiB LLC
  performance: +X% on workload group A
  area: +Y%
  leakage: +Z%
  latency: +N cycles or possible timing risk
  workload risk: little gain on streaming or branch-bound workloads
```

- 最後給出 decision：

```txt
Choose 8 MiB only if:
  target workloads show enough LLC miss reduction
  latency increase is hidden or small
  area/power budget accepts the cost
  thermal impact does not reduce sustained frequency
```

## 12. ARM/RISC-V ISA and System Features

### Architect 要會什麼

- ISA 不是只背 instruction syntax。
- Architect 要知道 ISA feature 為什麼存在，以及它對 performance model 的影響。

- ARM 方向要熟：
  - AArch64 execution model
  - NEON
  - SVE / SVE2
  - load/store pair
  - prefetch hint
  - atomics
  - acquire/release semantics
  - memory barrier
  - exception level
  - page table and memory attribute

- RISC-V 方向要熟：
  - RV64GC baseline
  - compressed instruction extension
  - atomic extension
  - vector extension
  - bit manipulation extension
  - custom extension tradeoff
  - RVWMO memory model
  - fence / acquire-release
  - Sv39 / Sv48 virtual memory
  - PMP and privilege model

### 對應要怎麼做

- 評估 ISA feature 時，不是問「支不支援」而已。
- 要問：
  - dynamic instruction count 是否下降？
  - vector utilization 是否提高？
  - memory bandwidth 是否變成新的瓶頸？
  - compiler 是否能穩定產生該 feature？
  - runtime / OS 是否需要改？
  - verification cost 是否可接受？
  - backward compatibility 如何處理？

- 例子：

```txt
Feature:
  RISC-V vector extension or ARM SVE2

Possible gain:
  lower instruction count
  better data-level parallelism

Possible new bottleneck:
  load/store bandwidth
  cache bandwidth
  register file power
  vector pipeline occupancy
  compiler auto-vectorization quality
```

## 13. Benchmarks and Real Applications

### Architect 要會什麼

- 要知道 benchmark 的用途，也要知道 benchmark 的限制。
- 常見 benchmark 類型：
  - SPEC CPU
  - CoreMark
  - Geekbench
  - MLPerf
  - browser / JavaScript benchmark
  - internal microbenchmark
  - real application trace

- Benchmark 提升不代表 real app 一定提升。
- 可能原因：
  - workload mix 不同
  - dataset 不同
  - phase behavior 不同
  - compiler path 不同
  - OS/runtime overhead 不同
  - thermal throttling
  - memory footprint 不同
  - benchmark 只打到某個 feature

### 對應要怎麼做

- 評估 CPU 時要建立 benchmark matrix：

```txt
single-thread latency workload
multi-thread throughput workload
branch-heavy workload
memory-latency workload
memory-bandwidth workload
vector / ML workload
OS / driver / runtime-sensitive workload
```

- 每個 benchmark 都要搭配 counter profile。
- 不要只看 score。
- 要能回答：

```txt
這個分數提升是來自 instruction count 下降？
還是 IPC 上升？
還是 frequency 上升？
還是 memory stalls 下降？
還是 benchmark phase 剛好改變？
```

## 14. Coding and Tooling

### Architect 要會什麼

- 不一定要像純 SWE 一樣考很難的 algorithm。
- 但一定要能用程式處理 performance data。

- Python 常見用途：
  - parse PMU counter
  - parse trace
  - build CPI stack
  - plot sensitivity curve
  - compare model vs RTL result
  - identify outlier workload

- C/C++ 常見用途：
  - 寫 microbenchmark
  - 寫簡單 cache simulator
  - 寫 branch trace analyzer
  - 建立簡化 performance model
  - 控制 memory layout 和 access pattern

### 對應要怎麼做

- 可以練這些小工具：
  - IPC/CPI 報表產生器
  - MPKI summary script
  - CPI stack visualizer
  - cache miss curve simulator
  - branch predictor toy simulator
  - pointer-chasing benchmark
  - streaming bandwidth benchmark
  - false-sharing benchmark
  - atomic/lock contention benchmark

## 15. Architect 應該能交付的結果

- 一個 CPU Performance Architect 最後要能交付的不只是口頭分析。
- 應該能產出：
  - workload characterization report
  - performance model
  - model assumption document
  - bottleneck analysis
  - CPI stack
  - model vs RTL correlation report
  - pre-silicon bug triage
  - post-silicon performance triage
  - CPU subsystem architecture specification
  - feature ROI analysis
  - benchmark methodology

- 好的結論長這樣：

```txt
The IPC drop is mainly memory-latency bound.
The extra 0.42 CPI comes from L2/LLC miss latency.
Branch MPKI is unchanged.
Frontend stalls are not dominant.
DRAM bandwidth is below saturation, so this is not bandwidth bound.
The hot PCs are pointer-chasing loads in phase 3.
Increasing issue width is unlikely to help.
Useful next options are prefetcher tuning, data layout change, or larger effective cache capacity.
```

## 最後總結

- CPU Performance Architect 的重點是從資料做決策。
- 不是只知道 pipeline、cache、branch predictor。
- 而是要能把這些概念連到：
  - measurement
  - bottleneck mapping
  - modeling
  - correlation
  - feature ROI
  - subsystem specification

- 面對 ARM/RISC-V 系統時，architect 還要能把 ISA、compiler、Linux、driver、runtime、memory model、cache coherency、power/thermal 一起納入考量。
- 最核心的能力可以濃縮成一句話：

```txt
看到 performance 現象，能量化原因；
看到設計選項，能估算收益與成本；
看到 model 和 RTL 不一致，能定位 mismatch。
```
