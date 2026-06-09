---
name: performance-optimization
description: Use when code is slow, latency is high, or memory usage is excessive — before guessing at fixes
---

# Performance Optimization

## Overview

Guessing at performance fixes wastes time and breaks things. Profile first, optimize second.

**Core principle:** NEVER optimize without measuring. Intuition about bottlenecks is almost always wrong.

**Announce at start:** "I'm using the performance-optimization skill."

## The Iron Law

```
NO OPTIMIZATION WITHOUT A MEASUREMENT BASELINE FIRST
```

If you don't have numbers before and after, you don't know if you helped.

## When to Use

- Response times are too slow
- Memory usage is growing or excessive
- CPU usage is unexpectedly high
- A specific operation is taking too long
- Scaling problems appear under load
- "This feels slow" complaints from users

## The Four Phases

### Phase 1: Establish Baseline

**Before touching any code:**

1. **Define the metric**
   - What exactly is slow? (endpoint latency, render time, batch job duration, memory at idle)
   - What is the current measurement?
   - What is the acceptable target?

2. **Create a reproducible benchmark**
   - Automated, not manual — you must be able to re-run this in 10 seconds
   - Isolate the specific operation (don't benchmark the whole app when one function is suspect)
   - Run it 3+ times to get a stable baseline (throw out cold-start outlier)

3. **Profile, don't guess**

   | Language | Profiling tool |
   |----------|---------------|
   | Python   | `cProfile`, `py-spy`, `memory_profiler` |
   | Node.js  | `--prof`, Chrome DevTools, `clinic.js` |
   | Go       | `pprof` |
   | Java/JVM | JFR, async-profiler |
   | Browser  | Chrome DevTools Performance tab |
   | DB       | `EXPLAIN ANALYZE`, slow query log |

4. **Find the actual bottleneck**
   - Where is 80%+ of the time/memory going?
   - Is it CPU, I/O, memory, or network?
   - Is it one hot path or many small ones?

   **Common traps:**
   - N+1 queries (looks fine in dev, catastrophic at scale)
   - Serialization/deserialization in a hot loop
   - Missing database indexes
   - Synchronous I/O blocking async code
   - Allocations inside tight loops (GC pressure)
   - Repeated computation of the same result

### Phase 2: Form a Hypothesis

**For each candidate bottleneck:**

1. State clearly: "I believe X is slow because Y, and fixing it should improve Z by ~N%"
2. Estimate the impact BEFORE implementing — if fixing it saves 2ms on a 2000ms operation, it's not worth doing
3. Pick the highest-impact bottleneck first

**Prioritization:**
- Database / network I/O → almost always highest impact
- Algorithmic complexity (O(n²) → O(n log n)) → high impact at scale
- Caching repeated computation → high impact if called frequently
- Memory allocation reduction → medium impact, reduces GC pauses
- Micro-optimizations (loop unrolling, bit tricks) → almost never worth it

### Phase 3: Implement One Change

**One change at a time. Always.**

1. Make the single change that addresses the bottleneck
2. Re-run your benchmark from Phase 1
3. Record: before vs. after numbers
4. Did it improve? By how much?

**If it didn't improve:**
- Re-examine your profiling — did you find the real bottleneck?
- Revert the change
- Go back to Phase 1 with fresh eyes

**Common fixes by category:**

**Database:**
```sql
-- Add missing index
CREATE INDEX idx_users_email ON users(email);

-- Avoid N+1: use JOIN or eager load
SELECT posts.*, users.name FROM posts JOIN users ON posts.user_id = users.id;
-- instead of: posts.map(p => db.query("SELECT name FROM users WHERE id = ?", p.user_id))
```

**Caching:**
```python
# Memoize pure functions
from functools import lru_cache

@lru_cache(maxsize=128)
def expensive_computation(input):
    ...
```

**Async I/O:**
```javascript
// Parallel, not sequential
const [a, b, c] = await Promise.all([fetchA(), fetchB(), fetchC()]);
// instead of: const a = await fetchA(); const b = await fetchB(); ...
```

**Algorithmic:**
```python
# O(1) lookup instead of O(n) scan
seen = set()           # not: seen = []
if item in seen: ...   # not: if item in seen: ... (list scan)
```

### Phase 4: Verify and Document

1. **Confirm the improvement is real**
   - Run benchmark 3+ times — is the improvement consistent?
   - Test with production-like data volume, not toy data
   - Check that nothing else regressed

2. **Document the result**
   ```
   Before: 450ms p99 latency
   After:  85ms p99 latency
   Change: Added index on users.email + eager-loaded associations
   ```

3. **Check for regressions**
   - Memory vs. speed tradeoffs (caching uses memory)
   - Correctness — caches need invalidation strategies
   - Complexity — is the optimization worth the added code complexity?

4. **Decide if more optimization is needed**
   - Is the target met? → Done.
   - Not yet? → Return to Phase 1 with the updated profile.
   - Diminishing returns? → Ship what you have, optimize more later if needed.

## Red Flags — STOP and Re-Profile

If you catch yourself:
- "This looks like it might be slow" → Profile it, don't assume
- "Let me cache everything" → Caching adds bugs. Only cache proven bottlenecks.
- "Rewriting in a faster language will fix it" → Your algorithm is probably the problem
- "I'll add an index on every column" → Indexes slow writes and use space
- Making the third change without measuring after each → Stop. Measure.
- Optimizing code that runs once at startup → Wrong target

## Quick Reference

| Symptom | Likely Cause | First Step |
|---------|-------------|------------|
| Slow API endpoint | N+1 queries or missing index | `EXPLAIN ANALYZE` the queries |
| High memory usage | Large objects in memory / memory leak | Heap snapshot or `memory_profiler` |
| Slow page render | Too many re-renders / large bundle | Chrome DevTools Performance tab |
| Slow batch job | Algorithmic complexity / missing bulk ops | Profile with `cProfile` or `py-spy` |
| High CPU | Tight loop / unnecessary computation | CPU profiler, find hot functions |
| Latency spikes | GC pauses / lock contention | GC logs, thread dumps |

## Related Skills

- **superpowers:systematic-debugging** — if performance problem is actually a bug
- **superpowers:test-driven-development** — write a benchmark test before optimizing
- **superpowers:verification-before-completion** — verify improvement before claiming success
