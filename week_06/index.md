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
