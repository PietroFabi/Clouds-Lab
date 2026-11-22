# Phase 3 — Comparison Writeup

All numbers in section 1 are copied from `submission/phase3_report.json`.

---

## 1. Facts from your runs

| | Value |
|---|---|
| Map tasks in the cloud job (`mapreduce.num_splits`) | <!--answer:num_map_tasks-->12<!--/answer--> |
| Reduce tasks in the cloud job (`mapreduce.num_reducers`) | <!--answer:num_reduce_tasks-->4<!--/answer--> |
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->29.2<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.006<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->3.0<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->8<!--/answer--> |

**Explain the timings.**

<!--answer:timing_explanation-->
Ranking: local single process (0.006 s) < Spark TF-IDF (3.0 s) < cloud MapReduce (29.2 s). The like-for-like pair is local vs cloud: the same word count over the same 60 documents. Locally it is pure computation in memory. The cloud job spends its time almost entirely on overhead: 16 function invocations (with cold starts), dozens of Cloud Storage reads and writes for inputs, shuffle and outputs, workflow scheduling, and the retry backoff of the chaos run. Spark does strictly more work (two reduceByKeys and a join) yet takes about a tenth of the cloud time, because its shuffles stay in one machine's memory and local disk. So the cloud job's time is network and coordination, not computation: the corpus is far too small to amortise per-task overhead.
<!--/answer-->

**What the counts tell you.**

<!--answer:counts_interpretation-->
4 stages means 3 shuffles: the reduceByKey in term_freq, the reduceByKey in doc_freq, and the join of TF with DF in build_tfidf_rdd. With 8 retries, the 16 tasks needed 24 attempts, so 8/24 ≈ 33% of attempts failed (0.5 retries per task). That is close to the 30% per-attempt failure rate --chaos 0.3 should produce; the expected number of retries is 16 × 0.3/0.7 ≈ 7.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.**

<!--answer:transformations_vs_actions-->
Transformations: parallelize (creates the source RDD), flatMap and reduceByKey in term_freq, map and reduceByKey in doc_freq, and the join and map in build_tfidf_rdd. The single action is collect. Running build_tfidf_rdd returned instantly: Spark only recorded the lineage graph (printed with toDebugString in 2b) and computed nothing. Calling collect turned that graph into 4 stages and 12 tasks, executed them, and returned the 5975 rows to the driver.
<!--/answer-->

**Lineage and recovery.**

<!--answer:lineage_recovery-->
The first reduceByKey is the shuffle boundary at ShuffledRDD[3] (spark_tfidf.py line 31). If the worker holding one of its output partitions died, Spark recomputes only that partition: it re-runs that one task in the stage right after this first shuffle, re-reading the map-side shuffle files of stage 0, which it reuses (they are re-run from ParallelCollectionRDD[0] only if those files were lost too). Because the TF RDD feeds both the DF branch and the join, the lost partition is needed downstream in both places. All other partitions and every other completed stage's shuffle output are reused.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->Cloud Workflows, the mr-wordcount workflow defined in workflow.yaml (parallel fan-out, implicit barrier, try/retry).<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->The mr-worker Cloud Run function (main.py), invoked once per map or reduce task and running as the mr-runner service account.<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->Cloud Storage: the eurecom-cloud-project-eurecomgpt bucket, under the job's input/, shuffle/ and output/ prefixes.<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->partition() in mapreduce.py: zlib.crc32(word) % num_reducers, a hash that is stable across processes and machines.<!--/answer-->

**The barrier.**

<!--answer:barrier_location-->
The barrier is the map_phase parallel step itself. A parallel step only completes when every one of its branches (all 12 map tasks, including retries) has finished, so reduce_phase, the next step, cannot start until the last mapper is done.
<!--/answer-->

**Idempotence.**

<!--answer:idempotence-->
run_map always writes complete files with a deterministic name, {job}/shuffle/split-NNN/part-R.json, computed from the same input split, and each write overwrites the whole object. A retried task therefore produces exactly the same files, and running it twice equals running it once. If run_map appended its pairs instead, a task that crashed after writing (or that ran twice) would leave duplicate pairs in the shuffle files, and the reducers would over-count those words: the result would no longer match the local reference.
<!--/answer-->

**The chaos run.**

<!--answer:chaos_observations-->
In the execution view, several map and reduce steps showed extra attempts, retried after the exponential backoff. In Logs Explorer the mr-worker request log showed 8 responses with HTTP 503, each one a simulated worker crash. Every failed attempt was retried and succeeded, the execution finished SUCCEEDED in 29.2 s, and the result was identical to the local reference (matches_local: true, 2221 distinct words).
<!--/answer-->
