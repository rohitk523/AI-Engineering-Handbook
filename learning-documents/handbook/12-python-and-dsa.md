# 12 — Python and DSA Brush-Up for Backend and AI Roles

[← 11 Landscape](11-genai-landscape-2026.md) · [Index](00-INDEX.md) · Next: [13 System design →](13-system-design-for-ai.md)

**Short answer to "do I need DSA?"** Yes, a focused refresh. Senior AI/backend interviews usually include 1–2 coding rounds at LeetCode **medium** level, plus a "practical coding" round (like your cafe-billing task: a tool loop, an API, a parser) and system design. You don't need competitive-programming hards. You do need fluent Python and the core patterns.

---

## Part A: Python you must know cold

### Data model and built-ins
- Complexity of `list` (append O(1) amortized, `insert(0)` / `pop(0)` O(n)), `dict`/`set` (O(1) average), `collections.deque` (O(1) both ends), `heapq` (min-heap, O(log n)), `bisect`.
- `collections`: `Counter`, `defaultdict`, `deque`, `OrderedDict` (LRU caches), `namedtuple`.
- `dataclasses` (`frozen=True`, `slots=True`) for immutable value objects; **Pydantic** at boundaries (validation of external input).
- Comprehensions, generators (`yield`, lazy pipelines, memory efficiency), `itertools` (`groupby`, `chain`, `islice`, `product`), `functools` (`lru_cache`, `cache`, `partial`, `reduce`).
- Context managers (`with`, `contextlib.contextmanager`, `asynccontextmanager`), decorators (with `functools.wraps`), exceptions (custom hierarchies, `raise ... from ...`).
- Mutability traps: mutable default arguments, aliasing lists, shallow vs deep copy.

### Typing (expected in modern codebases)
`list[int]`, `X | None`, `TypedDict`, `Protocol` (structural interfaces), `Generic`/`TypeVar` (and the 3.12 `class Box[T]:` syntax), `Literal`, `Annotated` (used heavily by FastAPI), `Self`. Run **mypy** or **pyright**.

### Concurrency (heavily tested for backend roles)
| Tool | Use for | Why |
|---|---|---|
| `asyncio` | Many concurrent I/O waits (LLM APIs, HTTP, DB) | Single thread, cooperative; `await` yields control |
| `threading` / `ThreadPoolExecutor` | Blocking I/O libraries | Threads release the GIL during I/O |
| `multiprocessing` / `ProcessPoolExecutor` | CPU-bound work (image preprocessing) | Separate interpreters, no GIL contention |
| Free-threaded Python (3.13t / 3.14t) | Experimental/optional builds without the GIL | Becoming practical; check library support before relying on it |

asyncio essentials:
```python
import asyncio

async def fetch_all(urls: list[str]) -> list[str]:
    sem = asyncio.Semaphore(10)                       # bound concurrency (provider rate limits)
    async with httpx.AsyncClient(timeout=30) as client:
        async def one(url: str) -> str:
            async with sem:
                r = await client.get(url)
                r.raise_for_status()
                return r.text
        async with asyncio.TaskGroup() as tg:         # structured concurrency (3.11+)
            tasks = [tg.create_task(one(u)) for u in urls]
    return [t.result() for t in tasks]
```
Know: `gather` vs `TaskGroup`, `asyncio.timeout()`, cancellation, `Queue` producer/consumer, and **never block the loop** ([06 §3](06-fastapi-fundamentals.md)).

### Tooling
`uv` (fast package/project manager, now the common default), `ruff` (lint + format), `pytest` (fixtures, parametrize, `monkeypatch`), `mypy`/`pyright`, `pre-commit`, Docker.

### Internals interviewers like
GIL (what it is, why it matters less for I/O), reference counting + cyclic GC, `is` vs `==`, small int caching, how `dict` keeps insertion order, `__slots__`, descriptors and properties, MRO with multiple inheritance, `*args/**kwargs`, closures and late binding in loops.

---

## Part B: DSA, the patterns that cover most interviews

Learn the **pattern**, then do 2–4 problems each. About 60–80 problems total is enough for a strong refresh.

| # | Pattern | Recognize it when | Representative problems |
|---|---|---|---|
| 1 | Hash map / counting | "find pair", "duplicates", "anagram" | Two Sum, Group Anagrams, Top K Frequent |
| 2 | Two pointers | Sorted array, pairs, in-place | 3Sum, Container With Most Water |
| 3 | Sliding window | Longest/shortest subarray or substring with a condition | Longest Substring Without Repeating, Min Window Substring |
| 4 | Prefix sums | Range sums, subarray sum = k | Subarray Sum Equals K |
| 5 | Stack / monotonic stack | Matching brackets, next greater element | Valid Parentheses, Daily Temperatures |
| 6 | Binary search (incl. on the answer) | Sorted data, "minimum capacity such that..." | Search Rotated Array, Koko Eating Bananas |
| 7 | Linked lists | Pointer manipulation | Reverse List, LRU Cache (dict + doubly linked list) |
| 8 | Trees (DFS/BFS) | Hierarchies | Level Order, Validate BST, Lowest Common Ancestor |
| 9 | Heaps | Top-k, merge k, streaming median | Kth Largest, Merge K Sorted Lists, Median from Data Stream |
| 10 | Graphs (BFS/DFS, topological sort) | Dependencies, grids, shortest unweighted path | Number of Islands, Course Schedule, Rotting Oranges |
| 11 | Union-Find | Connectivity, grouping | Number of Provinces, Redundant Connection |
| 12 | Shortest paths (Dijkstra) | Weighted graphs | Network Delay Time |
| 13 | Intervals | Overlaps, scheduling | Merge Intervals, Meeting Rooms II |
| 14 | Backtracking | All combinations/permutations | Subsets, Combination Sum, Word Search |
| 15 | Dynamic programming (1D/2D) | Optimal count/score with overlapping subproblems | Climbing Stairs, Coin Change, LCS, Edit Distance |
| 16 | Tries | Prefix search, autocomplete | Implement Trie, Word Search II |

**Why these matter in AI/backend work, not only in interviews:** topological sort = agent/workflow DAGs and task dependencies; heaps = top-k retrieval and schedulers; LRU = caches; graphs = knowledge graphs and dependency resolution; sliding window = rate limiters; edit distance = fuzzy matching; tries = autocomplete and tokenizer tries; ANN search = vector DBs.

### "Practical coding" round problems (increasingly common for AI roles)
Practice these in 20–45 minutes each, **without AI assistance**:
- Write a tool-calling agent loop against an OpenAI-compatible API ([02](02-tool-calling-and-agent-loop.md)).
- Implement an LRU cache and a TTL cache.
- Implement a token-bucket rate limiter (then make it work across processes with Redis).
- Write a retry decorator with exponential backoff and jitter.
- Build a small FastAPI CRUD service with validation and tests.
- Chunk a document by headings with a max token size and overlap.
- Implement BM25 or cosine-similarity top-k with NumPy.
- Parse a messy log/CSV file and aggregate stats.
- Implement an async producer/consumer with bounded concurrency.

---

## Part C: Backend fundamentals checklist

- **HTTP**: methods, idempotency (GET/PUT/DELETE idempotent, POST not), status codes, headers, caching (ETag, Cache-Control), CORS.
- **REST design**: resources, pagination (cursor vs offset), filtering, versioning, error envelopes.
- **SQL**: joins, GROUP BY, window functions, indexes (B-tree, when they're used), EXPLAIN, transactions + isolation levels, N+1 queries, migrations (Alembic).
- **Postgres specifics**: JSONB, pgvector, connection pooling (PgBouncer), row-level security.
- **Redis**: caching, TTLs, rate limiting, pub/sub, streams, distributed locks (and their caveats).
- **Queues**: at-least-once delivery → idempotent consumers; dead-letter queues; retries.
- **Distributed systems basics**: CAP intuition, eventual consistency, horizontal scaling, stateless services, load balancing, timeouts everywhere.
- **Observability**: structured logs, metrics, traces (OpenTelemetry).
- **Security**: [07](07-fastapi-auth.md), OWASP Top 10.

---

## Part D: 4-week drill plan

| Week | DSA (≈5 hrs) | Python/backend (≈4 hrs) |
|---|---|---|
| 1 | Patterns 1–6 (≈20 problems) | Collections, generators, typing, pytest |
| 2 | Patterns 7–11 (≈20 problems) | asyncio deep dive, write the async fetcher and rate limiter |
| 3 | Patterns 12–16 (≈20 problems) | SQL + Postgres + Redis exercises; FastAPI CRUD with tests |
| 4 | 2 timed mock interviews + revisit misses | 4 practical-coding problems from the list above, timed, no AI |

Use NeetCode 150 / Blind 75 style lists for problem selection. Practice explaining your approach and complexity **out loud**.
