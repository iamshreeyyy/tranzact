# Approach Document

> **Design status:** This document proposes a failure-aware architecture and provides a reproducible validation plan. It deliberately separates requirements covered by design from performance and cost claims that require measurement. No load-test results or AWS Pricing Calculator export are claimed unless attached as evidence. The only candidate-specific submission step is to add genuine public AI conversation links in Section 5; never fabricate those URLs.

---

## Requirement map (read this first)

| TranZact requirement | Answer in one line | Where | Status |
|---|---|---|---|
| Single + bulk (up to 100) with Puppeteer and HTML templates | One queue per job type, Puppeteer worker pools, bulk fanned out per document | Decision 1 | Aligned |
| ~1,000 singles/min + 5-10 bulk requests/min | Queue + isolated worker pools; provisioned capacity and admission control | Sections 3, Decision 1 | **Validation required**; worker capacity and peak duration must be measured |
| Single download starts within 5 s | Requires enough pre-warmed render capacity; 202 acceptance alone does not satisfy this | Sections 3.3, 4, 8 | **Conditional**; cannot be claimed until end-to-end load tests and full-cost estimate pass |
| Bulk progress updates | Per-document state, progress = done ÷ total | Decision 1 | Aligned |
| 500+ line items crash Puppeteer | Heavy routing, isolated workers, chunked rendering | Decision 3 | Untested |
| Unreliable downloads | Stored zip, presigned URLs with Range requests, per-file fallback | Decision 4 | Aligned |
| Stale data (invoice #87) | Request-time snapshot with versions; workers never re-read the ERP | Decision 2 | Aligned (needs the ERP payload contract) |
| Tamper evidence | SHA-256 + Object Lock + KMS-signed manifest | Decision 5 | Aligned |
| Under ₹12,500/month | Itemised estimate + spend guardrail and explicit feasibility limits | Section 8 | **Estimate only**; confirm with AWS Pricing Calculator and measured peak duration |
| Existing ERP, no re-architecture | ERP adds one HTTP client and a payload builder; its Redis and RDS are untouched | Decision 1, diagram | Aligned |
| Caching layer in the diagram | No new cache; job state lives in DynamoDB; the existing Redis stays ERP-only | Diagram | Aligned |

---

## 1. Initial Understanding

The central problem is not converting HTML into a PDF; it is isolating a memory-heavy renderer from the existing ERP while handling a highly uneven workload, returning usable files quickly, preserving the exact data used for each document, and staying within a strict monthly budget. At the stated 2–3 seconds per render, 1,000 single requests/minute implies roughly 33–50 concurrent renders before bulk traffic is included. At the maximum stated bulk rate, another 500–1,000 document renders/minute may arrive. These peak figures are radically different from the monthly average of about 0.69 PDFs/minute, so peak duration and whether both traffic classes peak together materially affect the design.

The key engineering constraint is that a durable queue prevents accepted work from being lost, but it does not make a PDF ready within five seconds. Autoscaling cannot replace pre-provisioned capacity during a sudden burst. Therefore, the architecture separates a budget-controlled operating mode from the stricter service level: if five seconds is a hard requirement at the full combined peak, the capacity and full monthly cost must be measured and the budget must be revisited if the measured design exceeds it. The service must never report an accepted job as completed before the artifact is actually downloadable. Bulk inputs must be frozen at acceptance, and large documents need isolated rendering because their failure mode is memory pressure, not merely latency.

*This section records the understanding reflected in the final design; it should not be represented as a verbatim record of my unaided first thoughts.*

---

## 2. Assumptions & Clarifying Questions


### Assumptions
1. The ERP sends the **full document data in the JSON payload** (not just IDs), built from **one consistent read** with a version per document (the snapshot contract in Decision 2). If it only sends IDs, Decision 2 describes the fallback.
2. The 1,000 single requests/min peak is **not** this one customer's load (see Section 3). I size for a short burst, not for 24/7 peak.
3. Bulk requests (5-10/min, up to 100 PDFs) arrive **on top of** the single requests.
4. A PDF takes 2-3 s and ~400 MB, as stated. I have **not measured** this. Worker sizing uses this number and should be validated on the real instance type.
5. The ERP and the PDF service share a VPC, so the ERP can call the service over private networking.
6. Compliance retention period is set by the compliance team; I use a placeholder of 8 years for cost purposes *(assumption)*.
7. PDFs are downloaded about once each on average.
8. Users are authenticated by the ERP; the PDF service trusts only signed requests from the ERP.


### Questions I would ask the product / engineering team
1. Is the 1,000/min peak for this customer or for the whole platform? (30,000 PDFs/month is 0.69/min on average; the stated peak would consume the whole month in 30 minutes.)
2. How long does a peak last, and how often per day? This decides whether burst capacity is affordable.
3. Is the ₹12,500 budget per customer or for the whole service?
4. Is the 5 s target a **hard SLA at the 1,000/min peak**, including cold-start bursts, or a normal-load objective? If hard, which constraint can change: the budget, peak rate, or SLA? A queue cannot satisfy a deadline by itself, and autoscaling cannot instantly add capacity. The current estimate does not show that both requirements can be met.
5. Does the ERP send full data or only document IDs?
6. What is the compliance retention period, and may archived PDFs be moved to cold storage?
7. Is PDF-level digital signing (e.g. PAdES) required by regulators, or is provable integrity enough?
8. How many line items do typical purchase orders have? What share is above 200 or above 500?
9. If the monthly burst-compute budget is used up, may the service stop scaling (queue bulk, return 429 to singles), or must it keep spending? Who owns that trade-off?
10. Does the compliance/security team require private-only networking for the workers? (At my unverified endpoint prices it costs roughly ₹3,600/month more per AZ; see Section 8.7.)

---

## 3. Capacity Planning & Math


### 3.1 Monthly workload vs peak load
| Quantity | Calculation | Result |
|---|---|---|
| Monthly render time | 30,000 × 2-3 s ÷ 3,600 | **16.7-25 h** |
| Average rate | 30,000 ÷ (30 × 24 × 60 = 43,200 min) | **0.69 PDFs/min** |
| Peak singles in flight (Little's Law) | (1,000 ÷ 60 = 16.7/s) × 2-3 s | **33-50 concurrent** |
| Memory for singles | 33-50 × 400 MB | **13.3-20 GB** |
| Bulk extra load | 5-10 requests/min × 100 PDFs | **500-1,000 PDFs/min** |
| Bulk in flight | 8.3-16.7/s × 2-3 s | **17-50 concurrent** |
| Combined peak | 1,500-2,000 PDFs/min = 25-33/s | **50-100 concurrent, 20-40 GB** |

Peak-to-average ratio: 1,000 ÷ 0.69 ≈ **1,440×**. That is not plausible for one small business, which is why I treat the peak as platform-wide or short bursts and ask in Section 2.

**Implication:** the monthly work is only ~17-25 render-hours (2.3-3.5% utilization of a 720 h month). Provisioning 50-100 slots 24/7 would waste ~97% of the spend. The design uses a queue, a small always-on baseline, and burst workers.


### 3.2 Worker capacity (preliminary estimate, not validated)
- The assignment gives 400 MB per render but nothing about CPU per render. Chromium rendering is CPU-heavy, so **CPU, not memory, is probably the binding limit**. CPU contention makes each render slower than 2-3 s, which raises concurrency and queue depth (a feedback loop).
- **Working hypothesis (unmeasured):** ~12 concurrent renders per 8 vCPU / 16 GB node (memory: 12 × 400 MB = 4.8 GB, so memory is not the limit at that level) and 3 slots on the 2 vCPU / 8 GB baseline node (2 single + 1 batch). Three concurrent Chromium renders on 2 vCPUs is itself doubtful (see Decision 1), so the baseline is a hypothesis too.
- Needed slots: 50-100 concurrent → "~6 burst nodes (72 slots)" is a **starting number, not proven capacity**.
- Sensitivity of burst cost to the unknown renders-per-node (72 slots, 20 burst hours/month, $0.36/h on-demand, spot assumed 50% off):

| Renders per node | Nodes for 72 slots | Burst compute cost, on-demand | Burst compute cost, spot |
|---|---|---|---|
| 6 | 12 | $86.4 | $43.2 |
| 12 (hypothesis) | 6 | $43.2 | $21.6 |
| 18 | 4 | $28.8 | $14.4 |

Section 8 shows how this moves the monthly total. Renders per node is the first number I would measure.


### 3.3 What happens at the start of a peak (queue depth)
- **Routing assumption (defined in Decision 1):** 5% of documents are heavy and run in the batch group. At the peak, ordinary singles are 16.7 × 95% ≈ 15.8/s (single pool) and heavy singles are 16.7 × 5% ≈ 0.83/s (batch group).
- The baseline single pool serves 2 slots ÷ 2.5 s = 0.8 ordinary PDFs/s, so the ordinary-single backlog grows by about 15.8 − 0.8 = 15/s until burst nodes arrive.
- Burst nodes need about a minute of metric lag plus 1-2 minutes to start, so for roughly 150 s only the baseline serves. About 15 × 150 ≈ **2,250 ordinary single requests** exceed what the single pool can serve within 5 s. The baseline batch slot (one heavy render of ~10 s ≈ 0.1/s) also cannot keep up with 0.83 heavy singles/s, which adds about 0.73 × 150 ≈ 110 more (heavy bulk documents come on top). The rough figure of **≈2,400** is therefore the combined total across both pools. Under the backpressure rule (Decision 1) most of these requests get 429 or wait, and the ERP/UI must retry or show "busy".
- Once the single pool is at full cap (50 slots ≈ 20 PDFs/s at 2.5 s), the net drain of ordinary singles is 20 − 15.8 ≈ 4.2/s, so a backlog of 500 clears in about 120 s. At 3 s per render the pool serves 16.7/s, a net drain of only ~0.9/s (500 jobs take about 9 minutes), and above roughly 3.2 s per render (50 ÷ 15.8) the single pool falls behind.
- **Conclusion:** the current budget-limited design does **not** meet a hard five-second download-start SLA during a sudden peak. SQS plus autoscaling protects durability and isolates work; it does not remove queueing delay. EC2 Auto Scaling warm pools can reduce instance launch/initialization delay, but warm capacity still costs money and does not create render capacity beyond what is provisioned. AWS documents warm pools as a way to reduce scale-out latency, not as a guarantee of end-to-end request latency (<Link url="https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-warm-pools.html" title="AWS Auto Scaling warm pools documentation" />). The current design must not claim the SLA is met. The scaling signal also lags; measure the actual queue-age-to-ready delay under load rather than assuming a fixed launch time.

**What "5 seconds" means (two different measurements):**
- *Accept latency:* the time for the API to return 202 with a job ID. It only validates, stores the snapshot and records, and enqueues, so the target is well under 1 s at p95 *(assumption)*.
- *Time to ready:* the time from the request until the PDF can start downloading. This is the assignment's "download starts within 5 seconds". It is queue wait + render (2-3 s) + upload + link, roughly 3-4 s when the queue is empty *(estimate)*. At normal load (≈0.7 PDFs/min) the queue is empty, so the target is met. At a sudden peak it is best effort (above).
- For singles, `GET /jobs/{id}?wait=8` long-polls and returns the presigned link as soon as the PDF exists, so the download starts the moment it is ready and the client does not busy-poll. The metric to alarm on is p95 time to ready.

**Service contract for the 5 seconds (proposed; to be confirmed, question 4):**
- **Normal-load objective:** target 3–4 seconds end-to-end when the queue is empty (2–3 seconds rendering plus upload/link overhead). This is an estimate to test, not a measured result. The baseline's nominal 40–48 singles/min is also unverified and may be lower under CPU contention.
- **Peak SLA:** the present design cannot guarantee a download-ready PDF within five seconds at 1,000 singles/min. If the SLA is hard, keep enough render capacity already running before the peak, size it using measured p95 render time and CPU/memory, and maintain headroom for heavy documents and bulk traffic. A warm pool may shorten provisioning but should not be counted as ready capacity until the worker is healthy and polling. This capacity is not costed as 24/7 in Section 8 and is likely to exceed the budget; therefore the two constraints are currently in conflict.
- **Budget sanity check (not a final sizing):** at the unverified hypothesis of 12 ordinary render slots per 8-vCPU node, 50 concurrent ordinary single renders require at least 5 nodes already running. At the draft price of $0.36/node-hour, five on-demand nodes running 24×7 cost 5 × $0.36 × 720 = **$1,296/month (~₹1.08 lakh at ₹83/USD)** for compute alone, before heavy-document capacity, bulk workers, ALB, storage, logs or the baseline. This rough lower-bound check shows why the current $150 total budget cannot support a hard 1,000-single/min, 5-second peak guarantee under the stated render-time assumptions. The node/slot assumption must be benchmarked, but it cannot reasonably erase this order-of-magnitude gap.
- **Chosen provisional policy if the budget is non-negotiable:** protect accepted work, apply rate limits/admission control before queues become unbounded, return `429` with `Retry-After` for requests that cannot be accepted within the service envelope, and show an explicit “busy/retry” state. This is honest overload handling, but it does **not** satisfy a requirement that every peak request's download start within five seconds. Product/engineering must approve this trade-off before implementation.
- **An accepted request is not a finished PDF.** The status API reports `ACCEPTED → RENDERING → READY` (or `FAILED`) and `ready_at`. The ERP UI must show "preparing" until `READY`, never "done".
- **Decision gate:** do not mark the five-second requirement “met” until product clarifies whether it is a hard peak SLA. If it is hard, revise the budget or peak envelope and recalculate capacity with a load test. If the ₹12,500 cap is hard, document a lower guaranteed admission rate and explicitly label peak five-second latency as best-effort. These constraints cannot be declared simultaneously satisfied by this estimate.


3.4 Storage, network, queue
- **PDF storage:** 30,000 × 100 KB-1 MB = 3-30 GB/month (mid ≈ 15 GB). One year ≈ 180 GB at the midpoint.
- **Snapshots (Decision 2):** ~20 KB average JSON × 30,000 ≈ 0.6 GB/month.
- **Network at peak:** 16.7 PDFs/s × 0.5 MB ≈ 8.3 MB/s (≈ 67 Mbps) for singles, up to ~134 Mbps at 1 MB. Downloads go directly from S3 via presigned URLs, so they never pass through our instances.
- **Download volume:** ≈ 15 GB/month, which is small.
- **Queue messages:** 30,000 PDFs ≈ 30,000 messages/month plus retries. Trivial for SQS.
- **Bulk job latency:** 100 PDFs × 2.5 s = 250 s serial. With ~10 parallel slots ≈ 25-30 s plus zip assembly.

---



3.5 Benchmark plan: prove capacity before claiming the SLA

The numbers above are a sizing hypothesis, not a capacity result. The benchmark must be run against the actual HTML templates, representative purchase orders/invoices, the chosen EC2 family, and the production container limits. Do not use a synthetic one-page PDF as the only workload.

**Test harness and workload matrix**

1. Build a deterministic fixture set: ordinary documents (5–100 rows), boundary documents (101–200 rows), heavy documents (201–500+ rows), invoices with long descriptions, Unicode/GST fields, images if templates support them, and the largest expected payload. Use synthetic or sanitized data.
2. Warm the service and Chromium pool; then run both cold-start and warm steady-state tests. Record p50/p95/p99 end-to-end time, render time, queue wait, time-to-download-ready, CPU, RSS/container memory, OOM/restarts, retries, queue age, 429 rate, S3 upload time and error rate.
3. Test ordinary singles at 1, 5, 10, 16.7 requests/s; test bulk at 5 and 10 requests/min with 100 PDFs each; then combine the top stated load (16.7 single requests/s + 16.7 bulk-document jobs/s) for 15 minutes, 30 minutes and a 60-minute soak. If the 30,000/month figure implies that the peak cannot persist, that duration is an assumption to confirm, not a reason to skip a stress test.
4. Run a burst-arrival test: inject 1,000 single requests over 60 seconds without pre-announcing the load. Repeat with the warm pool enabled and disabled. This measures whether warm capacity, not just autoscaling, meets the latency objective.
5. Run a memory-failure test with multiple 500+ row documents, intentionally kill a worker mid-render, and verify that other jobs continue, retries do not duplicate progress, and poison documents reach the DLQ.
6. Run a bulk reliability test: interrupt the client download after 80%, resume the same ZIP using HTTP Range, verify byte-for-byte equality with the original, and confirm individual-file fallback.
7. Run a consistency test: change invoice 87 after snapshot acceptance but before rendering; verify the resulting PDF contains the accepted snapshot version, not the newer live version.
8. Run an integrity test: modify one byte in a copied PDF and verify the SHA-256 manifest check fails; verify the KMS signature, object version and Object Lock retention configuration independently.
9. Repeat each key scenario at least three times. Report raw results and configuration, not only averages. Use p95/p99, since a five-second expectation is a tail-latency requirement.

**Pass/fail gates**

| Gate | Acceptance criterion | What failure means |
|---|---|---|
| Single latency | p95 time from accepted request to download-ready ≤ 5 s at the agreed peak; also report p99 | If this fails, do not claim the five-second target. A 202 response alone is not success. |
| Throughput | Sustained measured completion rate meets the agreed single + bulk offered load without a growing queue | Increase measured capacity, reduce offered peak, or document overload rejection with product approval. |
| Memory safety | No OOM at supported concurrency for the representative heavy-document set; memory stays below the container limit with operational headroom | Lower concurrency, increase isolation/memory, or improve chunking and retest. |
| Queue stability | Queue age returns to baseline after the defined burst; no unbounded backlog | Capacity or admission policy is insufficient. |
| Retry correctness | One logical result and one progress increment per document despite worker termination at each crash window | Fix idempotency/state transitions before load testing further. |
| Download recovery | Resumed ZIP hash matches original; individual files remain available | Fix Range support/client logic or fallback UX. |
| Snapshot correctness | Invoice 87 reflects the accepted snapshot version | Fix ERP snapshot contract before launch. |
| Tamper evidence | Altered bytes fail verification; signed manifest verifies against the trusted KMS public key | Fix artifact finalization/verification; do not claim audit-grade tamper evidence. |
| Budget | Measured workload mapped to a current AWS Pricing Calculator estimate and a 30-day Cost Explorer projection stays below ₹12,500 with headroom | The cost requirement is not met; do not use the compute-only guardrail as proof. |

**Tooling:** use k6 (or an equivalent load generator) for HTTP arrival patterns, a Node.js/Puppeteer worker benchmark for per-render CPU/RSS/latency, CloudWatch/OS metrics for infrastructure, and scripted checksum/Range tests for downloads. Keep the load generator outside the worker hosts so it does not distort results. The repository may include a `bench/` directory with sanitized fixtures, a repeatable runbook, raw CSV/JSON results, and the exact container/instance configuration.

**Decision rule:** do not promote the 12-renders-per-node hypothesis into the capacity plan until this benchmark passes. If the five-second target is mandatory at 1,000 singles/minute, benchmark the single path against that rate while bulk load is present. If the measured warm capacity and its full monthly cost exceed the cap, record the requirements as mutually infeasible under current assumptions and request a product decision; do not quietly return 429s and call the SLA met.


---

4. Design Decisions


### Decision 1: Queue and worker architecture

**Alternatives considered:**
1. Render inside the HTTP request. Simple, but memory grows with traffic (33-50 concurrent renders at peak ≈ 13-20 GB) and a spike can crash every in-flight job.
2. One shared queue. Easy, but one 100-PDF bulk job (~250 s of render time) can sit in front of singles that must start within 5 s.
3. Separate queues with memory-capped workers (chosen).

**Chosen approach:** Three SQS queues plus a dead-letter queue:
- **single**: highest priority, one job per PDF.
- **bulk**: a bulk request is fanned out into one job per document. Progress = completed ÷ total. A failed document retries alone instead of failing all 100.
- **heavy**: documents above **200 line items**, routed by the API at submit time, run at concurrency 1 on memory-capped containers. The threshold is a **safety hypothesis, not a proven boundary**: crashes are reported at 500+, so 200 keeps a margin; it should be tuned from a measured memory-versus-rows curve and the row distribution in question 8.

Flow: the ERP calls the PDF API (signed request plus idempotency key). The API validates, stores a snapshot, writes a job row, enqueues, and returns **202 + job ID**. Workers render with Puppeteer and write PDFs to S3. The client polls job status and receives a download link.

**Browser lifecycle (important for the 1,000-request/minute burst):** a worker does **not** launch a new Chromium process for every request. Each worker starts Chromium during worker initialization, reuses that browser for bounded page-level jobs, and creates a fresh page/context per document. A semaphore limits active renders by the *measured* CPU and memory budget; pages are always closed in `finally`, and the browser is recycled after a measured render count, crash, or memory threshold. Heavy documents run in isolated workers with concurrency 1 and a larger memory limit. Reuse removes repeated browser-startup overhead, but it does not remove the CPU/memory cost of rendering; only benchmark results can establish safe concurrency.

**Admission control:** the API validates and authenticates before accepting work. It reserves capacity for single jobs, limits per-tenant submissions, bounds each queue and total queued work, and returns `429` with `Retry-After` *before* durable acceptance when the configured service envelope is exceeded. A `202` is returned only after the snapshot, idempotency/job record and queue publication are durably reconciled. A request that is accepted is never silently discarded. This protects memory and the ERP; it cannot guarantee five-second completion for a peak larger than provisioned capacity. The product owner must approve the overload contract.

Scheduling policy (SQS provides **no** priority across queues; priority comes from the worker layout):
- **Separate worker pools**, each polling only its own queue and scaling on its own signal. There is **no cross-queue borrowing in v1**: simpler, and idle bulk capacity is cheap because the bulk pool scales down.
- Slot allocation (all counts are provisional starting assumptions to validate by benchmark):

| Pool | Queue | Baseline node (t3.large, 2 vCPU; 3 slots, hypothesis) | Burst capacity | Scales on |
|---|---|---|---|---|
| single | single | 2 slots, always on | single burst group, on-demand preferred: up to 4 nodes × 12 = 48 slots (50 with baseline, the upper end of the singles estimate) | oldest message age > 2 s |
| bulk | bulk | shares 1 batch slot with heavy | batch burst group (bulk + heavy), spot: 2 nodes at the design point, **hard cap 4 nodes × 12 = 48 slots** (49 with the baseline slot) | backlog larger than ~60 s of drain time |
| heavy | heavy | shares the 1 batch slot (concurrency 1, higher memory limit) | none of its own: heavy containers run on the batch nodes, inside the 4-node allowance | queue depth |

- **Capacity levels:** the *design point* is 6 burst nodes (4 single + 2 batch; ~72 slots plus 3 baseline). The *hard cap* is 8 burst nodes (4 single + 4 batch; 99 slots with the baseline), the upper end of the combined estimate. The node cap limits **instantaneous capacity only**; it does not limit monthly spend, which is controlled separately by the burst budget below.
- A 100-document bulk job can only occupy bulk slots, so it cannot delay singles. Singles cannot starve bulk, because bulk has its own slots and its own scale-out trigger. Node caps protect the budget.
- **Deployment model:** Docker containers on EC2 (no ECS/EKS/Fargate in v1). The baseline node runs the API plus one container per pool with CPU and memory limits. Burst nodes come from a launch template (user-data starts the worker container) managed by Auto Scaling groups. Section 8 costs exactly this; network paths are in 8.7.

Other rules:
- Per-worker concurrency = min(CPU-based limit, (container memory − headroom) ÷ 400 MB). Total memory is bounded by design, not by traffic. Treat 400 MB as a measured planning input, not a hard upper bound: Chromium overhead, font/image assets and 500+ row documents can exceed it. Leave OS/Node/browser headroom, set container memory limits, enforce a render semaphore, and test 1/2/4 concurrent pages under cgroup memory monitoring. If a worker reaches its limit, isolate the failure to that worker, retry from the immutable snapshot, and send repeated failures to the DLQ with an actionable status. Never respond to overload by launching more browsers.
- Autoscale on **queue age (oldest message)**, not CPU, which lags behind the real problem.
- Messages not acknowledged (worker OOM-killed) return to the queue after the visibility timeout (set longer than the maximum render time); max ~3 attempts, then DLQ with a clear failure reason.
- Backpressure: if the single queue's oldest message exceeds a threshold, the API returns 429 with `Retry-After` instead of accepting unbounded work.
- The job queue is **not** the monolith's Redis, so PDF load can never affect ERP caching.

Idempotent processing (SQS standard queues deliver **at least once**, so duplicates happen even without crashes; the goal is an *effectively-once result*, not exactly-once execution):
- Each document has one item in DynamoDB: `QUEUED → RENDERING (lease_until, attempt) → DONE (sha256)`.
- The output key is **stable and deterministic**: `pdfs/{tenant}/{job_id}/{doc_id}.pdf`.
- Worker steps:
  1. Claim with a **conditional update**: allowed only if status is `QUEUED`, or `RENDERING` with an expired lease. If status is `DONE`, delete the message and stop. If another worker holds a live lease, skip.
  2. `HEAD` the output key. If the object exists (a previous attempt uploaded it), read its stored SHA-256 and jump to step 5. **Do not re-render.**
  3. Render from the snapshot and compute SHA-256.
  4. Upload with an S3 conditional write (`If-None-Match: *`), storing the SHA-256 as a checksum. If it fails because the object exists, another attempt won; discard this render and use the existing object's hash. *(Verify that conditional writes work on a versioned, Object Lock bucket. Fallback: the HEAD check plus the lease, which leaves a small race window.)*
  5. Finalize with one **DynamoDB transaction**: set `DONE` (condition: not already `DONE`) and increment the job's progress counter in the same transaction, so progress is counted once per document.
  6. Delete the SQS message.
- Crash windows: before step 4 completes (nothing stored, retry re-renders); between steps 4 and 5 (retry finds the object at step 2 and only finalizes); after step 5 before step 6 (retry sees `DONE` and deletes the message).
- Why write-once matters: re-rendering the same snapshot does **not** produce byte-identical PDFs (timestamps, internal IDs), and with Object Lock plus versioning an overwrite would create a second locked version that cannot be deleted. So a retry must detect an existing artifact and reuse it, never overwrite it.

Capacity accounting (one definition, used in every calculation, the cost model and the diagram):

| Group | Baseline slots | Burst nodes, design point | Burst nodes, hard cap | Slots at hard cap |
|---|---|---|---|---|
| Single pool | 2 | 4 | 4 | 50 |
| Batch group (bulk + heavy) | 1 | 2 | 4 | 49 |
| **Total** | 3 | 6 | 8 | **99** |

- **Heavy documents are not extra capacity.** Any request above 200 rows, whether it arrives as a single or inside a bulk request, runs in the batch group, so it reduces what is left for bulk. **Definition used below:** until question 8 is answered, 5% of *all* documents (singles and bulk alike) are assumed heavy, and a heavy render takes about 10 s. At the top of the estimate (16.7 singles/s + 16.7 bulk documents/s = 33.3/s):
  - Singles: 16.7/s splits into 15.8/s ordinary (single pool: 15.8 × 2.5 s ≈ 40 of its 50 slots; at 3 s per render 47.5 of 50, about 5% spare) and 0.83/s heavy (batch group).
  - Bulk: 16.7/s splits into 15.8/s ordinary (batch group) and 0.83/s heavy (batch group).
  - Heavy total: 1.67/s × 10 s ≈ 17 slots, all in the batch group. At the hard cap that leaves about 32 slots (49 − 17) for ordinary bulk, which needs 15.8 × 2.5 ≈ 40. The shortfall of ~7 slots means the bulk backlog grows by about 3 documents/s (~180/min) while ordinary singles are unaffected. At the design point only ~8 slots (25 − 17) remain, so the backlog grows much faster (worked example under Overload behaviour). Without heavy documents the hard-cap batch group would roughly keep pace (16.7 × 2.5 ≈ 42 slots needed against 49).
- **Definition of a slot.** A *slot* is the capacity to run one ordinary render at a time (assumed about 1 vCPU and about 400 MB; 12 slots per 8 vCPU node is the unmeasured hypothesis). The heavy figures are **time-weighted**: a heavy render is assumed to occupy one slot for 10 s instead of 2.5 s, so 1.67 heavy documents/s × 10 s ≈ 17 slots in use on average. That is not the same as 17 ordinary renders running at once, and it ignores that a heavy render probably needs more memory (and possibly more CPU) than one slot; the heavy container's higher memory limit is not modelled by the 12-per-node hypothesis. "Concurrency 1" means each heavy container renders one document at a time; at the peak about 17 heavy containers run in parallel, spread over the batch nodes (about 4 per node at the hard cap). If a heavy render really needs two slots' worth of resources, heavy demand doubles to about 34 slots, so this is a priority benchmark.
- **Baseline CPU is a hypothesis.** Three concurrent Chromium renders on 2 vCPUs, alongside the API and controller, is probably too many if each render needs about one core. Memory allows it (3 × 400 MB) but latency may not. The benchmark decides. Fallbacks: drop the baseline to 2 single slots (bulk and heavy then wait for a burst node), or use a larger baseline node, which at assumed prices costs about $57/month more and would break the budget in most scenarios.

Spend controls (a guardrail, not a billing guarantee; two separate limits):
- **Limit A, capacity:** at most 8 burst nodes at once (4 single + 4 bulk).
- **Limit B, compute money:** a monthly **burst-compute budget of $40**. It covers burst-node *compute* only and is **not a cap on the whole AWS bill**. It is about 88% of the $45.5 left after fixed costs in Section 8; the rest covers per-node IPv4 and EBS charges and estimate error. It is tracked at assumed prices: $0.36 per on-demand node-hour, $0.18 per spot node-hour. At full cap that funds about **22 hours/month with 6 nodes** (4 on-demand + 2 spot, $1.80/h), **18.5 hours with 8 nodes** (4 on-demand + 4 spot, $2.16/h), or **13.9 hours with 8 on-demand nodes** ($2.88/h).
- **Mechanism:** a small controller process on the baseline node runs about once a minute. It computes burst node-minutes from the Auto Scaling groups' running instances, keeps the running total in DynamoDB, and lowers each group's maximum size when a threshold is crossed. At a threshold it stops new scale-outs and starts a **controlled scale-in**: nodes stop pulling new messages, finish their in-flight jobs (an Auto Scaling lifecycle hook gives them a drain window), then terminate. A job interrupted anyway is retried safely (idempotent processing).
- **A guardrail, not a guarantee:** the controller sees instance counts only once a minute, instances launched just before the threshold are still running, spot prices vary, and the real bill includes charges the counter does not model. The 10% margin absorbs part of this. The AWS bill remains the source of truth, so the counter should be compared with the bill every month. AWS Budgets gives an independent alert, but it does not stop instances.
- **When a limit is reached** (provisional policy; product team to confirm, question 9):
  - Node cap reached (A): the overload behaviour below.
  - 70% of the burst-compute budget used: the batch group (bulk + heavy) stops scaling above its baseline slot (bulk and heavy are queued and slow down); singles keep scaling.
  - 100% used: no more burst nodes. Singles run on the baseline (2 slots, about 40-48 PDFs/min); above that the 429 rule applies. Bulk and heavy documents drain slowly on the one shared batch slot. Ops is alerted, and the budget can be raised manually if the business decides to spend more.
- **Cost of this policy:** in a heavy month customers see slower bulk jobs and rejected singles late in the month. That is the price of a hard budget, so it has to be agreed with the product team and not assumed.

Submission and recovery (partial failures at accept time). The API must write S3 snapshots, DynamoDB records and SQS messages, which cannot be one atomic operation. This is a design proposal until it is fault-tested (Section 6, item 7). Two states are kept distinct:
- **Durably accepted** (job `ACCEPTED`): the snapshots and every document record are stored, so the work can be recovered even if the API dies right now. The **202 is returned only at this point**, never earlier.
- **Enqueued** (document `QUEUED`): the SQS message was published. A document can be accepted but not yet enqueued; the reconciler closes that gap.
- The reconciler is safe to run repeatedly and concurrently: re-enqueueing a document is harmless (duplicates are detected at claim time), and it only touches documents still `ACCEPTED` after a grace period.

1. The ERP sends an `Idempotency-Key` (the job ID). Repeating a call with the same key returns the same job.
2. Write the snapshots to deterministic S3 keys (repeatable: same key, same content).
3. Create the job item as `ACCEPTING`, conditional on the key not existing. If it exists, resume instead of duplicating.
4. Create the per-document items in batches of 25 (I do not rely on one transaction for 100+ items), each `ACCEPTED` with a sparse-index attribute `pending_enqueue_at`.
5. Mark the job `ACCEPTED` and return **202**. A crash before this point means the caller never got a 202: a retry with the same key resumes, and a cleanup job deletes `ACCEPTING` jobs older than ~1 hour.
6. Enqueue (SendMessageBatch, 10 per call). Messages carry only IDs and S3 keys (SQS messages are limited to 256 KB). Then conditionally mark each document `QUEUED` and clear `pending_enqueue_at`.
7. The **reconciler**, part of a small controller process on the baseline node that runs about once a minute (it also runs the spend guard), queries the sparse index for documents still `ACCEPTED` after ~60 s and enqueues them. This covers an API crash after the 202 but before enqueueing, an SQS error, and a failure to mark `QUEUED`.
- If a message is enqueued twice (enqueue, then crash before marking `QUEUED`), the claim step in the idempotent-processing flow above makes the duplicate harmless.
- Alarm on "documents `ACCEPTED` for more than 5 minutes". If the reconciler itself is down, new submissions still flow; only recovery is delayed.

Overload behaviour (when demand exceeds the node cap):
- **Singles:** the API estimates wait = queue depth ÷ current service rate. Above a soft limit of about 15 s *(provisional product decision, to confirm)* it returns **429 + Retry-After** instead of accepting. Below it the request is accepted, the UI shows "preparing", and the PDF may arrive later than 5 s.
- **Bulk:** there is no 5 s promise. Jobs are accepted and queued, and the status shows queued count and an ETA. A backlog cap (e.g. 5,000 queued documents) triggers 429, and a per-tenant limit on concurrent bulk jobs (e.g. 2) stops one customer from filling the queue.
- **Worked example, ordinary bulk at the top of the stated range.** Bulk is 1,000 documents/min = 16.7/s, of which 95% (15.8/s) is ordinary; heavy demand is assumed to keep its ~17 slots (definition above).
  - *Design point:* 25 batch slots − 17 for heavy ≈ 8 slots ≈ 3.3 ordinary documents/s at 2.5 s, against 15.8/s arriving. The backlog grows by about **12.5 documents/s**. Over a 10-minute burst the *offered* backlog would grow by about 7,500 documents, exceeding the 5,000-document backlog cap. The cap therefore triggers 429 responses for the excess bulk submissions (about 2,500 documents, which the ERP must retry later), and the *actual* queued backlog stays bounded at about 5,000. After the burst ends, all 25 slots clear that backlog at 10 documents/s in about 8 minutes, assuming arrivals, including heavy documents, return to normal and the batch group stays available.
  - *Hard cap:* 49 − 17 ≈ 32 slots ≈ 12.9 documents/s, a deficit of about 2.9/s (consistent with the ~3/s above). A 10-minute burst builds about 1,700 queued documents, cleared in about 90 s afterwards (49 slots ≈ 19.6/s). That assumes heavy and ordinary arrivals return to normal levels, the batch group remains available, and the residual arrival rate is small relative to its drain capacity; sustained heavy traffic or further bursts increase the clearing time.
  - **Under the 5%-heavy, 10 s assumption the batch group does not keep pace, even at the cap.** Without heavy documents it would (19.6/s against 16.7/s). Bulk users see slow progress, not failures, and the backlog cap and per-tenant limits protect the queue.
- **Single pool at the cap:** ordinary singles are about 15.8/s (heavy singles go to the batch group). 50 slots ÷ 2.5 s = 20/s, about 79% utilization. At 3 s per render the pool serves 16.7/s against 15.8/s (~95% utilization), so any queue builds or drains very slowly; the break-even render time is 50 ÷ 15.8 ≈ 3.2 s, above which the single pool falls behind. The upper end of the estimate sits right at the edge, which is why the 429 path exists.
- **Capacity is provisional and close to the limit.** The hard cap can only *approximately* match the estimated peak, and only under the assumed render times and 12 renders per node. At the top of the estimate the combined arrival rate is 33.3 PDFs/s (2,000/min). The cap gives about 99 slots (50 single + 49 batch, with heavy documents drawing from the batch share as shown above). If every render took the same time: at 2.5 s the cap serves ~39.6/s (about 19% headroom), at 3 s about 33/s (none), and at 3.5 s about 28/s, a deficit of ~5/s, so the queue grows by roughly 300 jobs per minute **even with every node running**. Counting heavy documents at 10 s (5% of documents) lowers this: total demand is about 96 slots at 2.5 s per ordinary render (31.7/s × 2.5 s + 17) against 99 available (about 3% headroom), and about 112 slots at 3 s, which is a shortfall. CPU contention, heavy documents (which also occupy slots) and renders slower than 3 s all push in that direction. These figures rest on the unmeasured 12-renders-per-node hypothesis, so they are a plan to validate, not proof that the peak is covered.

**Why:** the capacity math shows the monthly work is tiny but the peak is huge. A queue lets cheap capacity absorb bursts. Separate queues map to separate requirements: the 5 s SLA, bulk progress, and OOM on large documents. Always-on capacity for the peak would cost roughly 11× the budget (Section 8 reasoning).

**Tradeoff accepted:**
- The 5 s target is **best effort at the start of an extreme peak** (Section 3.3). Accepting a request quickly is not the same as delivering the PDF: the API returns 202 fast, but a PDF can be ready later than 5 s while the queue is backed up. Mitigations: scale out early on queue age, optional scheduled pre-scaling if peaks are predictable, a warm baseline, and 429 backpressure. I will not claim a guarantee.
- More moving parts than one queue.
- Concurrency numbers are unmeasured and need validation.

---


### Decision 2: Bulk data consistency (the stale invoice #87)

**Alternatives considered:**
1. **Workers fetch live data from the ERP when each document renders.** This is the cause of the reported bug (invoice 87 rendered ~3 minutes later saw newer data).
2. **Hold a database snapshot / long transaction on the ERP side for the whole job.** Gives consistency, but long transactions on the production Postgres hurt the monolith, and they cannot span 3+ minutes safely.
3. **Optimistic version check:** render live, compare `updated_at`/version, and fail the document if it changed. Consistent, but users see random failures.
4. **Freeze the data at request time (chosen).**

**Chosen approach:** Freeze the data at the moment the request is accepted, under an explicit **snapshot contract**:
- The ERP builds the whole bulk payload from **one short, consistent read** (a single read-only transaction at REPEATABLE READ, seconds rather than minutes) and sends it in one request. Each document carries `doc_id` and `doc_version` (row version or `updated_at`); the request carries `snapshot_at`.
- The API rejects payloads without versions, writes each document as an **immutable snapshot** (`snapshots/{job_id}/{doc_id}.json` in S3) with `snapshot_at` and a content hash, and returns a `snapshot_sha256`.
- Workers render **only from the snapshot** and never call the ERP.
- The PDF footer and metadata show *"Data as of <snapshot_at>, version <doc_version>"*.
- A snapshot is only as coherent as the payload. The service cannot verify that the ERP built it from one read, so that is part of the contract; versions and timestamps are stored so any mismatch can be audited later.
- If the ERP only sends IDs (question 5): it must supply an explicit version per ID, **or** expose a bulk-read endpoint that the service calls **once, at accept time**, returning all documents from a single transaction. The service never fetches documents one by one while rendering.

**Why:** the bulk job takes minutes, so the only way to have one consistent point in time is to read the data once at the start and never again. This also makes retries safe: a retried document renders from the same snapshot and produces the same content.

**Tradeoff accepted:**
- A PDF reflects data as of request time. If the invoice is edited 30 seconds later, the PDF is older than the live data. That is correct for audit purposes and is made visible by the "data as of" stamp; users who want newer data run a new request.
- Extra storage (~0.6 GB/month *(assumption)*) and a larger request payload to validate. Max payload size must be enforced (100 docs × up to 500+ rows can reach several MB).

---


### Decision 3: Large documents (500+ line items) and OOM

**Alternatives considered:**
1. Give every worker much more memory. Simple but multiplies cost, and a larger limit only moves the crash point.
2. Increase Node's heap size. Does not help, because the crash is in the Chromium renderer process, not the Node heap.
3. Reject large documents. Not acceptable for the business.
4. **Route + isolate + chunk (chosen).**

**Chosen approach:**
- The API counts rows at submit time and routes documents above the threshold to the **heavy queue**.
- Heavy workers: concurrency 1, higher container memory limit, hard per-job timeout, run in a container so an OOM kills only that container, not other jobs. The queue's visibility timeout retries automatically.
- **Chunked rendering:** render the table in chunks of ~100 rows, producing partial PDFs, then merge them with a PDF library (e.g. pdf-lib or qpdf). This keeps the DOM small for each render. Page numbers ("Page X of Y") and grand totals are applied in a final pass, because they depend on the whole document.
- Recycle the browser after N renders to stop slow memory growth.
- After max retries, the job goes to the DLQ and the user sees a clear "failed: document too large, support notified" status, not a hang.

**Why:** the failure is caused by the size of one render, so the fix is to shrink the unit of work and isolate it. Isolation guarantees that a bad document cannot corrupt other jobs.

**Tradeoff accepted:**
- Chunking is **unproven**. It can change page breaks, repeated table headers, running totals and pagination, and it adds a merge step and a page-number pass. I must test with real 500-1,000 row documents to confirm both that it removes the OOM and that the layout stays correct. If it does not, the fallback is a single-pass render of a simplified template on the heavy pool with a higher memory limit.
- Large documents are slower (they run one at a time).
- Row-count routing is a proxy for memory; images or heavy CSS can also cause OOM, so a timeout and retry path is still needed.

---


### Decision 4: Reliable downloads on unreliable connections

**Failure case addressed:** if a 100-PDF ZIP download disconnects at 80%, the server does not regenerate the PDFs or rebuild the ZIP. The completed ZIP remains a durable S3 object under a deterministic job key; the client resumes with HTTP Range requests against the same object (using a client that supports resume and validating the returned range/ETag). If the client or proxy cannot resume ZIP downloads, the UI offers the already-generated individual PDFs and allows the ZIP to be retried. Keep the ZIP until its configured expiry/retention policy; never use an expiring local worker file as the only copy. The API authorizes the job and issues a short-lived presigned URL; the browser downloads directly from S3, not through the render worker.

**Alternatives considered:**
1. Stream a zip through the API/worker on demand. If the connection drops, the user restarts from zero, and the instance is tied up.
2. Email a link. Does not fix interrupted downloads.
3. **Pre-built artifacts on S3, served by presigned URLs (chosen).**

**Chosen approach:**
- Single PDFs and bulk zips are stored in S3. The client downloads via a short-lived **presigned URL** from a `GET /jobs/{id}/download` endpoint that is authenticated and can be called again any time to get a fresh URL.
- S3 supports HTTP **Range** requests, so browsers and download managers can **resume** a broken download instead of restarting.
- The bulk zip is built **once** by a final job, streamed to S3 with multipart upload (never held entirely in memory), and stored uncompressed (PDFs are already compressed).
- Per-file downloads stay available, with a manifest listing each file, size, and SHA-256 hash. A customer who lost 3 of 50 files re-downloads only those 3 and verifies them.
- Presigned links are short-lived and can be re-issued at any time. Bulk zips expire after about 7 days *(assumption)*. Each individual PDF is stored **once**, in the Object Lock bucket, for the compliance period, so there is no duplicate working copy to pay for.

**Why:** a failed download then costs the customer a few KB of retry, not the whole zip, and the service does not need to hold long-lived connections. It also keeps download traffic off our instances and avoids data-transfer costs beyond S3's.

**Tradeoff accepted:**
- Extra storage for zips and an extra assembly step before a bulk job is "ready".
- Resume depends on the client supporting Range requests (all common browsers do).
- Zip expiry means a customer has to re-request a bulk download after the window (the individual PDFs remain available).

---


### Decision 5: Tamper evidence and archival, kept simple

**Verification path:** after the PDF is uploaded, calculate SHA-256 from the exact bytes and store the digest, S3 object version ID, job/document ID, snapshot hash and generation timestamp in a manifest. Sign the canonical manifest with KMS; retain the PDF version and signed manifest under the approved Object Lock retention. To verify, download the exact object version, recompute SHA-256, compare it with the manifest, and verify the manifest signature. A mismatch proves the stored bytes differ from the recorded artifact. Object Lock protects retained object versions against overwrite/deletion; it does not by itself prove that a PDF was correct at generation time, and a hash stored only beside a mutable PDF would not be sufficient. Use a separate tightly controlled signing role and audit key usage.

**Alternatives considered:**
1. **Embedded digital signatures (PAdES) with a CA-issued certificate and trusted timestamps.** Strongest, and verifiable in a PDF reader, but needs certificate management, cost, and signing in the render pipeline.
2. Hash stored only in the database. Simple, but whoever can edit the PDF in storage and the DB together can tamper undetected.
3. Blockchain anchoring. Overkill.
4. **Hash + immutable storage + signed manifest (chosen).**

**Chosen approach:**
- After rendering, compute the **SHA-256** of each PDF and store it in the job table along with the snapshot hash, template version, and `generated_at`.
- Write archived PDFs to an S3 bucket with **Object Lock (compliance mode)** and a retention period set by compliance. Objects cannot be overwritten or deleted during retention, even by an admin.
- Once a day, build a **manifest** of all hashes, **sign it with an AWS KMS asymmetric key**, and store it in the same locked bucket. This closes the weakness of Option 2: even if the database were altered, the signed manifest still proves what existed.
- A lifecycle rule moves the PDFs to a cheaper storage class after ~90 days. The small snapshot JSON objects stay in S3 Standard, because cold storage classes bill a 128 KB minimum per object (Section 8.8).
- To verify: recompute SHA-256 of the PDF, compare with the record and the signed manifest, and verify the manifest signature.

**Why:** it gives provable integrity with managed services and no change to the render pipeline, which matches "without overcomplicating the architecture".

**Tradeoff accepted:**
- Verification needs our records or the manifest (the PDF is not self-verifying in a reader). PAdES is the upgrade if regulators or customers need it.
- Object Lock is irreversible during retention: an incorrect document cannot be deleted. The process is to issue a corrected document that supersedes it.
- Trust rests on AWS KMS and IAM controls; the key policy must restrict signing to the manifest job.

---

5. AI Usage Log

I used AI tools at different stages of the work: initial exploration, research, and then shaping and critically reviewing the proposed design. The links below are the public conversations associated with those stages.

1. **Initial exploration — Claude:** [Claude conversation](https://claude.ai/share/5452e699-2d5e-4f99-ac8a-c82174957b92)  
   Used for the initial problem exploration and early design thinking before the approach was refined.

2. **Research — Gemini:** [Gemini conversation](https://share.gemini.google/X3Bx1tFfoJok)  
   Used as an additional research input. Any suggestions were treated as hypotheses to evaluate against the assignment constraints, calculations, and the proposed architecture—not as verified benchmark or pricing evidence.

3. **Shaping and critical review — ChatGPT:** [ChatGPT conversation](https://chatgpt.com/share/6ad51027-71a0-83ee-ad7e-9dbcc0019489)  
   Used to structure and refine the approach, review the capacity arithmetic and failure-handling design, and challenge assumptions around concurrency, the five-second download target, and the monthly cost ceiling.

**How AI output was evaluated:** I did not treat generated recommendations as proof that the system meets the peak SLA or budget. In particular, the render concurrency, worker capacity, AWS costs, and five-second peak target still require validation through measured load tests and current pricing. I also kept the distinction between accepting a job (`202 Accepted`) and having a completed, downloadable PDF ready.


---

Core failure-mode walkthrough (explicit requirement coverage)

| Problem | Mechanism | What it guarantees / limitation |
|---|---|---|
| 1,000 requests arrive in one minute | Reuse warm Chromium per worker; bounded page semaphore; separate single/bulk/heavy pools; bounded queues and admission control | Prevents one browser per request and uncontrolled memory growth. Does **not** make the stated peak feasible under the $150 cap by itself. The hard five-second peak requirement remains a business/capacity conflict until measured and funded. |
| Each render uses ~400 MB; concurrency causes OOM | Calculate concurrency from measured CPU and memory with headroom; container limits; fresh page/context; recycle browser; isolate heavy docs at concurrency 1; bounded retries + DLQ | Ten renders imply at least ~4 GB for render memory alone, before Chromium/OS/Node and document-specific overhead. Concurrency must be load-tested; do not assume 12 renders/node. |
| ZIP download fails at 80% | Store completed ZIP on S3; Range-resume the same object; keep individual PDF fallback and job status | Network interruption does not lose generated work. Resume depends on the client supporting Range requests; otherwise retry the stored ZIP or download individual PDFs. |
| Invoice 87 changes during bulk generation | Freeze all document payloads at request acceptance from one consistent ERP read; version/hash snapshots; render only snapshots | Later ERP edits do not change PDFs in the accepted bulk job. Snapshot coherence depends on the ERP implementing the stated transaction/payload contract. |
| PDF is modified after generation | SHA-256 of exact bytes, immutable S3 object version, KMS-signed manifest, Object Lock retention; verify by recomputing digest and checking signature | Detects byte changes and prevents retained versions being overwritten/deleted under configured retention. This is tamper evidence, not proof that the original business data was correct. |

---

6. Weaknesses & Future Improvements

**Weakest parts**
1. **Everything is sized from the assignment's numbers, not measurements.** The 400 MB per render and ~12 renders per node are assumptions. If CPU or memory per render differs, the node count and cost change.
2. **The five-second peak SLA is currently unmet if interpreted as a hard guarantee** (Section 3.3). Autoscaling/warm pools reduce provisioning delay but do not substitute for pre-provisioned render capacity; hard SLA and current budget appear incompatible.
3. **Chunked rendering for large documents is unverified.**
4. **The baseline API/worker node is a single point of failure.** An Auto Scaling group with min 1 replaces it in minutes, and queued jobs survive, but requests during that window fail.
5. **Single region and a single AZ for the baseline**, to stay inside budget.
6. **The budget fit is not demonstrated** (Section 8). Peak hours, spot discount, prices and renders per node are unverified. The $40 burst-compute guardrail is not a cap on the complete AWS bill, and a controller cannot guarantee the final bill. Demand beyond the cap is rejected or delayed; the hard five-second peak SLA is therefore not met.
7. **Submission recovery is designed, not tested.** The accept path (idempotency key, `ACCEPTING` → `ACCEPTED`, reconciler) has several failure windows that need fault-injection tests (kill the API between steps 4 and 6, drop SQS calls).
8. **Network isolation is weaker than ideal.** Workers sit in public subnets with no inbound access but unrestricted outbound at the network level. Compensating controls (a firewall rule for the browser's OS user, request interception) are proposed and untested (Section 8.7).
9. **Archive cost grows every year.** The totals in 8.1-8.3 are first-year figures; by year 8 the retained PDFs add roughly ₹670/month and can push the on-demand scenarios over budget (Section 8.8).

**With 2 more days**
- Run a small benchmark on the target instance: 1 browser with 1/2/4 pages at 5, 100 and 500 line items; record peak memory (cgroup), p95 latency, and crashes. Use it to fix per-node concurrency.
- Build a proof of concept: API plus one worker plus S3, and test chunked rendering and merge with a 1,000-row document.
- Load test the queue and scale-out behaviour (k6), and measure real time to first capacity.
- Run an actual cost check in the AWS Pricing Calculator.

**At 10× load (300,000 PDFs/month, 10,000 single/min peak)**
- Move workers to containers on ECS/EKS with a warm pool, and maintain a pool of pre-launched browsers.
- Run multiple AZs and an ALB with proper health checks.
- Replace Chromium for simple, table-heavy documents with a lighter PDF renderer (benchmark first), and keep Puppeteer only for complex layouts.
- Shard queues per tenant to prevent noisy neighbours; add per-tenant rate limits.
- Use spot instances for bulk and committed-use pricing for the baseline.

---

7. One Thing the Problem Statement Didn't Mention

**Security of rendering untrusted data inside a browser.** Puppeteer renders HTML built from customer-supplied data (item descriptions, notes, names). If that data is inserted into the template unescaped, it can inject HTML or scripts, and Chromium can then load `file://` paths, call internal URLs (SSRF to the EC2 metadata service), or exfiltrate data. That would turn a PDF service into a way to read credentials or other customers' data.

Mitigations: escape all data in templates; disable JavaScript where templates don't need it; block all network requests from the render browser except an allow-list (request interception); run Chromium in a locked-down container (no sandbox-disable flags, minimal IAM role, IMDSv2 enforced); isolate customers' files by tenant-specific S3 prefixes and check authorization on every download.

It matters because this is the one failure that is a security incident rather than a slow or failed PDF. Closely related and also unlisted: observability (queue age, failure rate, DLQ depth alarms), because without it the first sign of trouble would be customer complaints.

---


---

Release Decision and Acceptance Gates

This is a **design proposal, not a claim of production readiness**. The implementation is eligible for launch only after these gates are met:

1. **Peak contract confirmed:** product defines whether 1,000 single requests/minute and 5–10 bulk requests/minute occur concurrently, the duration of the peak, and whether the five-second target is a hard p95/p99 SLO or a normal-load target.
2. **Capacity measured:** benchmark the actual templates and payload distribution on the selected instance family. Replace the renders-per-node hypothesis with measured safe concurrency and retain headroom for Chromium/OS memory and CPU.
3. **End-to-end latency measured:** count a request as successful only when the artifact is downloadable. Report accepted, rejected, failed, and ready jobs separately; include p50/p95/p99 and 429 rate.
4. **Failure correctness demonstrated:** fault-inject API crashes around acceptance, worker death around upload/finalization, duplicate SQS deliveries, 500+ row OOMs, invoice #87 mutation, and interrupted ZIP downloads.
5. **Budget validated:** use a current AWS Pricing Calculator estimate for the selected region and confirm with a 30-day cost projection. Include peak node-hours, Spot interruptions, data transfer, logs, archive growth, and endpoint/networking choices.
6. **Security controls verified:** test IAM boundaries, tenant isolation, HTML escaping, browser egress restrictions, S3 Object Lock retention, and KMS signature verification.

**Decision rule:** if the measured capacity needed to meet the hard five-second peak SLO exceeds ₹12,500/month, the requirements cannot honestly be marked as jointly satisfied under the measured assumptions. The product owner must change the budget, offered peak, or SLO. The system may still be deployed in budget-controlled mode, but it must expose backpressure explicitly and must not describe that mode as meeting the hard peak SLO.


---

## 8. Cost Estimation

*Assumptions:* ₹83 per USD (the assignment's own ratio: ₹12,500 ≈ $150). Burst usage is 20 hours/month of peak *(assumption; depends on the clarifying questions)*. Deployment model: Docker containers on EC2 with Auto Scaling groups, an **internal** ALB (the ERP is in the same VPC), no NAT Gateway.

**Research references used for design constraints:** AWS documents that EC2 Auto Scaling warm pools reduce scale-out initialization latency, but they require capacity to be maintained and can add cost ([AWS warm pools](https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-warm-pools.html)). AWS S3 Object Lock uses versioning and retention controls to provide WORM protection; Compliance mode prevents deletion of protected versions during retention ([AWS Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html), [configuration requirements](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock-configure.html)). These official docs validate feature behaviour, not the prices in the estimate.


### 8.1 Line-by-line (all on-demand, conservative)

**Important consistency note:** this all-on-demand line item is a comparison case, not the permitted operating configuration: six on-demand burst nodes for 20 hours cost **$43.20 in burst compute alone**, exceeding the **$40 monthly burst-compute guardrail** in Decision 1. The controller must curtail burst capacity before that scenario is reached. The selected six-node mixed scenario in Section 8.2 is the guardrail-compatible comparison; its estimated burst compute is $36.00 for 20 hours (4 on-demand nodes × 20 h × $0.36 + 2 Spot nodes × 20 h × $0.18), before per-node IPv4/EBS charges. These are still estimates, not a verified AWS bill.
| # | Item | Calculation | USD/month | INR/month |
|---|---|---|---|---|
| 1 | Baseline node: API + controller + 3 worker slots (t3.large, 2 vCPU / 8 GB) | 720 h × $0.09 | 64.8 | 5,378 |
| 2 | Burst workers: 6 × c6i.2xlarge | 6 × 20 h × $0.36 | 43.2 | 3,586 |
| 3 | Internal Application Load Balancer | ~$16 base + ~$6 usage | 22.0 | 1,826 |
| 4 | S3, **year-1 average only** (see 8.8 for the eight-year view; Glacier Instant Retrieval after ~90 days; retrievals assumed rare) | ~90 GB × ~$0.02 | 2.0 | 166 |
| 5 | SQS | ~300k requests (within 1M free tier) | 0 | 0 |
| 6 | DynamoDB on-demand (job rows, progress, per-document state) | ~100k small requests | 1.0 | 83 |
| 7 | KMS (1 key + signing calls) | $1 key + ~$1 | 2.0 | 166 |
| 8 | CloudWatch (logs, metrics, alarms) | estimate | 6.0 | 498 |
| 9 | ECR and misc | estimate | 1.0 | 83 |
| 10 | Public IPv4 addresses (no NAT: instances need public IPs to reach SQS/DynamoDB/KMS) | (720 h + 6 × 20 h) × $0.005 | 4.2 | 349 |
| 11 | EBS gp3 volumes (baseline 30 GB; burst root volumes prorated) | ~30 GB × ~$0.09 + prorated burst | 3.0 | 249 |
| 12 | Data transfer out | ~15 GB, under the free allowance | 0 | 0 |
| | **Total (on-demand burst)** | | **≈ 149.2** | **≈ ₹12,384** |

**Budget check:** ₹12,500 − ₹12,384 ≈ **₹116 headroom (~1%)**. On these conservative on-demand figures the design barely fits, so I treat the fit as unproven. ₹12,384 is an illustrative first-year scenario, not a validated monthly bill.

**Fixed vs variable split (used by 8.2 and 8.3).** Fixed costs, everything except burst nodes, are **$105.1**: baseline node $64.8, ALB $22, S3 $2, DynamoDB $1, KMS $2, CloudWatch $6, ECR $1, baseline public IPv4 $3.6, baseline EBS $2.7. Burst nodes are variable and are charged per node-hour: compute ($0.36 on-demand, $0.18 spot) + public IPv4 ($0.005) + EBS root volume (about $0.0025, assuming a 20 GB gp3 volume at ~$0.09/GB-month) = **$0.3675 per on-demand node-hour and $0.1875 per spot node-hour**. For the all-on-demand comparison in 8.1 (6 nodes × 20 h = 120 node-hours), variable burst-node charges are $44.1, which with the fixed $105.1 gives the $149.2 above. The mixed design-point scenario in 8.2 has lower variable charges. Its burst compute is $36.00 for 20 hours (not $32.40); adding the assumed $0.90 burst IPv4/EBS charges gives $36.90 variable cost and $142.00 total with the $105.10 fixed estimate.


### 8.2 Scenarios (this does not demonstrate that the budget fits)

**Interpretation under the $40 compute guardrail:** the table compares unconstrained usage scenarios; rows whose burst-compute component exceeds $40/month are not allowed by the proposed runtime policy and would be curtailed. In particular, six all-on-demand nodes for 20 hours, eight mixed nodes for 20 hours, and six all-spot nodes for 40 hours exceed the compute guardrail even where a row’s estimated whole-bill total appears near or below ₹12,500. The guardrail is based on compute spend, while the table totals also include per-node IPv4/EBS costs.
Spot is a **scenario** with an assumed 50% discount (the real discount varies by instance type and over time). Singles are the latency-sensitive pool, so the "mixed" column keeps single burst nodes on-demand and puts only the bulk burst nodes on spot.

Envelope: ₹12,500 ÷ 83 = $150.6, minus $105.1 of fixed costs = **$45.5/month** for everything variable (burst compute, IPv4 and EBS). The **$40 burst-compute budget** (Decision 1) buys about 111 on-demand node-hours (at $0.36) or 222 spot node-hours (at $0.18, only if the 50% assumption holds) and leaves about $5.5 for per-node IPv4/EBS charges and estimate error.

Totals = fixed $105.1 + burst node-hours × ($0.3675 on-demand or $0.1875 spot); 6 nodes = 4 single + 2 batch, 8 nodes = 4 single + 4 batch:

| Burst nodes | Peak hours/month | All on-demand | Singles on-demand, bulk on spot | All spot |
|---|---|---|---|---|
| 6 (design point) | 20 | $149.2 ≈ ₹12,384 | $142.0 ≈ ₹11,786 | $127.6 ≈ ₹10,591 |
| 8 (hard cap) | 20 | $163.9 ≈ ₹13,604 (over) | $149.5 ≈ ₹12,409 | $135.1 ≈ ₹11,213 |
| 6 | 40 | $19.3 ≈ ₹16,044 (over) | $178.9 ≈ ₹14,849 (over) | $150.1 ≈ ₹12,458 |
| 8 | 40 | $222.7 ≈ ₹18,484 (over) | $193.9 ≈ ₹16,094 (over) | $165.1 ≈ ₹13,703 (over) |

**Conclusion:** the budget fit is **not yet demonstrated**. Peak hours, spot discount, renders per node, and several AWS line items are unverified. Some scenarios are arithmetically below ₹12,500, but that is not proof of an actual bill; several have very little headroom. More importantly, this burst-only cost model does not price the always-running capacity required for a hard five-second SLA at 1,000 singles/min. The rough lower bound in Section 3.3 already exceeds the budget by a large margin. Do not claim both constraints are satisfied unless the business changes the peak/SLA/budget assumptions and a verified pricing model and load test demonstrate the revised target.

**What limits the bill:** the node cap does not by itself cap monthly spend (8 on-demand nodes for 20 hours already exceeds the budget). The limit is the **monthly burst-compute guardrail of $40** in Decision 1 (about 22 full-cap hours at 6 nodes, 18.5 at 8 mixed nodes, 13.9 at 8 on-demand nodes), enforced by a controller that stops scale-outs and drains nodes when thresholds are crossed. It covers burst compute only, so it is **not a cap on the whole AWS bill**, and it is a guardrail with a safety margin, not a guarantee: the AWS bill is the source of truth. Demand above either limit gets the overload behaviour (429 for singles, queueing for bulk). The product team must confirm that trade (questions 3, 4 and 9).


8.3 Sensitivity to renders per node (72 slots, 20 peak hours, fixed costs $105.1)
| Renders per node (nodes for 72 slots) | Total, on-demand burst | Total, all-spot burst |
|---|---|---|
| 6 (12 nodes) | $19.3 ≈ ₹16,044 (over) | $150.1 ≈ ₹12,458 |
| 12 (6 nodes, hypothesis) | $149.2 ≈ ₹12,384 | $127.6 ≈ ₹10,591 |
| 18 (4 nodes) | $134.5 ≈ ₹11,164 | $120.1 ≈ ₹9,968 |


### 8.4 What is verified and what is not
- **Loosely cross-checked (EC2 only):** a reseller's *indicative* Mumbai guide lists t3.medium at about ₹3.4/h, so t3.large is about ₹6.8/h (≈ $0.082), and m6i.xlarge at about ₹16/h, so m6i.2xlarge is about ₹32/h and c6i.2xlarge (cheaper than m6i) should be a bit lower. My $0.09 and $0.36 are therefore slightly conservative. This is not an AWS price list.
- **Documented rate:** to my knowledge AWS publishes public IPv4 at $0.005 per address-hour. How many addresses are billed depends on how many instances run, which 8.1-8.3 now account for per node-hour. Confirm on the AWS pricing page.
- **Not verified:** ALB, S3 and Glacier prices (including the 128 KB minimum billable object size and transition fees), CloudWatch, KMS, EBS, the spot discount, and the exchange rate. I did not produce an AWS Pricing Calculator estimate, so the full monthly bill is unvalidated. **Check all of these in the AWS Pricing Calculator for the chosen region before submitting.**


### 8.5 Comparison and cost decisions
- Always-on peak capacity: 40 workers × ₹5/h (placeholder) × 720 h = ₹1,44,000, about 11.5× the budget. The budget allows only about ₹17.4/hour for the entire service, so this is a comparison showing that always-on peak capacity cannot fit the cap.
- No NAT Gateway (~$35+/month) and no interface endpoints. This is a cost-driven choice with a known security weakness; the network paths and alternatives are in 8.7.
- No ElastiCache: job state in DynamoDB on-demand.
- Downloads go through S3 presigned URLs, so no proxy traffic through our instances.
- Burst capacity runs only while queues are deep.


### 8.6 Other levers and risks
- Drop the ALB and use private DNS to a single API node: saves about ₹1,800, but loses health checks.
- **Graviton is not assumed.** Chromium/Puppeteer arm64 support must be checked first.
- The baseline uses a burstable t3 instance. This is acceptable because average load is only 0.7 PDFs/min, but sustained peaks drain CPU credits, which is why the burst workers use non-burstable c6i.


### 8.7 Network paths and the no-NAT choice
Public IPs are **not** assumed to be safer or cheaper by themselves. Option A below is a budget-driven choice with a known weakness, and the alternatives are priced.

**Network paths (Option A):**
| From | To | Path | Control |
|---|---|---|---|
| ERP | ALB → API | private VPC | ALB security group allows only the ERP; API security group allows only the ALB |
| API, workers | S3, DynamoDB | free **gateway endpoints** (private) | IAM role per pool; bucket policy blocks public access and non-TLS |
| API, workers | SQS, KMS, CloudWatch, ECR | public service endpoints over TLS, from the instance's public IP | least-privilege IAM; no inbound from the internet |
| Customer browser | S3 | presigned URLs over the internet (HTTPS) | short expiry; URLs issued only after authentication |
| Admin | instances | SSM Session Manager (no SSH port) | IAM |

**Options considered (prices are unverified assumptions of ~$0.01 per endpoint-hour per AZ; check the Pricing Calculator):**
| Option | Network | Added cost per month | Assessment |
|---|---|---|---|
| **A (v1)** | public subnets, public IPs, no inbound, S3/DynamoDB gateway endpoints | $4.2 public IPv4 (already in 8.1) | cheapest; workers can reach the internet at the network level |
| B | private subnets + interface endpoints for SQS, KMS, CloudWatch Logs, CloudWatch Monitoring, ECR API and ECR DKR (6) | ≈ 6 × 720 h × $0.01 ≈ $43 ≈ ₹3,600 per AZ, plus data processing | strongest isolation; pushes most scenarios in 8.2 over budget |
| C | private subnets + NAT Gateway | ≈ $35+ plus per-GB processing | simple, but similar cost to B and no per-service isolation |
| D | private subnets + endpoints for SQS and KMS only | ≈ $14 per AZ | partial: logs, ECR and SSM still need another path |

**Compensating controls for Option A (design proposals, untested):**
- No inbound from the internet to any instance; SSM instead of SSH; IMDSv2 required with hop limit 1.
- The browser runs under a separate OS user. A host firewall rule drops all outbound traffic from that user, and Puppeteer request interception blocks every request except local content. Templates and fonts are bundled, so rendering needs no network. Only the Node worker user can talk to AWS.
- Least-privilege IAM per pool (workers can write only their own prefixes and claim jobs; the API cannot read archived PDFs).

**Decision:** Option A for v1, with Option B or D as the upgrade if the compliance team requires private-only networking (question 10) and the budget allows.


### 8.8 Eight-year retention (the totals above are first-year figures)
The S3 line in 8.1 uses a year-1 average. With the placeholder 8-year retention and the midpoint of 15 GB of new PDFs per month, the stored PDFs accumulate as follows:

| After | PDFs stored (15 GB/month) | Range (3-30 GB/month) |
|---|---|---|
| 1 year | 180 GB | 36-360 GB |
| 4 years | 720 GB | 144 GB-1.44 TB |
| 8 years | 1,440 GB (≈1.44 TB) | 288 GB-2.88 TB |

**What is retained, and for how long:**
- PDFs: stored once in the Object Lock bucket for the retention period (placeholder 8 years), moved to Glacier Instant Retrieval after about 90 days.
- Snapshots: same retention period but **kept in S3 Standard**. They are small JSON objects (about 20 KB), and Glacier Instant Retrieval bills a minimum of 128 KB per object *(AWS-documented; verify)*, so they would be billed at about 6× their real size there, plus a transition fee per object. Snapshots add roughly 0.6 GB/month, about 58 GB at 8 years. Manifests are negligible.
- Bulk zips: expire after about 7 days and are not archived. Presigned links last minutes and can be re-issued.

**Steady-state monthly cost at year 8** (assumed, unverified prices: Standard ≈ $0.025/GB-month, Glacier Instant Retrieval ≈ $0.005/GB-month, ≈ $0.02 per 1,000 lifecycle transitions):
- PDFs in Standard (newest 90 days): 45 GB × $0.025 ≈ $1.1
- Snapshots in Standard: 58 GB × $0.025 ≈ $1.4
- PDFs in Glacier Instant Retrieval: 1,395 GB × $0.005 ≈ $7.0
- Lifecycle transitions (PDFs only): about 30,000/month × $0.02 per 1,000 ≈ $0.6
- **Total ≈ $10.1/month**, versus the $2.0 in 8.1: about **+$8.1 (≈ ₹670) per month by year 8**. This assumes few PDFs are under 128 KB (each such PDF is billed as 128 KB) and rare retrievals (Glacier Instant Retrieval charges per GB retrieved).
- **Rejected alternative:** putting snapshots in Glacier Instant Retrieval costs about the same once the 128 KB minimum is counted (96 months × 30,000 objects × 128 KB ≈ 360 GB billable ≈ $1.8/month, plus ~$0.6 in transitions) and saves nothing. Glacier Flexible Retrieval and Deep Archive differ in retrieval time, minimum storage duration and per-object metadata charges, so they are not drop-in substitutes; compliance must say what retrieval time is acceptable (question 6).

**Effect on the scenarios in 8.2 (20 peak hours, 6 burst nodes), year 8:**
| Burst pricing | First-year total | Year-8 total | Year-8 vs ₹12,500 |
|---|---|---|---|
| All on-demand | $149.2 ≈ ₹12,384 | $157.3 ≈ ₹13,056 | over |
| Singles on-demand, bulk on spot | $142.0 ≈ ₹11,786 | $150.1 ≈ ₹12,458 | nominally within (~₹42, negligible headroom) |
| All spot (50% off, assumed) | $127.6 ≈ ₹10,591 | $135.7 ≈ ₹11,263 | within (~10% headroom; prices unverified) |

At 30 GB/month the PDF archive roughly doubles (about $8 more per month by year 8 on these assumptions), so most year-8 scenarios go over budget.

**Conclusion:** the budget gets tighter each year as the archive grows, and the on-demand scenario that looks like it fits in year 1 does not fit by year 8. I do not claim the budget is validated.
