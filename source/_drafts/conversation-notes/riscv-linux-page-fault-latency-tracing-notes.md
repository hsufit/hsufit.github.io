---
title: RISC-V Linux Page Fault Latency Tracing Notes
description: Notes about measuring RISC-V Linux userspace page fault latency, kprobe noise, interrupts, context switches, tracepoints, and fentry.
tags:
- linux
- riscv
- page-fault
- tracing
- ebpf
---

## Question

- I wanted to measure userspace page fault latency.
- The path I cared about was roughly:

```txt
userspace page fault
    |
    v
CPU trap into kernel
    |
    v
exception entry path
    |
    v
handle_page_fault() / do_page_fault()
```

- The important concern was:
  - can interrupt or scheduler activity happen in the middle?
  - does `bpftrace` or `kprobe` pollute the measurement?
  - how should this be done on RISC-V?
  - is there a method that does not require patching the kernel?

## Main Warning

- Yes, this kind of measurement can be polluted.
- The measured latency may include:
  - page fault handling
  - interrupt handling
  - preemption
  - scheduler context switch
  - softirq
  - NMI-like events, depending on architecture
  - tracing overhead
- With `bpftrace` and `kprobe`, the result is usually:

```txt
fault path + system noise + tracing overhead
```

- It is not pure fault-handler execution time.

## What You May Actually Measure

- There are two different meanings of latency.

| Measurement Target | Meaning |
|---|---|
| wall-clock latency | real elapsed time seen by the system |
| pure CPU execution time | time spent actually executing the fault path |

- `ktime_get_ns()` or `nsecs` usually gives wall-clock latency.
- Wall-clock latency includes interrupts and descheduling.
- Pure CPU execution time is much harder to measure.

## RISC-V Page Fault Basics

- On RISC-V, a page fault is a hardware exception.
- It is not an `ecall`.
- `ecall` is used for environment calls, like syscall-style transitions.
- Page fault is raised automatically by the CPU when address translation or permission checking fails.
- The RISC-V privileged architecture reference at the end is the source for trap CSRs such as `scause`, `stval`, `sepc`, and `stvec`.

## RISC-V Page Fault Causes

- RISC-V records the exception cause in `scause`.
- Common page fault causes:

| Cause | Meaning |
|---|---|
| `12` | instruction page fault |
| `13` | load page fault |
| `15` | store/AMO page fault |

- The CPU also writes:
  - `stval`: faulting virtual address
  - `sepc`: faulting program counter
- Then the CPU switches into S-mode and jumps to `stvec`.

## RISC-V Linux Trap Flow

- The rough Linux flow is:

```txt
userspace
    |
    | page fault exception
    v
hardware trap
    |
    v
stvec
    |
    v
arch/riscv/kernel/entry.S
    |
    v
handle_exception
    |
    v
do_page_fault
    |
    v
handle_mm_fault
```

- `handle_exception` is the low-level trap entry.
- `do_page_fault` is the RISC-V Linux page fault handling path.
- `handle_mm_fault` is the generic memory-management fault handler.

## Can Interrupts Happen in the Middle?

- Yes.
- Early trap entry usually runs with interrupts disabled.
- Later, Linux may enable interrupts.
- After interrupts are enabled, the fault path may be interrupted by:
  - timer interrupt
  - external interrupt
  - IPI
  - scheduler tick
  - device interrupt
- This can inflate measured latency.

## Can Context Switch Happen?

- Yes, depending on the type of fault.
- Minor faults usually do not sleep.
- Examples of minor faults:
  - anonymous page allocation
  - zero page mapping
  - page table population
- Even minor faults may still be interrupted or preempted.
- Major faults may sleep.
- Example:
  - page data must be read from disk or storage
- A major fault can call into paths that eventually schedule another task.
- In that case, measured latency includes time while the task is not running.

## Why kprobe Adds Noise on RISC-V

- On RISC-V, `kprobe` commonly works by patching an instruction with `ebreak`.
- `ebreak` is the RISC-V breakpoint instruction.
- Example before kprobe:

```asm
addi sp, sp, -16
```

- Example after kprobe patching:

```asm
ebreak
```

- When the CPU executes `ebreak`, it raises a breakpoint exception.
- Linux handles that exception and runs the kprobe handler.

## kprobe Is Another Trap

- A page fault is already a trap.
- A kprobe on the page fault path inserts another trap.
- The timeline can become:

```txt
userspace store
    |
    v
hardware page fault
    |
    v
trap entry
    |
    v
ebreak for kprobe
    |
    v
kprobe breakpoint exception
    |
    v
eBPF handler
    |
    v
return to original fault path
```

- This pollutes the measurement.
- It is especially bad for sub-microsecond latency measurement.

## bpftrace Overhead

- `bpftrace` is convenient, but it is not free.
- Possible overhead sources:
  - kprobe breakpoint trap
  - BPF program execution
  - map update
  - timestamp helper
  - ring buffer output
  - printf-style output
- This makes `bpftrace` better for observability than cycle-accurate measurement.

## Tracepoint

- A tracepoint is a static tracing hook built into the Linux kernel.
- The kernel source contains predefined trace calls.
- Examples:
  - `exceptions:page_fault_user`
  - `exceptions:page_fault_kernel`
  - `sched:sched_switch`
  - syscall tracepoints

- Tracepoints are not dynamic instruction patching in the same way as kprobes.
- They do not insert an `ebreak` trap into the measured instruction stream.
- They are usually more stable and lower overhead than kprobes.
- The Linux tracepoint and bpftrace references at the end are the links to check for tracing syntax and concepts.

## kprobe vs Tracepoint

| Method | Mechanism | Measurement Risk |
|---|---|---|
| `kprobe` | patch instruction with `ebreak` | high overhead and extra trap |
| tracepoint | static kernel hook | lower overhead |
| fentry/fexit | function entry/exit instrumentation | often lower overhead than kprobe |
| kernel patch | custom instrumentation | most precise but requires source changes |

- For RISC-V page-fault latency, kprobe is usually not ideal.
- Tracepoints are better when available.
- fentry is better if the kernel and architecture support it.

## How to List Tracepoints

- Use `bpftrace` to list available tracepoints:

```sh
sudo bpftrace -l 'tracepoint:*page*'
sudo bpftrace -l 'tracepoint:exceptions:*'
```

- Possible results may include:

```txt
tracepoint:exceptions:page_fault_user
tracepoint:exceptions:page_fault_kernel
```

- Exact names depend on kernel version and configuration.

## Recommended No-Kernel-Patch Method

- If kernel patching is not allowed, use:
  - tracepoint + eBPF
  - fentry/fexit if supported
- Avoid kprobe when measuring tiny trap-path latency.

## Example bpftrace Shape

- A possible starting point:

```sh
sudo bpftrace -e '
tracepoint:exceptions:page_fault_user
{
    @start[tid] = nsecs;
}

kretprobe:handle_mm_fault
/@start[tid]/
{
    printf("%d ns\n", nsecs - @start[tid]);
    delete(@start[tid]);
}
'
```

- This still uses a kretprobe at the end.
- It reduces kprobe pollution at the start point.
- A full tracepoint-only or fentry-based version is better if available.

## What Tracepoint Can and Cannot Measure

- Tracepoint can measure Linux-visible page fault handling latency.
- It does not measure from the very first instruction of trap entry.
- It is already after some low-level entry work.

- If the target is:

```txt
hardware trap entry -> first instruction of handle_exception
```

- Then tracepoint is not early enough.
- Without kernel patching, this is very hard to measure accurately.

## fentry and fexit

- `fentry` and `fexit` attach to function entry and exit.
- They are usually lower overhead than kprobes.
- They do not rely on the same `ebreak` breakpoint trap model.
- On RISC-V, availability depends on:
  - kernel version
  - compiler options
  - BPF/ftrace support
  - architecture support
- If supported, `fentry` on `do_page_fault` is preferable to `kprobe:do_page_fault`.
- The bpftrace and fprobe references below are useful when checking whether a kernel supports these lower-overhead hooks.

## RISC-V cycle Counter

- RISC-V has a cycle counter CSR.
- It can be read with:

```asm
csrr a0, cycle
```

- In C-style kernel code, this may appear as something like:

```c
rdcycle()
```

- `cycle` is lower overhead than a wall-clock timestamp.
- But using it in the earliest trap path usually requires kernel changes.

## Low-Noise Setup

- For better measurement quality:
  - pin the workload to one hart
  - use `taskset`
  - isolate the CPU
  - configure IRQ affinity
  - reduce scheduler tick with `nohz_full`
  - use fixed CPU frequency
  - warm caches
  - avoid printing inside the hot path
  - disable SMT if the platform has SMT

- Example direction:

```sh
taskset -c 1 ./fault_test
```

- Kernel boot options may include:
  - `isolcpus=`
  - `nohz_full=`

## Research-Grade Method

- For paper-level micro-latency measurement, researchers often patch the kernel.
- A more precise setup may:
  - add direct `rdcycle` instrumentation in `entry.S`
  - add another timestamp near `do_page_fault`
  - run on an isolated hart
  - disable unnecessary interrupts
  - fix CPU frequency
  - avoid tracing frameworks in the hot path

- This is more invasive but much cleaner.

## Practical Recommendation

- If the goal is practical observability:
  - use tracepoints
  - accept that interrupt and scheduler noise are real system behavior
- If the goal is pure trap-path cycles:
  - avoid kprobe
  - avoid `bpftrace` printing
  - prefer fentry if available
  - patch the kernel if accuracy really matters

## References

- RISC-V privileged architecture manual:
  - https://riscv.github.io/riscv-isa-manual/snapshot/privileged
- Linux tracepoints documentation:
  - https://www.kernel.org/doc/html/latest/trace/tracepoints.html
- Linux fprobe/fentry-style tracing documentation:
  - https://www.kernel.org/doc/html/latest/trace/fprobetrace.html
- bpftrace language documentation:
  - https://github.com/bpftrace/bpftrace/blob/master/docs/language.md

## Final Summary

- On RISC-V, page fault is a hardware exception, not an `ecall`.
- Linux enters through `stvec` and the RISC-V trap entry path.
- Interrupts can happen inside the fault path after Linux enables them.
- Major page faults can sleep and cause context switches.
- `kprobe` on RISC-V often uses `ebreak`, which adds another exception.
- This means kprobe can heavily pollute latency measurement.
- Tracepoints are better because they are static kernel hooks.
- For no-kernel-patch measurement, start with `tracepoint:exceptions:page_fault_user`.
- For lower noise, use fentry/fexit if supported.
- For cycle-accurate research, custom kernel instrumentation is still the cleanest path.
