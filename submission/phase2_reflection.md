# Phase 2 — Reflection: where does parallelism stop helping?

All numbers below are copied from `submission/phase2_report.json` (Colab, T4 GPU runtime).

---

## 1. Your measurements

| | Value |
|---|---|
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->594.077<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->7.082<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->489879.2<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->17196.7<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.4565<!--/answer--> |

**The thread sweep.**

<!--answer:thread_sweep-->
1 thread: 2524.764 ms, 2 threads: 2748.81 ms, 4 threads: 3060.352 ms, 8 threads: 2480.684 ms. Fastest: 8 threads (2480.684 ms), but only 1.7% faster than 1 thread, which is within run-to-run noise. In practice no thread count beat a single thread.
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.**

<!--answer:vectorization_why-->
The naive version spends almost all of its 594.077 ms in the Python interpreter: every multiply-add is a separate bytecode dispatch on boxed objects, one scalar at a time, with no reuse of data in cache. The vectorized version hands whole matrices to NumPy's compiled BLAS, which runs tight cache-blocked loops using SIMD (AVX) instructions that process several floats per instruction. Same arithmetic, executed by the hardware instead of the interpreter: 7.082 ms, 83.89x faster.
<!--/answer-->

**Where the thread curve flattens, and why.**

<!--answer:threading_plateau-->
It never improved: 2 threads (2748.81 ms) and 4 threads (3060.352 ms) were slower than 1 (2524.764 ms), and 8 (2480.684 ms) was only noise-level faster. The free Colab runtime has 2 vCPUs, which are typically two hyperthreads of one physical core sharing the same floating-point units and memory bandwidth, so there was no extra compute to split work onto. Each operator also has to fork, synchronize and join its threads, and with small matrices (batch 16, n_embd 256) that overhead outweighs the gain; at 4 and 8 threads the cores are oversubscribed and the threads contend.
<!--/answer-->

**What made the GPU faster.**

<!--answer:gpu_why-->
A transformer training step is dominated by dense matrix multiplications (QKV and output projections, attention scores, MLP layers): millions of independent multiply-adds with no branching, which the T4 spreads across thousands of CUDA cores with far higher memory bandwidth than the CPU. That gave 489879.2 ms vs 17196.7 ms (28.5x). A workload that would not benefit: autoregressive generation of one sequence at batch size 1, which is inherently sequential token by token and launches tiny kernels, or branch-heavy serial code like the naive Python loop.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?**

<!--answer:parallelism_limit-->
Each experiment hit a different kind of limit. Work that cannot be divided: the naive attention was bound by the serial Python interpreter (594.077 ms); vectorization removed that bottleneck rather than adding parallel workers, giving 7.082 ms. Hardware you do not have: the thread sweep was bound by the runtime's 2 vCPUs (likely one physical core), so 4 threads had nothing to run on and took 3060.352 ms versus 2524.764 ms for 1. Coordination that costs more than it saves: even 2 threads were slower (2748.81 ms), because the per-operator synchronization outweighed splitting such small matrices. Data that has to move: the GPU won by 28.5x, but at about 172 ms per step (17196.7 ms / 100) for a 4.9M-parameter model it is not saturated: every batch is assembled on the CPU and copied host-to-device, and each small layer pays kernel-launch overhead. Parallelism stops helping once the remaining time is serial work, missing cores, synchronization or data transfer instead of arithmetic.
<!--/answer-->

**What you would measure next.**

<!--answer:next_experiment-->
A GPU batch-size sweep (32, 64, 128, 256), reporting time per sample. If time per sample keeps dropping sharply, the T4 was starved by launch and transfer overhead at batch 32; if it stays flat, it was already compute-bound and my data-movement claim is wrong. Repeating the thread sweep on a 4+ physical-core machine would test the hardware claim the same way.
<!--/answer-->
