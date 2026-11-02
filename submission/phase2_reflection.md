# Phase 2 — Reflection: where does parallelism stop helping?

Copy this file to `submission/phase2_reflection.md`, fill in every answer slot, and commit it:

```bash
cp phase-2-tiny-gpt/reflection_template.md submission/phase2_reflection.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or your threading plot freely.

Numbers in section 1 must **match `submission/phase2_report.json`** — copy them across, do not
retype from memory; they are checked against it.

---

## 1. Your measurements

| | Value |
|---|---|
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->240.783<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->0.951<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->429414.8<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->12654.0<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.4865<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
1 thread: 2360.311 ms
2 threads: 2257.346 ms
4 threads: 2269.802 ms
8 threads: 2208.607 ms

Fastest: 8 threads (2208.607 ms per step).
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
Both attention implementations perform identical calculations, but the naive version repeatedly executes small operations through Python loops, introducing interpreter overhead and inefficient CPU execution. The vectorized version delegates matrix operations to NumPy and OpenBLAS, which use optimized native code, cache-aware computation, and SIMD instructions to process multiple values simultaneously. This improves hardware utilization and reduces execution overhead. In my experiment, execution time decreased from 240.783 ms to 0.951 ms, achieving a 253.19x speedup without changing the mathematical result.
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
My Colab environment reported 2 vCPUs when checked after reconnection. The threading experiment showed limited and non-monotonic improvements. Execution time decreased from 2360.311 ms with 1 thread to 2257.346 ms with 2 threads, but increased slightly to 2269.802 ms with 4 threads. Eight threads performed best at 2208.607 ms, only about 6.4% faster than one thread. With limited CPU resources, additional software threads compete for CPU time, shared caches, and memory bandwidth. Scheduling and synchronization overhead can offset parallel execution benefits. The original runtime's vCPU allocation was not independently recorded.
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
Transformer training involves matrix multiplications, attention calculations, and gradient operations that contain many independent arithmetic computations. GPUs can execute these operations simultaneously across thousands of processing units, achieving much greater throughput than CPUs. The Tesla T4 completed 100 training steps in 12654.0 ms, compared with 429414.8 ms on the CPU, achieving a 33.9x speedup. However, very small workloads with little parallel computation may not benefit significantly because GPU kernel-launch overhead and CPU-GPU data transfers can outweigh the computational advantage.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
Parallelism stops helping when additional execution resources cannot compensate for sequential dependencies, hardware limitations, communication costs, and coordination overhead.

The attention experiment demonstrated the importance of efficient vectorized execution. Naive Python loops required 240.783 ms, while vectorized OpenBLAS operations took only 0.951 ms, achieving a 253.19x speedup by reducing interpreter overhead and exploiting SIMD instructions.

The CPU threading experiment demonstrated hardware and coordination limits. Increasing the thread count from 1 to 8 reduced execution time from 2360.311 ms to 2208.607 ms, an improvement of only about 6.4%. Four threads were slower than two. Limited CPU resources, thread scheduling, cache contention, and synchronization overhead can reduce the benefits of additional threads.

The GPU experiment demonstrated massive data parallelism. Training time decreased from 429414.8 ms on the CPU to 12654.0 ms on the T4 GPU. However, sequential dependencies, kernel-launch overhead, and CPU-GPU data movement can still limit speedups, although their individual contributions were not measured.

Overall, parallelism is useful only when sufficient independent work exists to justify the additional hardware and coordination costs.
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
I would repeat the CPU-GPU training comparison using several batch sizes while measuring GPU execution time and data-transfer overhead separately. If GPU speedup increases with batch size, this would support the importance of fixed launch and transfer costs. If speedup remains constant or decreases, I would investigate memory bandwidth and GPU utilization instead.
<!--/answer-->