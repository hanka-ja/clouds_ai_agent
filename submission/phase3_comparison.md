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
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->38.4<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.019<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->6.0<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->5<!--/answer--> |

**Explain the timings.** The three wall times above cover the same 60 documents, but not the
same job: the single-process run and the cloud MapReduce are *the same word count*; Spark's
time is for *TF-IDF*, which is more work (two `reduceByKey`s and a `join`). Rank the three and
explain what each is spending its time on. Say which comparison is like-for-like, and what the
Spark number — a heavier job, on one machine — tells you about where the cloud job's time goes.
(~80 words)

<!--answer:timing_explanation-->
Local (0.019 s) is fastest, then Spark (6.0 s), then the cloud MapReduce (38.4 s). Local vs. cloud is the like-for-like pair: same word count, same code. The cloud timer starts after the splits are uploaded and stops when the execution ends. It covers starting the execution, 21 function calls (16 tasks plus 5 retries, with backoff waits), the workers' 60 reads and 52 writes in Cloud Storage, and status polling every 3 s. Without chaos the job took 10.8 s. Spark does more work on one machine in 6 s, so the cloud time is coordination and I/O, not computation.
<!--/answer-->

**What the counts tell you.** From your stage count: how many shuffles did the Spark job have,
and which operations in `spark_tfidf.py` caused them? From your retry count: roughly what
fraction of the 16 task attempts was retried, and is that what `--chaos 0.3` should produce?
(~50 words)

<!--answer:counts_interpretation-->
4 stages means 3 shuffles: the reduceByKey in term_freq, the reduceByKey in doc_freq and the join. In the chaos run 5 of 21 attempts failed (24%), which is the per-attempt rate that --chaos 0.3 controls; 24% instead of 30% is normal variation for 21 attempts. Relative to the 16 tasks, the 5 retries add 31% extra attempts.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.** Go through `spark_tfidf.py` and the notebook's Spark cells.
Name each RDD operation and say whether it is a transformation or an action. What happened
when you ran `build_tfidf_rdd` — and what happened when you ran `collect`? (~60 words)

<!--answer:transformations_vs_actions-->
Transformations: flatMap and reduceByKey in term_freq, map and reduceByKey in doc_freq, the map to (term, (doc_id, tf)), the join and the final map to rows. The only action is collect. build_tfidf_rdd returned at once because it only recorded the lineage, which toDebugString printed before any task ran. collect then ran the whole job: 4 stages, 12 tasks, 6.0 s.
<!--/answer-->

**Lineage and recovery.** Using the lineage you printed in 2b and the timeline in 3a: if the
worker holding one output partition of the *first* `reduceByKey` died, what would Spark have
to recompute, and what could it reuse? Name the stage or shuffle boundary where the recovery
starts. (~50 words)

<!--answer:lineage_recovery-->
The first reduceByKey produces ShuffledRDD[3], which stages 1 and 2 both read. If one of its partitions were lost, Spark would fetch that partition's blocks from the stage 0 shuffle files again and recompute only that partition and the tasks that depend on it. Recovery starts at the boundary between stage 0 and stage 1. Only if the stage 0 shuffle files were lost too would those map tasks have to rerun.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

Name the GCP service or file that played each role (one line each):

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->Cloud Workflows, the `mr-wordcount` workflow defined in `workflow.yaml` (fan-out with `parallel` + `for`, retry with `try`/`retry`).<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->The Cloud Run function `mr-worker` (`main.py`); each map or reduce task is one HTTP invocation of it.<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->Cloud Storage, the bucket `project-a8899376-13d6-4f4d-88d-eurecomgpt` with the `input/`, `shuffle/` and `output/` prefixes of each job.<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->`partition()` in `mapreduce.py`: `zlib.crc32(word) % num_reducers`, so every mapper sends a word to the same reducer.<!--/answer-->

**The barrier.** There is no explicit "wait for all mappers" step in `workflow.yaml`. Where,
exactly, is the barrier between the map and reduce phases — which construct provides it?
(~30 words)

<!--answer:barrier_location-->
It is the map_phase parallel step itself: a parallel step only finishes when all its branches have finished, so reduce_phase cannot start earlier. In my chaos run the last map request was logged at 21:06:02 and the first reduce request at 21:06:10.
<!--/answer-->

**Idempotence.** The workflow retries a task that failed. Why is that safe with the worker as
written, and what would go wrong if `run_map` *appended* its pairs to the shuffle file instead
of overwriting it? (~60 words)

<!--answer:idempotence-->
Each task reads the same input and overwrites fixed object names (shuffle/split-i/part-r.json, output/part-r.json), so running it twice leaves the same files and the workflow can retry without knowing how far the failed attempt got. If run_map appended instead, a retried map task that had already written some partitions would add its pairs again, those words would be counted twice and the result would no longer match the local reference.
<!--/answer-->

**The chaos run.** What did you see in the execution log and in Logs Explorer when you ran
with `--chaos 0.3`? Did the job still succeed, and was the final answer still identical to the
local reference? (~50 words)

<!--answer:chaos_observations-->
The execution still succeeded (2221 words, identical: True) but took 38.4 s instead of 10.8 s. I did not open the execution view or Logs Explorer in the console; I read the mr-worker request logs with gcloud logging read instead. They show 21 requests: 16 with status 200 and 5 with 503, the status main.py returns for a simulated worker crash. Three map tasks failed once and one reduce task failed twice before succeeding.
<!--/answer-->
