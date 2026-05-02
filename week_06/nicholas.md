# Case Study #1

## The Scenario

Imagine you're a software engineer working at Instagram, and you and a few coworkers
are working on a new feature called **Feature "Z"**.

Everything was passing the unit tests and the metrics looked good, but in production
the code crashed in 36 hours with a test failing...

You and your coworker go and investigate.

---

## The Problem

Your coworker has found the piece of code that's failing, but they don't know why —
so they came to you! Here's what the code does:

- Your system processes user IDs from `0` to `4,294,967,295` (`uint32` limit).
- We want to ensure that the feature works for **all users**.
- Your coworker wrote a function which takes in a user ID and tests the feature on them.
- But testing it on each user's profile might be infeasible (4 billion is a large number).

> **Q: How can we be confident that our feature works correctly for all users without testing every possible user ID?**

**A:** Use **random sampling** — generate uniformly distributed random IDs to verify the algorithm works, and assume it works on a larger scale.

Your coworker said that they ran their sample on **10 million random IDs**, and no tests failed. You decide to look at their unit test.

```cpp
#include <cstdlib>  // Has rand & srand
#include <cstdint>  // For uint32_t
#include <ctime>    // To randomly seed RNG
#include <cassert>  // For assertions

// Random testing function
void test() {
    std::srand(static_cast<unsigned>(std::time(nullptr))); // seed RNG

    // # of random samples we want
    const int NUM_TESTS = 1000000;

    // Iterate
    for (int i = 0; i < NUM_TESTS; i++) {
        uint32_t user_id = static_cast<uint32_t>(std::rand()); // Generate random uuid
        assert(testFeature(user_id)); // Assert test passed
    }
}
```

---

## The Root Cause

Here is the underlying `rand()` implementation your coworker was relying on:

```cpp
static uint32_t state = 1;  // internal hidden state which determines output!

// Dummy srand function — Seeds random function
void srand(uint32_t seed) {
    state = seed;
}

// Dummy rand function
int rand() {
    // Changes state
    state = 1664525 * state + 1013904223;

    // Return pseudo random number based off state variable
    return (state >> 16) & 0x7FFF;
}
```

> **Does anyone see a potential problem with this?**

### Problem with `rand()`

- If the **state variable repeats**, so will all of the subsequent numbers — this creates a **loop!**
- Every time you call the function, there's a ~`1 / 2^32` chance of state repeating.
- If you call the function `n` times, there's a ~`n² / 2^33` chance of at least 1 repetition (the **Birthday Paradox**).
- This means that on average, after **~65,536 calls**, you will have a repetition.

> So instead of testing it on 10 million users, we actually only tested on **~65,536!**

---

## Lesson: Testing at Scale is Hard!

> *This `#include` wasn't designed for scale.*

Systems often behave differently when pushed to extreme **sizes, durations, or loads**. Libraries are not magic — they have limits, assumptions, and failure modes, something which becomes more apparent when testing at scale.

---

## Q: What Makes a Good Random Number Generator?

- ✅ No period (no cyclic repetitions)
- ✅ Uniformity
- ✅ Independence across threads/machines
- ✅ Thread safety
- ✅ Reproducibility

### You May Have Seen:

```cpp
std::mt19937      // For 32-bit output
std::mt19937_64   // For 64-bit output
```

| Property | Status |
|---|---|
| Period of `2^19937 - 1` | ✅ |
| Very small amount of bias | ✅ |
| Reproducible via a seed | ✅ |
| Thread safe | ❌ |
| Thread/machine independent | ❌ |

> **Q: How can we make it thread safe & thread/machine independent?**

---

## Types of Random Distributions

| Type | Description |
|---|---|
| **Uniform** | Evenly distributed values |
| **Skewed / Hot** | A small subset of values occur more frequently |
| **Sequential** | Sequential values |
| **Zipfian** | A subset of skewed, where frequency is proportional to a power law |
| **Adversarial** | Values chosen to trigger the worst-case behavior of your system |

---

## What Should You Test?

It depends on the system, but you should ideally test **all of them**:

- **Best Case** — *"Did we accidentally make the fast path slow?"*
- **Average Case** — *"What is the typical user experience like?"*
- **Tail Case (slowest 1%)** — *"Is the slowest 1% an outlier?"*
- **Worst Case** — *"At what point will our systems crash?"*

> All of these cases can lead to different mindsets for optimizations!

---

# Case Study #2

## The Scenario

Imagine you're a software engineer at **Cloudflare**. Your team ships a new feature in a high-throughput request pipeline (thousands of requests/sec). Every request is temporarily held in a buffer until it can be processed.

The live system goes up and runs for a few days without a hitch.

Then one of your servers receives a **sudden burst of requests**!

---

## The Virtuous Cycle (Normal Operation)

Under normal conditions, the system operates in a healthy loop:

```
Thousands of Requests Incoming
        ↓  [Sent to]
Thousands of Requests Get Processed
        ↓  [Makes Room For]
Thousands of Requests Incoming  ↺
```

---

## The Problem

After the burst, in production you notice:

- 📈 **p99 latency jumps**
- 🔄 **Timeouts trigger retries**
- ❌ **Error rate spikes**
- 😡 **Angry customers**

> **Q: You take a look at the heap usage on the affected machine — what might be happening at those red points, and what impact could it have on the system?**

*(The heap chart shows repeated sharp spikes followed by drops, with an eventual plateau as the system degrades.)*

---

## The Typical Cycle (With a Garbage Collector)

Normally, memory is managed by a garbage collector (GC) in a three-step cycle:

```
Thousands of Requests Incoming
        ↓  [Sent to]
Thousands of Requests Get Processed
        ↓  [Freed By]
Garbage Collector
        ↓  [Clears memory for]
Thousands of Requests Incoming  ↺
```

---

## Garbage Collector Thrashing

When a **large burst of requests** arrives, the cycle breaks down:

**Stage 1:** Requests can't be sent for processing fast enough.

```
Thousands of Requests Incoming
+ A Large Burst of Requests
        ✗ [Can't send to processor fast enough]
Thousands of Requests Get Processed
        ↓  [Freed By]
Garbage Collector
        ↓  [Clears memory for]
Thousands of Requests Incoming  ↺
```

**Stage 2:** Users experience failures and retry, making things worse.

```
Thousands of Requests Incoming
+ A Large Burst of Requests
+ User Retries          ← [Users Retry]
        ✗ [Blocked]
Thousands of Requests Get Processed
        ↓  [Freed By]
Garbage Collector
        ↓  [Clears memory for]
Thousands of Requests Incoming  ↺
```

**Stage 3:** The GC itself gets overwhelmed — it can no longer free memory fast enough to keep up.

```
Thousands of Requests Incoming
+ A Large Burst of Requests
        ✗ [Blocked]
Thousands of Requests Get Processed
        ✗ [Can't be freed fast enough]
Garbage Collector
        ✗ [Can't clear memory fast enough]
Thousands of Requests Incoming  ↺
```

> **The GC can become a bottleneck for your program and can create a vicious cycle when dealing with systems at scale!**

---

## Q&A

> **Q: How might you try to recover from an overloaded system, and how might you prevent something like this from happening in the first place?**

### Recovery

- **Rate Limit:** Drop non-critical requests
- **Back off retries** — don't let user retries compound the load

### Prevention

- Allocate less per message / reduce object lifetime
- Bound queues tighter than existing limits, and reject requests earlier
- Increase GC headroom / buy more RAM
- Build a better garbage collector

---

> **Q: It's tempting to "just restart the server" — but what issues could that cause?**

Restarting fixes the heap issue on one server, but it will **add load to other servers**, potentially causing them to GC thrash as well — leading to **cascading failures** across your fleet.

If you do need to restart, make sure it's **gradual**, or done at a time when traffic is unlikely to spike (e.g., late at night).

---

## A Recent Improvement

A relatively new GC was added to **Go** back in 2025, reportedly saving up to **40% of the time spent in garbage collection**!

It achieves this by using a different algorithm that:

- ✅ Runs **concurrently** with your program
- ✅ Makes better use of **microarchitectural improvements**
- ✅ Uses **vector acceleration**

All of which adds up to improved performance at scale!