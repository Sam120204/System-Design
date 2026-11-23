# Design a Text-to-Video Generation Platform

A client submits a text prompt and video settings, receives a job ID, then polls for status or gets a completion webhook. The baseline output is a 15-second, 720p video. A higher-quality profile supports up to 60 seconds at 1080p. Start with one region, multiple availability zones, and pretrained models.

Target **p99 under 300 ms** for job creation, **p99 under 200 ms** for status reads, and **p95 under five minutes** for standard jobs, including queue time. The submit and status APIs target **99.95% availability**. Use synchronous database replication across availability zones and acknowledge submission only after the metadata and outbox commit can survive the loss of a single zone.

## 1. Architecture and request flow

![Original architecture sketch showing the API gateway, job database, queue, workflow controller, and CDN](images/design-video-generator.png)

1. **Accept the request.** The API gateway authenticates the client and enforces tenant rate limits. The backend checks quota and estimates GPU work from the requested model, duration, and resolution. It can return `429` when the tenant exceeds its quota or the queue lacks room to meet the latency target. Reserve capacity across job classes so long renders cannot crowd out short ones.
2. **Save and enqueue the job.** The backend writes the job and a transactional outbox record together, pinning the model and runtime versions. Once the transaction commits, it returns `202 Accepted` and a job ID. An outbox relay publishes a command with a stable ID to a durable queue and retries failed publishes. Within a tenant, reusing an idempotency key with the same request returns the existing job. Reusing it with different parameters returns `409 Conflict`. If full prompt moderation would exceed the creation latency budget, start the job in `validating` and enqueue inference only after moderation passes.
3. **Run the workflow.** The controller claims a job with a time-limited lease and a monotonically increasing fencing token. It schedules preprocessing, inference, and postprocessing in order. Workers renew the lease by heartbeat and save intermediate artifacts under an attempt-specific key. Progress and output commits require the current token and a valid lease.
4. **Publish the result.** After the final video is durable in object storage and passes integrity and safety checks, the controller records the asset reference, marks the job `completed`, and writes a status event in one database transaction. The API reads status from that database. A separate webhook service delivers the committed event, while clients download the video from the CDN.

### DNS for the API and CDN

| Hostname | DNS target | What happens after resolution |
| --- | --- | --- |
| `api.example.com` | Public address of the regional load balancer or managed API gateway | The client connects over HTTPS to submit a job, poll status, or request its content URL. The ingress routes the HTTP request to a healthy backend. |
| `media.example.com` | CDN distribution hostname, typically via a CNAME | The client follows the API's short-lived signed URL and connects to a CDN edge. The CDN validates the signature and expiry before serving a cached object or fetching it from the private object storage origin. |

The DNS resolver caches the address according to its TTL, so changing a record takes time to reach clients. Depending on the CDN, DNS can steer clients to an edge address, or DNS can return an anycast address and let network routing select the point of presence. The signed URL's expiry is independent of the DNS TTL. Outbound webhook workers also resolve registered callback hostnames and check resolved destinations against the service's egress policy before connecting. [CDN routing deep dive](<../02-Layered Architecture/Cloudflare-Design-CDN.md>)

## 2. GPU execution pipeline

![Original pipeline sketch showing one GPU for preprocessing, eight for inference, and one for postprocessing](images/design-video-generator-pipeline.png)

| Stage | Example allocation | Work |
| --- | --- | --- |
| Preprocessing | 1 GPU | Check the input, optionally enhance the prompt, and check the enhanced prompt. Run cheap validation before admitting the job. |
| Model inference | 8 GPUs | A text encoder produces conditioning embeddings. The Diffusion Transformer (DiT) denoises the video latent, and the decoder turns it into video frames. |
| Postprocessing | 1 GPU | Optionally increase spatial resolution or frame rate, run final safety checks, add a watermark, and encode the video and preview. |
| Delivery | Object storage and CDN | Verify the encoded file's checksum, persist it, then make the completed asset available to authorized clients. |

The **1 → 8 → 1** allocation applies to successive stages. Release each allocation when its stage finishes, so encoding and delivery do not hold an eight-GPU group. Persist stage status and artifact references so a retry can resume from the last usable stage. A `32 × 4 × 96 × 96` latent tensor takes about **2.36 MB** at 16 bits per element. A raw RGB8 buffer for 256 frames at `720 × 720` takes about **398 MB**. Decode and upscale workers need memory and I/O headroom for those larger buffers, even when the encoded output is much smaller.

A 30B-parameter model has about **60 GB** of weights at 16 bits, before activations and runtime overhead. Load weights when workers warm up and reuse them across jobs. During planned maintenance, drain workers by stopping new assignments and allowing active jobs to finish. If adding mid-inference preemption, measure the cost of saving and restoring the latent, denoising step, and sampler state against the cost of restarting the stage.

## 3. API and job state

| Endpoint | Behavior |
| --- | --- |
| `POST /generations` | Accepts a prompt, output settings, an `Idempotency-Key`, and an optional `webhook_endpoint_id`. Returns `202` and the job ID after the database commit. |
| `GET /generations/{job_id}` | Returns status, a monotonic `status_version`, the current stage, and an error code on terminal failure. |
| `GET /generations/{job_id}/content` | Checks ownership, completion, and retention, then returns a `302` redirect to a short-lived signed CDN URL. |
| `POST /webhook-endpoints` | Registers and verifies an HTTPS callback URL for signed status notifications. |

The internal path is `ready → scheduling → running → completed`, with an optional `validating` stage before `ready`. The API reports `queued` while a job awaits validation or its first run, and `in_progress` during execution and automatic retries. Acknowledge a queue message after committing the next stage and its outbox command, a terminal state, or a durable retry. With 30-day asset retention, a job can remain `completed` after its video expires.

### Webhook delivery

Publish committed status events to a durable delivery queue. Dispatchers enforce per-endpoint and per-tenant concurrency limits so a slow receiver cannot consume all delivery capacity. Sign the timestamp and raw request body, and treat any `2xx` response as an acknowledgement. Retry timeouts, `429`, and `5xx` responses with bounded exponential backoff and jitter. Deliveries that exceed the retry window go to a dead-letter queue for inspection or replay. Delivery is at least once. Retries and replay keep the same `event_id` so receivers can deduplicate, and `status_version` lets them detect stale events.

## 4. Failure recovery

| Failure | Recovery |
| --- | --- |
| The backend committed the job but publishing failed | The relay retries the outbox record. A reconciler finds jobs that have remained queued without a live attempt. The controller uses a conditional claim to handle duplicate messages. |
| A worker loses its lease and later finishes | A new attempt gets a higher fencing token. The old worker's status and output commits are rejected, and its unreferenced artifacts can be garbage-collected. |
| A worker crashes after uploading the result | Uncommitted objects remain private and eligible for cleanup. If the database already recorded completion, a redelivered queue message is acknowledged without running inference again. |
| A job fails during validation or execution | Retry transient failures within a bounded budget. End jobs rejected by policy or invalid input. Quarantine jobs that repeatedly crash the model. |
| A compatible GPU group is unavailable | Keep accepted work queued during temporary shortages. Tighten admission as queue age approaches the latency budget. |

Periodically reconcile expired leases, terminal events without delivery records, and completed jobs whose assets are missing or corrupt. Restore the asset from a replica when possible and surface an availability error until the content is restored. The recorded job outcome stays separate from the asset's availability.

## 5. Capacity and scheduling

### Two sizing scenarios

| Scenario | Traffic | Capacity estimate |
| --- | --- | --- |
| Initial deployment | 100,000 jobs/day, about 1.2 jobs/s on average, with a 10 jobs/s peak | At a weighted average of 180 GPU-seconds/job, the fleet needs about 208 busy GPU slots on average, or about **300 slots at 70% utilization** before adding peak and failure headroom. At 25 MB/job for video and previews, expect about **2.5 TB/day** or **75 TB** over 30 days before replicas and temporary storage. If every job polls 18 times, average status traffic is about **21 reads/s**. |
| Scale target from the sketch | 5 million jobs/day, about 58 jobs/s on average, with a 200 jobs/s peak | If every job produces an encoded video in the 50–180 MB range, expect about **250–900 TB/day**. Size GPU capacity from the workload mix and measured GPU-seconds for each profile. |

At the initial scale, database throughput is modest. The queue absorbs bursts and helps share scarce GPU time across tenants. Measure peak status traffic before adding a separate read cache or replica.

### Scheduling and autoscaling

Start with a durable queue and a warm worker pool if all jobs fit the same hardware. Split queues or pools when model variants require different workers, jobs need different GPU counts or memory, or long jobs block the interactive latency target. Include the model version, expected GPU count, GPU-seconds, memory, interconnect requirements, and maximum queue delay in each job's resource profile.

Workers pull only when they can run the next stage. Within a resource class, weighted fair scheduling allocates capacity across tenants, while aging moves long-waiting jobs up in priority. Cap each tenant's queued GPU-seconds and concurrent GPU allocation. Reserve compatible GPUs as a group for multi-GPU jobs and retry the group together if a rank fails. A cluster can have eight free GPUs and still lack an eight-GPU group on the same high-bandwidth interconnect.

For the illustrated pipeline, estimate work as `preprocessing seconds + 8 × inference seconds + postprocessing seconds`. For each resource class, use `(incoming GPU work per second + backlog GPU-seconds / target drain seconds) / target utilization` as a first estimate of fleet size. Count queued work and the remaining work on active attempts in the backlog. Track queue age, provisioning delay, and available GPU groups. Maintain enough warm capacity to handle normal traffic, and drain workers before scaling down.

Benchmark batching for jobs with compatible model versions, tensor shapes, and duration classes. Include the wait to form a batch and the effect of slower jobs on completion latency when measuring throughput gains.

## 6. Showoff time — GPU parallelism and FLOPs

![Original advanced sketch showing the video token calculation, CP4 plus TP2, and LLM and DiT FLOP estimates](images/design-video-generator-showoff.png)

**Derive the token count.** A 16-second clip at 16 fps has 256 frames. With `768 × 768` frames and an assumed 8× compression along time, height, and width, the latent grid is `32 × 96 × 96`. Applying spatial `2 × 2` patches gives `32 × 48 × 48 = 73,728` video tokens per denoising step.

**Shard the long sequence and the model.** The sketch lists Unified Sequence Parallelism (USP) and gives **CP4 + TP2** as an eight-GPU layout. Four-way context parallelism partitions the sequence into **18,432 token positions per context shard**, reducing per-rank activation pressure. Global attention still needs key and value information from the other context shards. Two-way tensor parallelism splits large layer operations and their weights across two GPUs for each context shard. Collectives exchange the partial results.

**Optimize for the fabric and the workload.** Allocate the eight ranks together on a high-bandwidth interconnect and overlap communication with computation where the attention implementation supports it. Profile peak GPU memory, collective overhead, p95 inference latency, and GPU-seconds per completed video across output profiles. Tune context and tensor parallelism for each profile. If a smaller profile fits and meets its latency target on fewer GPUs, use the recovered capacity for independent jobs. Replicate inference groups to raise throughput.

**Estimate the compute.** A rough count for dense weight matrix multiplies is `2 × parameters × tokens`. That gives **140 TFLOPs** for a 70B LLM processing 1,000 tokens. For a 30B DiT processing 73,728 video tokens over 50 denoising steps, it gives **about 221 PFLOPs**. If the model uses full bidirectional attention, the attention score and value products add roughly `4n²dLS` FLOPs for sequence length `n`, hidden width `d`, `L` layers, and `S` steps. Use the model's actual attention pattern for capacity estimates and measured wall-clock time to account for communication and utilization.

Related case studies include [Job Scheduler](design-job-scheduler.md) and [Multi-Tenant Webhooks](design-webhook.md).
