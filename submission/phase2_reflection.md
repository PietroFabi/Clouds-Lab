# Phase 2 — Reflection: where does parallelism stop helping?

All numbers below are copied from `submission/phase2_report.json` (Colab, T4 GPU runtime).

---

## 1. Your measurements

| | Value |
|---|---|
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->396.929<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->5.866<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->482278.1<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->16250.4<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.4965<!--/answer--> |

**The thread sweep.**

<!--answer:thread_sweep-->
1 thread: 3929.34 ms, 2 threads: 1968.147 ms, 4 threads: 2490.048 ms, 8 threads: 2605.638 ms. Fastest: 2 threads (1968.147 ms).
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.**

<!--answer:vectorization_why-->
The naive version spends almost all of its 396.929 ms in the Python interpreter: every multiply-add is a separate bytecode dispatch on boxed objects, one scalar at a time, with no reuse of data in cache. The vectorized version hands whole matrices to NumPy's compiled BLAS, which runs tight cache-blocked loops using SIMD (AVX) instructions that process several floats per instruction. Same arithmetic, executed by the hardware instead of the interpreter: 5.866 ms, 67.67x faster.
<!--/answer-->

**Where the thread curve flattens, and why.**

<!--answer:threading_plateau-->
The curve turns at 2 threads: going from 1 to 2 threads halved the step time (3929.34 ms to 1968.147 ms, 2.0x), but 4 threads (2490.048 ms) and 8 threads (2605.638 ms) were slower than 2. The Colab runtime has 2 vCPUs, so 2 threads use all the hardware there is; beyond that, threads are oversubscribed and the OS time-slices them on the same cores, adding context switches and cache thrashing. Each operator also has to fork, synchronize and join its threads at a barrier, and with small matrices (batch 16, n_embd 256) that coordination cost grows with the thread count while the compute available does not.
<!--/answer-->

**What made the GPU faster.**

<!--answer:gpu_why-->
A transformer training step is dominated by dense matrix multiplications (QKV and output projections, attention scores, MLP layers): millions of independent multiply-adds with no branching, which the T4 spreads across thousands of CUDA cores with far higher memory bandwidth than the CPU. That gave 482278.1 ms vs 16250.4 ms (29.7x). A workload that would not benefit: autoregressive generation of one sequence at batch size 1, which is inherently sequential token by token and launches tiny kernels, or branch-heavy serial code like the naive Python loop.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?**

<!--answer:parallelism_limit-->
Each experiment hit a different kind of limit. Work that cannot be divided: the naive attention was bound by the serial Python interpreter (396.929 ms); vectorization removed that bottleneck rather than adding parallel workers, giving 5.866 ms. Hardware you do not have: threads helped exactly up to the 2 vCPUs of the runtime (3929.34 ms at 1 thread to 1968.147 ms at 2) and no further. Coordination that costs more than it saves: past 2 threads the extra threads only add synchronization and context switching, so 4 threads took 2490.048 ms and 8 took 2605.638 ms, both slower than 2. Data that has to move: the GPU won by 29.7x, but at about 162 ms per step (16250.4 ms / 100) for a 4.9M-parameter model it is not saturated: every batch is assembled on the CPU and copied host-to-device, and each small layer pays kernel-launch overhead. Parallelism stops helping once the remaining time is serial work, missing cores, synchronization or data transfer instead of arithmetic.
<!--/answer-->

**What you would measure next.**

<!--answer:next_experiment-->
A GPU batch-size sweep (32, 64, 128, 256), reporting time per sample. If time per sample keeps dropping sharply, the T4 was starved by launch and transfer overhead at batch 32; if it stays flat, it was already compute-bound and my data-movement claim is wrong. I would also repeat the thread sweep several times on a machine with 4+ physical cores, to check that the turning point moves with the core count.
<!--/answer-->
