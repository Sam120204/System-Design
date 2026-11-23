# Fundamentals: Big Data Processing

Big data processing can be divided into batch, micro-batch, and stream processing based on **when computation happens and how continuously arriving data is handled**. Choose an approach by first setting the end-to-end latency the business can tolerate, then considering data volume, late events, correctness requirements, and operational cost.

## 1. Batch Processing

Typical tools: Hadoop MapReduce, Spark Batch, and SQL on a data warehouse.

**How it works:** Accumulate a dataset, then process it offline on a schedule. For example, read data already stored on disk every hour or day to produce reports or aggregates.

**Key characteristics**

- The input is usually a bounded historical dataset that can be scanned in full or across a large range.
- Output becomes available after the job finishes. When the historical data is complete, rerunning the job can produce a consistent result.
- Batch jobs are well suited to producing corrected final results, though correctness still depends on source data quality and the computation logic.

**Advantages**

- **Easy to recompute and reconcile:** Retaining raw data lets you fix logic and recalculate results.
- **High throughput and cost efficiency:** Large sequential scans, columnar storage, compression, and vectorized execution can use resources efficiently.
- **A relatively simple operating model:** Jobs have clear start and finish points, making scheduling, retries, and troubleshooting straightforward.

**Disadvantages**

- **High latency:** Results commonly take minutes or hours, so batch processing does not fit workloads that need continuous updates.
- **Pronounced resource peaks:** Scheduled jobs that start together can create waves of load.
- **Poor fit for immediate decisions:** Real-time alerts or budget controls cannot wait for the next batch run.

**Typical use cases:** Daily or hourly settlement reports, historical backfills, offline model training, audits, and reconciliation.

## 2. Micro-batch Processing

Typical tool: Spark Structured Streaming in its default micro-batch execution mode.

**How it works:** Data arrives continuously, but the engine divides it into small batches at short trigger intervals, such as 1, 5, or 10 seconds. It processes each batch and updates the output.

**Key characteristics**

- Execution and failure recovery happen in batches, making this a practical option for near-real-time workloads.
- The trigger interval is only part of the latency. Queuing, computation, state updates, and writes to downstream systems all add end-to-end delay.
- If processing a batch regularly takes longer than the trigger interval, the backlog keeps growing.

**Advantages**

- **Straightforward to implement and debug:** Each batch has a clear input range, which helps with retries and replay.
- **Seconds of latency are often sufficient:** With capacity planning and monitoring, micro-batches can support many near-real-time dashboards.
- **Strong throughput:** Batch execution can amortize some scheduling and I/O overhead.

**Disadvantages**

- **Scheduling and batch overhead set a floor:** Consistently very low tail latency is difficult to achieve.
- **Updates arrive in steps:** Results usually refresh when each batch completes.
- **Bursts can cause delays:** Larger batches make later batches wait, increasing latency further.

**Typical use cases:** Near-real-time dashboards, minute-level aggregates, trend analysis, and business metrics with a latency target from a few seconds to tens of seconds. If the target is “refresh within 15 seconds,” measure **end-to-end latency** under peak load; a 5-second trigger interval alone does not guarantee it.

## 3. Stream Processing

Typical tools: Apache Flink and Kafka Streams. Apache Beam is a unified programming model that can run on different execution engines.

**How it works:** Continuously consume an unbounded event stream and incrementally update state and aggregates. An engine may process records one at a time or buffer and transfer them in batches internally. The important point is that the application does not need to wait for a fixed, large batch to finish.

**Key characteristics**

- Long-running computations keep state for intermediate information such as windows, deduplication sets, and join results.
- Event time describes when an event actually happened. A watermark represents the system's estimate of how far event-time processing has progressed.
- Checkpoints help restore state and input positions after a failure. Output correctness also depends on how data is written downstream.

**Advantages**

- **Low latency:** Suitable for updates in subsecond to seconds ranges, though tail latency still depends on traffic, state size, and downstream systems.
- **Rich time semantics:** Windows, out-of-order events, and corrections for late events fit naturally into the model.
- **Good for complex stateful computation:** Examples include session windows, deduplication, stream joins, and dynamic Top K calculations.

**Disadvantages**

- **Higher engineering and operational demands:** Teams must manage state growth, checkpoints, backpressure, hot keys, state migration, and upgrades.
- **End-to-end exactly-once does not happen automatically:** In addition to recovering computation state, downstream writes need transactions, idempotent upserts, or deduplication.
- **Resources are continuously in use:** Jobs usually run all the time, and scaling them may require state migration.

**Typical use cases:** Real-time risk detection, fraud prevention, billing or budget control, alerts, online features, and second-level dashboards.

## Comparing the Three Approaches

| Dimension | Batch | Micro-batch | Streaming |
| --- | --- | --- | --- |
| End-to-end latency | Usually minutes to hours | Usually seconds; varies with batch duration and backlog | Usually subsecond to seconds; depends on system design |
| Throughput and unit cost | Efficient for large offline scans | High, with some batch scheduling overhead | Can be high; state, networking, and backpressure matter more |
| Engineering complexity | Low to medium | Medium | High |
| Correctness | Easy to recompute and reconcile from historical data | Batches can be retried; duplicate output still needs handling | Strong guarantees are possible when state, input, and sinks work together |
| Late and out-of-order events | Can be included during recomputation | Event time is possible, subject to engine capabilities and configuration | Well suited to event time, watermarks, and late-event corrections |
| Resource model | Scheduled jobs can use elastic resources | Continuously schedules small batches | Long-running jobs; scaling may move state |
| Failure recovery | Rerun the job | Rerun a failed batch | Restore from a checkpoint and handle output replay |
| Typical output | Daily reports, offline training, final reconciliation | Second-level dashboards and live trends | Risk detection, billing, online features, low-latency Top K |

These latency ranges are common, **not SLAs guaranteed by a tool**. Measure the full path from event creation and ingestion through queuing and computation to writes in the query system.

## Lambda and Kappa Architectures

**Lambda architecture** maintains a fast path and a batch path in parallel. Stream processing produces a near-real-time result first; batch processing periodically recomputes from raw data and then corrects or replaces the fast-path output.

**Kappa architecture** uses stream processing as its primary computation path. When recomputation is needed, the system replays historical events through the stream job. It still needs reliable historical storage, state management, and a way to replace results.

### How the Two Architectures Process Data

In Lambda, both paths consume the same underlying data, but they work independently and on different schedules. The stream path publishes a quick result while the batch path later computes a more complete result. The batch job does not have to wait for the stream job to finish; its output is used to correct or replace the stream result.

```text
                    ┌──> Stream processing ──> Fast, near-real-time result ──┐
Events ──> Durable log                                                   ├──> Served result
                    └──> Batch processing ──> Recomputed result ────────────┘
```

In Kappa, live events and replayed historical events use the same stream-processing logic. A replay is commonly written to a separate result version, checked, and then switched into service. Large replays may need throttling or separate resources so they do not delay live processing.

```text
Live events ────────────┐
                        ├──> Stream processing ──> Served result
Historical log replay ──┘
```

The word **Lambda** in Lambda architecture is unrelated to a **lambda function**, such as AWS Lambda. AWS Lambda is a serverless compute service; it can be used as a component in a data system, but using it does not make the system a Lambda architecture.

| Concern | Lambda | Kappa |
| --- | --- | --- |
| Computation logic | Separate batch and stream jobs can drift in metric definitions | Tries to reuse one stream implementation, which must still support historical data |
| Recalculation | Recompute all or selected partitions from raw data | Replay retained event logs to rebuild state and output |
| Result merging | Define how batch results replace stream results | Isolate replay output and switch versions after validation |
| Resource peaks | Scheduled batch jobs create peaks | Large replays may compete with online processing |
| Tends to fit when | Auditing, reconciliation, and historical recomputation matter | Logs can be replayed long enough and stream logic is reusable |

### Example: Card Fraud Detection

- **Kappa:** Evaluate each payment in a stream to spot suspicious patterns quickly. If the detection logic changes, replay the retained transaction log through the updated job.
- **Lambda:** Use the same fast stream alerts, plus a separate nightly batch job to recalculate fraud totals for auditing or reconciliation.

If replay is sufficient for the required corrections, Kappa keeps the processing logic simpler. If an independent historical calculation is important, Lambda may be a better fit. A payment that must be blocked before authorization still needs a low-latency decision in the payment path; these architectures support analysis and monitoring around that decision.

### Correctness and Auditability

Lambda's batch path can recompute from **immutable raw data**, which makes it useful for billing reconciliation and audits. A batch job can only correct data that was retained; it cannot recover an event that was lost before ingestion.

For Kappa to achieve comparable correctness, logs must be retained long enough, computation must be repeatable, and output must support idempotency or transactions. Replay also needs to account for schema evolution and external dependencies. If computation reads user profiles or configuration that has since changed, replaying the same old events may produce a different result.

### Recalculation and Backfills

- **Lambda:** Batch jobs are suited to large historical scans, but merging batch and live results requires clear keys, versions, and replacement order so an older result does not overwrite a newer one.
- **Kappa:** Replay reuses stream logic, but should be isolated from online traffic. A common approach is to write a separate result version, validate it, and then switch readers over so intermediate replay results are not exposed.

### How to Choose

1. **Start with the latency target:** Use batch for daily reports; evaluate micro-batches for second-level dashboards; consider streaming for subsecond needs or complex real-time state.
2. **Check whether results can be recomputed:** Data completeness, event-log retention, and historical schema compatibility determine whether results can be corrected after a failure.
3. **Define output semantics:** Decide how to deduplicate events, correct late arrivals, avoid duplicate writes after retries, and expose new result versions to readers.
4. **Validate at peak traffic:** Monitor backlog, end-to-end latency, state size, checkpoint duration, and the time required for a replay.

Many systems combine all three approaches: streaming provides immediate results, batch processing supports offline analysis and audits, and micro-batches serve workloads with intermediate latency needs. Keep each path's result definitions and responsibilities clear.
