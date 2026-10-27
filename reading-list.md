# Inference

Stage 1 — Foundations (start here)

- Jay Alammar — "The Illustrated Transformer" (jalammar.github.io/illustrated-transformer/). Beginner. Builds the intuition for attention and why there's a "context" at all. Prerequisite for everything else.
- Sebastian Raschka — "Understanding and Coding the KV Cache in LLMs" (his Ahead of AI / Magazine site). Beginner→intermediate. Explains why the KV cache exists, with runnable code. The clearest on-ramp to the single most important concept for serving.

Stage 2 — Why inference is memory-bound, and why batching wins

- Horace He — "Making Deep Learning Go Brrrr From First Principles" (horace.io/brrr_intro.html). Intermediate. The compute-bound vs memory-bandwidth-bound vs overhead-bound framing. This is the "aha" that explains why batching is close to free and why decode is slow.
- Anyscale — "How Continuous Batching Enables 23x Throughput in LLM Inference" (Anyscale blog). Intermediate. The canonical explainer on continuous (a.k.a. in-flight) batching vs static batching. Directly answers "how many streams per GPU."

Stage 3 — Inference arithmetic (do the math yourself)

- kipply (Carol Chen) — "Transformer Inference Arithmetic" (kipp.ly/transformer-inference-arithmetic/). Intermediate→advanced. Sizes KV cache memory, latency, and throughput from first principles. This is the piece that would have let you actually size the GPU fleet instead of black-boxing it. If you read one thing, read this.
- Lilian Weng — "Large Transformer Model Inference Optimization" (lilianweng.github.io, Jan 2023). Advanced. Broad, rigorous survey: KV cache, quantization, distillation, sparsity, parallelism. Great map of the whole territory.

Stage 4 — Production systems & serving engines

- vLLM — "Efficient Memory Management for LLM Serving with PagedAttention" (blog at blog.vllm.ai; paper arXiv:2309.06180). Advanced. PagedAttention treats KV cache like OS virtual memory. This is the state-of-the-art batching/memory story and shows up constantly in interviews.
- NVIDIA — "Mastering LLM Techniques: Inference Optimization" (NVIDIA Developer blog). Intermediate→advanced. Vendor but genuinely good on batching, KV cache, and tensor/pipeline parallelism.
- Databricks/Mosaic — "LLM Inference Performance Engineering: Best Practices" (Databricks blog). Advanced. Roofline thinking applied to real serving; ties latency/throughput targets to hardware.
- Character.AI — "Optimizing AI Inference at Character.AI" (research.character.ai). Advanced, systems-flavored. A production war story on slashing KV-cache cost (multi-query attention, int8, cross-layer cache sharing) at ChatGPT-like scale. Closest to this exact interview problem.

Suggested order for you specifically: Raschka (KV cache) → Horace He (memory-bound) → Anyscale (continuous batching) → kipply (do the arithmetic) → vLLM + Character.AI (production). That path takes you from "I don't work with GPUs" to being able to size the fleet and discuss batching trade-offs out loud — which is exactly the gap that capped this round.

# Redis Pub-Sub, message fanout

1. Redis Pub/Sub fundamentals (start here)

- Redis Pub/Sub guide — https://redis.io/docs/latest/develop/pubsub/
  The whole model in one page: SUBSCRIBE/PUBLISH/PSUBSCRIBE, message format, pattern matching. Read the "Delivery semantics" section carefully: it's at-most-once (fire-and-forget). If no subscriber is listening, or one disconnects, the message is gone forever. That single fact drives every design decision below.
- Command refs (stable): SUBSCRIBE · PUBLISH

2. The cross-node scaling nuance (the part that matters for your question)

- Sharded Pub/Sub — same page, "Sharded Pub/Sub" section, plus SSUBSCRIBE/SPUBLISH (Redis 7.0+).
  This is the key gotcha. In a Redis Cluster, classic PUBLISH propagates every message to every node over the cluster bus — so the bus itself becomes your bottleneck exactly on hot channels. Sharded pub/sub confines a channel to one shard, so you scale fan-out by adding shards. If you take one thing from this list for the auction fan-out, it's this.

3. Pub/Sub vs. Streams — when fire-and-forget isn't enough

- Redis Streams — https://redis.io/docs/latest/develop/data-types/streams/
  Persistent append-only log, consumer groups, at-least-once with XACK, replay via XRANGE. This is what you reach for when a dropped message matters or a late joiner needs history. For your coalesced-monotonic-max display, pub/sub's loss is self-healing so you don't need this — but knowing the boundary is the point.

4. The actual WebSocket-routing pattern in practice (the reference design)

- Socket.IO Redis adapter — https://socket.io/docs/v4/redis-adapter/
  This is the canonical, production implementation of exactly what you described: multiple WebSocket server instances, each subscribes to Redis; a broadcast goes to local sockets immediately and is PUBLISHed so the other nodes deliver to their sockets. Read it even if you're not on Node — it's the pattern, concretely. Note it now recommends the sharded adapter (built on
  §2's sharded pub/sub).
- Socket.IO adapter concept — https://socket.io/docs/v4/adapter/ (why a backplane is needed at all once you have >1 server).

# Kafka and Stream Processing

4. Resources (focused, canonical)

Kafka

- Jay Kreps, "The Log: What every software engineer should know about real-time data's unifying abstraction" (2013) — the foundational mental model.
- Confluent blog: "Exactly-Once Semantics Are Possible in Kafka" — directly fixes miss #2.
- Kafka: The Definitive Guide (O'Reilly, free PDF via Confluent) — chapters on reliability & consumers.

Streaming (Flink / Kafka Streams)

- Tyler Akidau, "Streaming 101" and "Streaming 102" (O'Reilly Radar) — event-time, watermarks, windowing. This is the pair to read; it directly targets misses #1 and #5.
- Book-length version: Streaming Systems (Akidau/Chernyak/Lax).
- Flink docs → "Concepts: Timely Stream Processing" and "Working with State."

TSDB / high-cardinality OLAP (and note: the real names here are Druid, Pinot, ClickHouse, M3 — not DataFusion, which is a query-engine library)

- Apache Druid design doc / the original Druid paper — how segmented columnar + rollup serves exactly your dashboard query.
- Uber Engineering: M3 blog posts (high-cardinality metrics at scale).
- ClickHouse docs on AggregatingMergeTree — the merge-on-write rollup pattern you were groping toward.
- RedisBloom docs — for the probabilistic dedup in §1.

1. The Secret Lives of Data — Raft (thesecretlivesofdata.com/raft/). An interactive, animated walkthrough of leader election + log replication. Start here; 20 minutes and the whole thing clicks.
2. raft.github.io — the official hub. Has another live visualization (see the nodes time out, vote, replicate) and links to everything below.
3. The Raft paper: "In Search of an Understandable Consensus Algorithm" (Ongaro & Ousterhout, 2014) — read the extended version. Raft was explicitly designed to be teachable, so the paper is unusually readable.
   This is the one to actually sit with. The three subproblems — leader election, log replication, safety — are the mental model to walk away with.
4. Ongaro's talk "Designing for Understandability: The Raft Consensus Algorithm" (YouTube) — same content, narrated.
5. Go deeper if you want mastery:
   - MIT 6.824 / 6.5840 Distributed Systems (pdos.csail.mit.edu/6.824) — lectures on YouTube + labs where you implement Raft in Go. Implementing it is what makes it permanent.
   - Ongaro's PhD dissertation, "Consensus: Bridging Theory and Practice" — the exhaustive version (membership changes, log compaction, client interaction).

6. The package doc comment itself (the definitive source).
   k8s.io/client-go/tools/leaderelection — the leaderelection.go file opens with a long doc comment that explains the whole algorithm and its caveats (notably: it guarantees at most one leader only if clocks and
   lease timing assumptions hold — it's not a perfect fencing mechanism).

- Godoc: pkg.go.dev/k8s.io/client-go/tools/leaderelection
- Source: github.com/kubernetes/client-go → tools/leaderelection/leaderelection.go
- Look specifically at the tryAcquireOrRenew function — that's the heart of it.

2. The lock implementation.
   Same repo: tools/leaderelection/resourcelock/ — the LeaseLock (there are also older ConfigMap/Endpoints locks). This is where you see the actual etcd-backed mechanism: it reads/writes a Lease object and updates it
   using the API server's optimistic concurrency (resourceVersion compare-and-swap), which is exactly etcd's CAS surfaced through the Kubernetes API. That's the "atomically claim a key that expires" primitive in
   action.

3. A runnable example.
   github.com/kubernetes/client-go → examples/leader-election/main.go. Small, self-contained program that spins up a LeaderElector with a LeaseLock and prints when it becomes/loses leader. Clone it and run it against
   a local cluster — watching the Lease object update in real time is what makes it concrete.

4. The Lease object + the concept doc.

- kubernetes.io/docs/concepts/architecture/leases/ — explains how kube-controller-manager and kube-scheduler use Leases for HA leader election in the kube-system namespace.
- The API: Lease in coordination.k8s.io/v1 (API reference on kubernetes.io). Inspect a live one: kubectl -n kube-system get lease then kubectl -n kube-system get lease kube-scheduler -o yaml — you'll see
  holderIdentity, leaseDurationSeconds, renewTime. That single object is the election.

5. The tuning knobs (to understand the failure/timing model).
   The component flags — --leader-elect, --leader-elect-lease-duration, --leader-elect-renew-deadline, --leader-elect-retry-period — documented in the kube-controller-manager / kube-scheduler reference on
   kubernetes.io. Reading what these control teaches you the split-brain-vs-failover tradeoff (short lease = fast failover but more sensitive to pauses/clock skew).

The mental model to hold while reading: the leader loop is just read the Lease → is it expired? → if so, CAS-update it to me → keep renewing before it expires. etcd (via the API server) provides the atomic CAS and
the durability; leaderelection is a thin client loop on top. Same shape as the etcd concurrency.Election and ZooKeeper ephemeral-znode patterns — only the lock primitive differs.

# Algorithms

## Knapsack/subset/partition

Following links assume basic familarity with the psuedo-polynomial DP based algorithm which runs in O(nW) time.

1. Using bitset to optimize constant in runtime - https://www.electicode.com/en/resources/dynamic-programming/bitset-optimization
2. Topcode additional tricks - https://codeforces.com/blog/entry/98663
3. Pisinger's linear time algo for bounded weights - https://stackoverflow.com/questions/18821453/bounded-knapsack-special-case-small-individual-item-weight-is-small-compared-t
4. Another explanation of the above - https://chaoxu.prof/posts/2018-12-18-subset-sum-through-balancing.html
5. Full thesis by Pisinger - https://hjemmesider.diku.dk/~pisinger/95-1.pdf

## SkipList

Used in LSM-Tree based databases for memtable

1. Tutorial with practice: https://shipthatcode.com/courses/build-skip-list/lessons/skip-idea

## Tries/Radix Trees

1. https://en.wikipedia.org/wiki/Radix_tree
2. https://newsletter.kentbeck.com/p/adaptive-radix-tree
3. https://duckdb.org/2022/07/27/art-storage
4. https://www.geeksforgeeks.org/dsa/pattern-searching-using-suffix-tree/
