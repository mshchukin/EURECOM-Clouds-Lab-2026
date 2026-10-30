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
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->323.491<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->1.024<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->451162.8<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->12498.1<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.4633<!--/answer--> |

**The thread sweep.** **Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say which count was fastest. (verbatim: the numbers, in order)**

<!--answer:thread_sweep-->
1 thread: 2514.321 ms; 2 threads: 2265.789 ms; 4 threads: 2230.512 ms; 8 threads: 2331.3 ms. Four threads was fastest.
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** **Both attention implementations do the same arithmetic and produce the same result (the unit test checks that). Explain where the naive version's time actually goes, and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)**

<!--answer:vectorization_why-->
The naive implementation spends much of its time on overhead rather than arithmetic: its many small operations repeatedly dispatch work, traverse data, and fail to use the processor’s wide vector units efficiently. The vectorized version presents the computation as larger array operations, allowing optimized kernels to process many elements together using SIMD instructions and efficient memory access. The measured time fell from 323.491 ms to 1.024 ms, about a 316× speed-up, with the same arithmetic and result.
<!--/answer-->

**Where the thread curve flattens, and why.** **Using your sweep: more threads stopped helping — or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU count is part of the answer; so is what the threads have to coordinate on. (~80 words)**

<!--answer:threading_plateau-->
Performance improved from 1 thread (2514.321 ms) to 2 (2265.789 ms) and 4 (2230.512 ms), but flattened and then worsened at 8 threads (2331.3 ms). The best result was at 4 threads. This suggests the workload was already using the available Colab vCPUs effectively by then; adding threads can oversubscribe the CPU. Threads also need to coordinate and share data, and scheduling, synchronization, and memory-bandwidth contention add overhead that can outweigh the extra parallel work. The exact vCPU count should be checked in the Colab runtime rather than assumed.
<!--/answer-->

**What made the GPU faster.** **The model, the data and the step count were identical; only the device changed. Say what about a transformer training step suits a GPU, and name one workload that would *not* see this speed-up. (~70 words)**

<!--answer:gpu_why-->
Transformer training performs large matrix multiplications and other tensor operations across many elements. A GPU has many compute units designed to execute these operations in parallel, so it can process the model’s dense tensor work far faster than the CPU in this measurement: 12,498.1 ms versus 451,162.8 ms for 100 steps. A small, sequential workload with frequent dependencies—such as parsing a short text file one line at a time—would not use the GPU’s parallel hardware effectively and might be slower after data-transfer and launch overhead.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** **Pull the three experiments together. Your answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware you do not have, coordination that costs more than it saves, data that has to move — and say which one bound each experiment. One concrete number from section 1 per claim. (~150 words)**

<!--answer:parallelism_limit-->
The attention experiment mainly shows the benefit of exposing parallel work to optimized hardware: vectorization reduced time from 323.491 ms to 1.024 ms. The remaining limit is not evidence that the arithmetic itself became smaller; it is the finite throughput of the processor and the cost of moving data through memory. In the thread sweep, scaling stopped helping after 4 threads: 4 took 2230.512 ms, while 8 took 2331.3 ms. That points to limited available CPU capacity plus coordination, scheduling, and contention costs outweighing extra parallelism. The GPU comparison shows the importance of having suitable hardware: training took 451,162.8 ms on CPU and 12,498.1 ms on GPU for 100 steps. The GPU’s parallel compute suits transformer tensor operations, but it cannot eliminate sequential dependencies, and moving data to and from the device also has a cost. Overall, parallelism helps where work is abundant and hardware matches it; it stops helping when work is indivisible, hardware resources are saturated, coordination is expensive, or data movement becomes the bottleneck.
<!--/answer-->

**What you would measure next.** **One experiment this phase did not run that would sharpen the answer, and what result would change your mind. (~50 words)**

<!--answer:next_experiment-->
I would repeat the thread sweep while recording the Colab vCPU count, CPU utilization, and memory bandwidth, and test more thread counts around the apparent optimum. If 8 threads consistently beat 4 when the runtime has at least 8 available vCPUs and utilization is high, I would reconsider whether the observed slowdown was due to oversubscription; low utilization or saturated memory bandwidth would instead point to coordination or data movement.
<!--/answer-->
