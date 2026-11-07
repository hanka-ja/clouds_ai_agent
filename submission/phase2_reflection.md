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
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->235.384<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->1.041<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->411133.1<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->14088.3<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.4383<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
The timings for 1, 2, 4 and 8 threads were 2145.856, 1952.592, 2274.375 and 2120.132 ms, respectively. The fastest result was when using 2 threads, the second fastest result was when using 8 threads, 3rd when using 1 thread and the slowest was when using 4 threads.
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
The vectorized approach uses the machine’s resources more efficiently. The naive version spends time repeatedly executing Python loops, indexing arrays and calling small operations. The vectorized version expresses the same calculation as whole-matrix operations, handled by optimized compiled BLAS code. This reduces Python overhead and allows SIMD instructions to process several numbers at once, with better use of CPU caches. In my run, this made it about 226 times faster.
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
Using 2 threads instead of 1 improved performance and achieved the best result, 1952.592 ms. My Colab runtime had 2 vCPUs, so adding more threads meant sharing the same CPU resources. With 4 threads, performance became worse than with a single thread. Surprisingly, 8 threads improved again, but remained slower than 2. Extra scheduling, synchronization between workers and competition for memory can outweigh the benefits. The difference between 4 and 8 could also reflect timing variability in this short benchmark.
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
The main reason the GPU was faster is that much of transformer training involves matrix multiplication. These calculations contain many similar operations that can run in parallel. GPUs are designed to execute large numbers of such operations efficiently, making my training run about 29.2 times faster. A highly sequential workload, such as a Python loop following a linked list where each address depends on the previous result, would not see this speed-up.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
Parallelism stops helping when work is sequential, available hardware is already fully used, or coordination and data movement cost more than they save. Vectorizing attention reduced the time from 235.384 to 1.041 ms by reducing Python-loop overhead and using optimized matrix operations. This does not remove dependencies between stages: attention scores must still be calculated before their softmax weights. With CPU threads, 2 gave the best time, 1952.592 ms, on my 2-vCPU runtime. Adding threads did not add hardware, and scheduling, synchronization and shared-memory contention could explain the slowdown at 4 and 8 threads. GPU training took 14088.3 ms compared with 411133.1 ms on CPU, showing that parallel computation outweighed its overhead for this workload. However, transferring data from RAM to GPU memory and launching operations can outweigh the savings for small batches. These timings show the overall benefit, but do not separately measure transfer or coordination costs.
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
I would like to test whether CPU data preparation and transfer were bottlenecks. I would compare GPU training with the current batching method against batches prepared and stored on the GPU beforehand. If training barely became faster, I would reconsider whether supplying data was an important limit in this experiment.
<!--/answer-->
