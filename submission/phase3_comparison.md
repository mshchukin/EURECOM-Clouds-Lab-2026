# Phase 3 — Comparison Writeup

Copy this file to `submission/phase3_comparison.md`, fill in every answer slot, and commit it:

```bash
cp phase-3-mapreduce-spark/comparison_template.md submission/phase3_comparison.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers in section 1 must **match `submission/phase3_report.json`** — copy them across, do not
retype from memory; they are checked against it.

---

## 1. Facts from your runs

Numbers only (e.g. `12`, `20.3`, `0.041`).

| | Value |
|---|---|
| Map tasks in the cloud job (`mapreduce.num_splits`) | <!--answer:num_map_tasks-->12<!--/answer--> |
| Reduce tasks in the cloud job (`mapreduce.num_reducers`) | <!--answer:num_reduce_tasks-->4<!--/answer--> |
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->51.0<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.009<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->5.4<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->6<!--/answer--> |

**Explain the timings.** The three wall times above cover the same 60 documents, but not the
same job: the single-process run and the cloud MapReduce are *the same word count*; Spark's
time is for *TF-IDF*, which is more work (two `reduceByKey`s and a `join`). Rank the three and
explain what each is spending its time on. Say which comparison is like-for-like, and what the
Spark number — a heavier job, on one machine — tells you about where the cloud job's time goes.
(~80 words)

<!--answer:timing_explanation-->
From fastest to slowest: local single-process word count (0.009s), Spark TF-IDF (5.4s), and cloud MapReduce word count (51.0s). Local versus cloud MapReduce is the like-for-like comparison since both run the exact same word count on 60 documents. Local executes purely in RAM in milliseconds, whereas cloud MapReduce spends nearly all 51.0s on HTTP invocations, cold starts, Cloud Workflows state transitions, retry backoffs, and 60+ GCS JSON reads/writes. Spark runs a heavier TF-IDF pipeline in-memory on one machine in 5.4s, showing cloud MapReduce is orchestration- and I/O-bound.
<!--/answer-->

**What the counts tell you.** From your stage count: how many shuffles did the Spark job have,
and which operations in `spark_tfidf.py` caused them? From your retry count: roughly what
fraction of the 16 task attempts was retried, and is that what `--chaos 0.3` should produce?
(~50 words)

<!--answer:counts_interpretation-->
The 4 stages reflect 3 shuffles, caused by reduceByKey in term_freq, reduceByKey in doc_freq, and join in build_tfidf_rdd. With 16 base tasks (12 map + 4 reduce) and 6 retries, 6 out of 22 total attempts failed (~27.3%, or 37.5% of base tasks), matching --chaos 0.3 (~30% failure rate).
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.** Go through `spark_tfidf.py` and the notebook's Spark cells.
Name each RDD operation and say whether it is a transformation or an action. What happened
when you ran `build_tfidf_rdd` — and what happened when you ran `collect`? (~60 words)

<!--answer:transformations_vs_actions-->
parallelize, flatMap, reduceByKey (in term_freq), map and reduceByKey (in doc_freq), and map, join, and mapValues (in build_tfidf_rdd) are all lazy transformations; collect() is the only action. Calling build_tfidf_rdd merely constructed the execution DAG without computing data, whereas calling collect() triggered Spark to execute all 4 stages and return the 5,975 rows to the driver.
<!--/answer-->

**Lineage and recovery.** Using the lineage you printed in 2b and the timeline in 3a: if the
worker holding one output partition of the *first* `reduceByKey` died, what would Spark have
to recompute, and what could it reuse? Name the stage or shuffle boundary where the recovery
starts. (~50 words)

<!--answer:lineage_recovery-->
Spark uses RDD lineage to recover lost partitions without restarting the whole job. If one output partition of the first reduceByKey (term_freq) is lost, Spark recomputes only that missing partition from the upstream input splits starting at Stage 0 (before the first shuffle boundary) and re-runs dependent downstream partitions in Stages 1–3, reusing intact shuffle partitions.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

Name the GCP service or file that played each role (one line each):

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->
  Cloud Workflows (mr-wordcount in phase-3-mapreduce-spark/workflow.yaml)
  <!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->
  Cloud Run functions (mr-worker in phase-3-mapreduce-spark/main.py)
  <!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->
  Google Cloud Storage (project-0787b5e3-aee9-467e-a11-eurecomgpt bucket)
  <!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->
  partition(word, num_reducers) in phase-3-mapreduce-spark/mapreduce.py
  <!--/answer-->

**The barrier.** There is no explicit "wait for all mappers" step in `workflow.yaml`. Where,
exactly, is the barrier between the map and reduce phases — which construct provides it?
(~30 words)

<!--answer:barrier_location-->
The barrier is the implicit completion of the parallel for loop in map_phase inside workflow.yaml: Cloud Workflows waits for every parallel mapper branch (including retries) to finish before starting reduce_phase.
<!--/answer-->

**Idempotence.** The workflow retries a task that failed. Why is that safe with the worker as
written, and what would go wrong if `run_map` *appended* its pairs to the shuffle file instead
of overwriting it? (~60 words)

<!--answer:idempotence-->
Each task reads immutable inputs and overwrites deterministic GCS object paths (shuffle/split-<i>/part-<r>.json or output/part-<r>.json), making tasks idempotent so retrying a failed attempt cleanly replaces any missing or partial output. If run_map appended instead of overwriting, a retry after a partial write would duplicate (word, 1) pairs in the shuffle file and inflate the final word counts.
<!--/answer-->

**The chaos run.** What did you see in the execution log and in Logs Explorer when you ran
with `--chaos 0.3`? Did the job still succeed, and was the final answer still identical to the
local reference? (~50 words)

<!--answer:chaos_observations-->
With --chaos 0.3, the Workflows execution log showed 6 task branches failing with HTTP 503 and automatically retrying via exponential backoff, while Logs Explorer recorded 6 ERROR entries (simulated worker crash) from mr-worker. The job still succeeded in 51.0s and produced matches_local: true.
<!--/answer-->
