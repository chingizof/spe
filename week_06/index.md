# Understanding Workload Properties

## Why Workload Properties Matter

Scale does not just mean more traffic. It changes the shape of the workload. Requests become unevenly distributed. Hot spots appear. Failure modes change.

A design that works under balanced load may fail under skew. Before choosing a sharding, caching, replication, or precomputation strategy, the system’s workload needs to be understood.

Key questions:

- Are reads or writes dominant?
- Is traffic evenly distributed?
- Do a few keys receive most of the load?
- Does one request create work for many users, objects, or services?
- Can the expensive work be parallelized?
- What happens when one object, user, tenant, region, or shard becomes hot?

At small scale, spare capacity can hide these issues. At large scale, they become the design problem.

---

## Horizontal Scaling and Its Assumptions

A common scaling strategy is to split work across machines. With **horizontal sharding**, data and requests are divided across multiple servers.

With **consistent hashing**, a request key is hashed and mapped to a shard. The key might be a user ID, object ID, tenant ID, or record key.

In theory, if a system has `N` servers, each server handles about `1/N` of the total load. Add servers, reduce per-node load.

That model assumes:

- Keys are evenly distributed.
- Each key creates roughly similar work.
- No small set of keys dominates the workload.
- Expensive operations can be spread across machines.

At scale, these assumptions often fail.

---

## The Problem With Averages

Averages hide skew.

Average request rate, CPU usage, or latency may look fine while a few users, objects, tenants, partitions, or shards are overloaded.

Many systems follow a **power law distribution**:

- A few users have huge follower counts.
- A few posts receive most views.
- A few tenants generate most API traffic.
- A few products receive most purchases.
- A few files, rows, or cache keys receive most reads.
- A few regions or time windows receive most demand.

A design built around the “average” entity can work for most cases and fail on the cases that dominate load.

At scale, outliers stop being edge cases.

---

## When One Operation Creates Many Operations

**Fan-out** happens when one action creates work in many places.

Examples:

- One message sent to a large group.
- One post published to millions of followers.
- One configuration change applied across many machines.
- One cache invalidation affecting many cached objects.
- One database update triggering indexes, notifications, analytics events, and replication.

The original request may look cheap while the backend work is expensive.

A social media timeline is a useful example. Many platforms optimize for reads by precomputing timelines. When a user posts, the system copies that post into follower timelines. Reads become cheap because the timeline is already prepared.

This is **fan-out on write**.

For a median user with fewer than 50 followers, this is cheap. The post creates a small number of writes, usually handled through a queue and worker pool.

For a celebrity, the same operation can create millions of writes. The API can acknowledge the post quickly, but the backend still has to process the queue, absorb database writes, and handle downstream notifications or analytics events.

The broader issue: one external request can expand into a large amount of internal work.

---

## Why Naive Horizontal Scaling Breaks

Horizontal scaling works when work is parallelizable and evenly distributed.

If each key receives similar traffic, sharding by key works well. Each shard receives a similar number of keys, and each key creates similar work.

If a small number of keys dominate traffic, sharding creates hot spots.

For example, if one popular object maps to one shard, a spike in reads or writes can saturate that shard while the rest of the cluster remains underused.

Adding machines does not fix a workload that cannot use them.

The important question is: **Can the expensive work be distributed across the added servers?**

---

## Designing for Skewed Workloads

Skewed workloads need workload-specific paths.

Common techniques:

- Replicate hot objects across nodes.
- Cache hot data closer to users.
- Split one hot key into multiple partitions.
- Move large tenants to dedicated infrastructure.
- Push expensive work through queues.
- Use backpressure to protect shared systems.
- Serve stale data when availability matters more than freshness.
- Separate read and write paths for different traffic classes.

In the timeline example, normal accounts can use fan-out on write. Large accounts can use **fan-out on read**: store the post once, then merge it into timelines when users open the app.

That gives a hybrid design:

- Normal accounts use the cheaper path for ordinary traffic.
- Hot accounts use a path that avoids massive write amplification.
- The system identifies hot keys and handles them separately.

One universal strategy is cleaner. Multiple workload-specific paths are usually more realistic.

---

## Design Lessons From Workload Scale

Scale exposes hidden assumptions.

A design may assume uniform load, independent requests, evenly distributed keys, or predictable user behavior. Those assumptions may hold in tests and fail in production.

Main lessons:

1. Identify hot keys, users, tenants, objects, and shards.
2. Check whether the expensive work is actually parallelizable.
3. Optimize hot paths.
4. Use separate paths for different workload shapes when necessary.
5. Degrade gracefully under overload.
6. Measure distributions, not just averages.

Graceful degradation may mean serving stale data, delaying non-critical updates, reducing recommendation quality, shedding optional work, or rate-limiting expensive requests.

---

# Case Study 3: Designing a Graph-based Memory/RAG System for LLM at Meta

## Setup

The premise is straightforward. Meta has roughly 3.9 billion users, a social graph that handles around a quadrillion queries per day, and an enormous amount of personal context about every user. Other AI companies don't have any of that. So if Meta can plug its social graph into Meta AI, it has a real reason for users to choose Meta AI over ChatGPT. The product becomes defensible, it keeps users in the app, and it opens up monetization through things like restaurant bookings, local business ads, and group messaging follow-ups. The hard part is making it actually work at Meta's scale without melting the GPUs or making users wait.

To make the problem concrete, we used a single user prompt as our running example: "Give me a restaurant my friends from music club will like." Every design choice in the writeup below is measured against how fast and how cheaply we can answer that one query, multiplied by billions of users.

## The Naive Pipeline

The natural first instinct is to do everything dynamically at request time. The flow looks like this:

1. Run a full LLM inference call to extract the user's intent and entities from the prompt. This takes roughly 800ms to 1.5 seconds because it's a real model invocation.
2. Pull the user's profile data, around 5 to 10ms.
3. Resolve "music club" to an actual group. If it's an exact match, this is a fast graph lookup. If it's fuzzy ("my music friends"), it requires an embedding similarity search on the order of 100 to 200ms.
4. Get the group members. This is a single graph edge query, very fast at 2 to 10ms.
5. For each member, dynamically scan their recent posts, likes, comments, check-ins, location history, saved content, search history, and followed business pages. With 4 members this comes out to roughly 2 to 3 seconds per member.
6. Pull a list of nearby restaurants from a local business database, around 500ms.
7. Make another LLM call to rank the candidates given all the member data, 1 to 2 seconds.
8. Make a third LLM call to generate a human readable explanation of why this restaurant was chosen, about 1 second.
9. Make a fourth LLM call to suggest follow-up actions like booking or messaging the group, around 300ms.

End to end this is 7 to 9 seconds per request, with 15 plus database queries and 4 LLM calls. At Meta's scale this is not a real product. It is a demo that gets you fired.

## Where the Time Actually Goes

When you break the latency down, roughly 75% (about 6 seconds) is spent inside LLM calls. Another 20% (2 to 3 seconds) is the per-member dynamic scanning across posts, comments, and likes. The graph and database queries themselves are only about 5% of total time, on the order of tens of milliseconds. The lesson here is important: the actual graph database is fine. Meta's TAO infrastructure is genuinely fast. What's killing us is that we are making the LLM do all the thinking at query time.

## Why This Matters

Latency is not a soft constraint, it's the constraint. Amazon found in 2006 that every additional 100ms of page load time cost them 1% in sales, which would be roughly $3.8 billion at today's revenue. Google saw similar effects on search engagement. If a personalized LLM takes 3 plus seconds to respond while a generic one answers in under a second, users will leave no matter how good the personalization is. Latency is the tax users pay for intelligence, and at our scale, the tax has to be small.

The other constraint is cost. 4 LLM calls per request at billions of requests per day translates directly into GPU hours and dollars. Even a 50% reduction in LLM calls per request is a real number on Meta's infrastructure bill.

## The Core Idea: Move Work Offline

The whole optimization comes down to one principle: anything that doesn't depend on the user's specific prompt should be precomputed. The user's request path needs to be as simple and fast as possible, which means the heavy lifting has to happen offline, before the user ever hits send.

Concretely, here's what we do offline. For every new post in the system, we run a topic extraction pass using a smarter LLM in batched mode. We can afford a more expensive model here because we are amortizing the cost across time and across many posts in a single batch. From there we generate embeddings for both the post and its topics, build weighted edges between them in a topic-post graph, and run cleanup passes for deduplication and domain blocking. We add a filtering layer for legal, privacy, and low quality content. Finally, we index everything into a vector database for fast retrieval.

The result is that by the time a user query arrives, the topic-post graph is already built, the embeddings are already computed, and the social signals are already attached.

## The Online Path

When the actual prompt comes in, the flow looks completely different from the naive version:

1. Skip the expensive LLM intent parsing entirely. Use embedding similarity to compare the prompt to topics. This is roughly 300ms total, structured as a cascading filter pipeline. L1 ranking is optimized for top 100 recall and latency, and L2 ranking is optimized for the probability that an item gets selected by the LLM downstream. Cheap filter first, expensive filter on the smaller set. This is a standard pattern in recommendation systems.
2. Build a list of about 30 relevant topics from the L2 output, and in parallel pull the posts associated with each topic. Together these steps come in around 100ms, and now we have a full topic-post subgraph for this query.
3. Layer in the social signals. For every candidate post, boost its rank if it was liked or shared by a friend or someone relevant to the user. This is around 40ms.
4. Run PageRank over the subgraph instead of asking an LLM to rank, about 200ms. PageRank is the same algorithm Google built its search business on, and it's the right tool here because ranking in a recommendation system really is just graph traversal with weights. An O(n) graph algorithm beats running n tokens through a transformer almost every time.
5. Take the top 10 results from PageRank and pass only those to a single LLM call for the final answer, including the explanation. Roughly 100ms because the input size is small.

Total end to end: about 640ms. That is more than a 10x improvement over the naive pipeline, and it eliminates 3 of the 4 LLM calls.

## Trade-offs We Are Making

This system is not free, and being honest about what we gave up matters more than pretending the optimization is magic.

First, we are trading some accuracy for latency. An LLM extracts topics more precisely than embeddings do, but embeddings are fast enough that the small accuracy hit is acceptable. Good is good enough.

Second, we replaced "restaurant candidate generation" with general post ranking. There is no hard guarantee that the top posts surfaced by PageRank will actually contain restaurants. In practice we run experiments on real traffic to verify the system finds restaurants reliably, and if it doesn't, we tune the topic-post weights until it does. This is the kind of thing you have to validate empirically rather than reason about analytically.

Third, PageRank can't reason the way an LLM can. It can't look at a group's members and conclude "Jordan goes wherever the group goes." We accept this loss because the pre-ranking layer narrows the input enough that the final LLM call can pick up most of the nuance.

## The Tail Latency Story

Average latency tells you almost nothing in production. What matters is p99, because at billions of requests per day, p99 is the experience for tens of millions of users every single day. The naive pipeline's p99 is probably 15 plus seconds because LLM calls have high variance, cold caches happen, and one slow graph query can stall the whole chain. The optimized pipeline's p99 stays under 1.5 seconds because embeddings and PageRank have predictable, bounded latency. Removing LLM calls from the critical path is what makes the tail behave.

## Final System

The full architecture splits cleanly into offline and online halves. Offline we have ingestion (live updates, topic and domain selection, batching, URL normalization), filtering (legal, privacy, content quality), signal and metadata extraction (social signals attached to each post), and indexing (chunking, embedding generation, feature generation), all feeding a central indexer service. Online we have the user prompt going through Meta AI, into the search plugin, through a query rewriter and spelling layer, into the indexer service, where L1 ranking, L2 ranking, and PageRank produce a small set of relevant items, which then get passed as RAG context to the final LLM call.

## Conclusion

The big lesson here is that scale changes which optimizations matter. When you are serving 100 users a day, doing everything dynamically at query time is fine. When you are serving billions, you have to identify the most expensive operation in your pipeline (LLM calls), figure out which parts of it actually depend on the live user query, and push everything else offline where you can amortize the cost across batches and time. Replacing one of the LLM ranking calls with PageRank is not a clever trick, it's the obvious move once you accept that you cannot afford four LLM calls per request. Pre-computing topic-post graphs is not a clever trick either, it's just acknowledging that the post written 6 hours ago does not need its embedding regenerated when a new query comes in.

Good performance engineering at scale is mostly about being honest with yourself about what work actually has to happen at request time, and ruthlessly moving everything else somewhere cheaper.

---

# Observability and Debugging

---

## Observability and Debugging at Scale

Scale also changes debugging.

In a small system, debugging is local. Inspect logs, reproduce the issue, add print statements, or read a stack trace. That breaks down in large distributed systems. A production request can cross many services, queues, caches, databases, and internal APIs. Each component has its own logs, metrics, deployments, and failure modes. At that point, knowing that something failed is not enough. Engineers need to know where it failed, why it failed, and what changed.

**Monitoring** tells you something is wrong.

**Observability** helps explain why.

A dashboard showing high latency is monitoring. A trace showing that one downstream service retried a database query three times is observability.

---

## Why Observability Is Hard

Large systems produce too much behavior to inspect manually. 

**Volume**: There is too much telemetry to search directly. Logs, traces, metrics, and events become expensive to store and query.

**Aggregation**: Aggregates hide detail. Average latency may look fine while p99 latency is terrible for a small user segment.

**Cross-Service Failures**: A stack trace explains one process. It does not explain the full path of a request across services.

**Reproduction**: Production failures often depend on real traffic, timing, data distribution, cache state, deployment order, or downstream behavior. Reproducing them locally is often unrealistic.

**Cost**: Observability infrastructure has its own cost. Logs need storage. Metrics need indexing. Traces need collection and sampling. Dashboards and alerts need maintenance.

#### Observability is part of system design, not an afterthought.

---

## Common Observability Tools

Most systems use metrics, logs, and traces.

### Time-Series Monitoring

Time-series monitoring tracks values over time: request rate, error rate, CPU usage, memory usage, queue depth, and latency percentiles. These metrics usually feed dashboards and threshold alerts.

Example: alert if error rate is above 1% for five minutes or p99 latency exceeds 500 ms.

The limitation: you have to know what to measure ahead of time.

### Centralized Logging

Centralized logging sends service logs into one storage and search system.

Logs provide context: request IDs, user IDs, errors, query names, feature flags, and internal state.

The cost is storage and noise. Many logs are never read. The logs needed during an incident may be missing, sampled out, or hard to find.

### Distributed Tracing

Distributed tracing follows a request across services.

A trace ID is attached to the request and propagated across service boundaries. Each service records a span with timing and local work. The final trace shows the request path. Tracing connects isolated service behavior into one view.

---

## Bad Observability Patterns

Some observability creates data without understanding.

### Low-Cardinality Trap

Cardinality is the number of distinct values a field can have. HTTP method is low-cardinality. User ID is high-cardinality.

Low-cardinality metrics are cheap, but limited:

```text
http_requests_total{method="GET", status="200", service="checkout"}
```

This can show successful GET requests to checkout. It cannot answer richer questions:

- Which cohort is affected?
- Is the issue isolated to one region?
- Did it start after a deployment?
- Is one user, tenant, shard, or feature flag responsible?

High-cardinality data answers better questions. It is also more expensive to store and query.

### Disconnected Tools

Metrics in one system, logs in another, traces in another, and deployment data somewhere else forces manual correlation during incidents.

The key incident question is often: **What changed?**

If the observability system cannot connect symptoms to deployments, config changes, traffic shifts, or dependency failures, it leaves engineers guessing.

---

## Better Observability

Good observability is high-cardinality, high-dimensional, and explorable.

- **High-cardinality:** supports fields like user ID, tenant ID, request ID, shard ID, host ID, region, build version, and feature flag.
- **High-dimensional:** supports questions across many fields at once.
- **Explorable:** supports new questions during an incident, not just predefined dashboards.

### Structured Events

Services should emit structured events with rich context instead of disconnected metrics, logs, and traces.

Example fields:

```text
trace_id
service
endpoint
user_id
region
status_code
latency_ms
build_version
feature_flags
shard_id
error_type
```

Structured events make it easier to slice system behavior after the fact.

### Distributed Tracing as a First-Class Primitive

Tracing should be part of the request path, especially in microservice-heavy systems.

Every request should carry correlation context across service boundaries. Without that context, engineers manually connect logs from different systems.

### Dynamic Sampling

Storing everything is too expensive. Random sampling can discard the data needed most.

Dynamic sampling keeps useful data while controlling cost:

- Keep all errors.
- Keep all slow requests.
- Sample normal requests based on volume.
- Use tail-based sampling when possible.