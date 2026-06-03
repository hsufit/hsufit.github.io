---
title: Python Algorithm and Standard Library Cheatsheet
description: Notes for quickly remembering common Python built-ins and standard-library modules used for algorithm problems.
tags:
- python
- algorithm
- standard-library
- cheatsheet
- competitive-programming
---

## Question

- I wanted a Python version of the C++ STL algorithm note.
- Python does not put all algorithms in one `<algorithm>`-like header.
- The practical tools come from:
  - built-in functions
  - list methods
  - `bisect`
  - `heapq`
  - `itertools`
  - `functools`

## Big Python Algorithm Memory Table

| Task | Python tool |
|---|---|
| sort | `sorted`, `list.sort` |
| binary search | `bisect_left`, `bisect_right` |
| min / max | `min`, `max` |
| sum | `sum` |
| all / any condition | `all`, `any` |
| frequency | `Counter` |
| heap | `heapq` |
| permutations | `itertools.permutations` |
| combinations | `itertools.combinations` |
| cartesian product | `itertools.product` |
| prefix accumulation | `itertools.accumulate` |
| memoization | `functools.cache`, `functools.lru_cache` |

## Sorting

- Return a new sorted list:

```python
arr = sorted(nums)
```

- Sort list in place:

```python
nums.sort()
```

- Reverse order:

```python
nums.sort(reverse=True)
```

- Sort by key:

```python
items.sort(key=lambda x: x[1])
```

- Sort by multiple fields:

```python
items.sort(key=lambda x: (x[0], -x[1]))
```

## Binary Search with `bisect`

- Use `bisect` on sorted lists.

```python
from bisect import bisect_left, bisect_right, insort
```

- First index where value is greater than or equal to `x`:

```python
i = bisect_left(nums, x)
```

- First index where value is greater than `x`:

```python
j = bisect_right(nums, x)
```

- Count `x`:

```python
cnt = bisect_right(nums, x) - bisect_left(nums, x)
```

- Insert while keeping sorted order:

```python
insort(nums, x)
```

## Min, Max, and Sum

- Common built-ins:

```python
min(nums)
max(nums)
sum(nums)
```

- With key:

```python
max(words, key=len)
min(points, key=lambda p: p[0] ** 2 + p[1] ** 2)
```

## Condition Checks

- All values satisfy a condition:

```python
all(x > 0 for x in nums)
```

- At least one value satisfies a condition:

```python
any(x > 100 for x in nums)
```

Memory rule:

```txt
all -> every item
any -> at least one item
```

## Counting and Frequency

- Count one value in a list:

```python
nums.count(x)
```

- Count all frequencies:

```python
from collections import Counter

cnt = Counter(nums)
```

- Most common values:

```python
cnt.most_common()
cnt.most_common(3)
```

## Heap with `heapq`

- Python heap is a min heap.

```python
import heapq

heapq.heapify(nums)
heapq.heappush(nums, x)
smallest = heapq.heappop(nums)
```

- Peek minimum:

```python
nums[0]
```

- Largest `k`:

```python
heapq.nlargest(k, nums)
```

- Smallest `k`:

```python
heapq.nsmallest(k, nums)
```

- Max heap trick:

```python
heapq.heappush(heap, -x)
largest = -heapq.heappop(heap)
```

## Itertools

- Permutations:

```python
from itertools import permutations

for p in permutations(nums):
    ...
```

- Combinations:

```python
from itertools import combinations

for c in combinations(nums, 2):
    ...
```

- Cartesian product:

```python
from itertools import product

for a, b in product(A, B):
    ...
```

- Prefix accumulation:

```python
from itertools import accumulate

prefix = list(accumulate(nums))
```

## Memoization

- Cache recursive function results:

```python
from functools import cache

@cache
def dfs(i, state):
    ...
```

- Older or size-limited version:

```python
from functools import lru_cache

@lru_cache(None)
def dfs(i):
    ...
```

## Loop Helpers

- Index and value:

```python
for i, x in enumerate(nums):
    ...
```

- Pair values:

```python
for a, b in zip(A, B):
    ...
```

- Reverse iteration:

```python
for x in reversed(nums):
    ...
```

## Top 10 for LeetCode and Interviews

| Tool | Purpose |
|---|---|
| `sorted` / `sort` | sorting |
| `bisect_left` | lower-bound style binary search |
| `bisect_right` | upper-bound style binary search |
| `Counter` | frequency counting |
| `defaultdict` | grouping / adjacency list |
| `deque` | BFS queue |
| `heapq` | priority queue |
| `enumerate` | index and value loop |
| `zip` | parallel iteration |
| `cache` / `lru_cache` | memoized recursion |

## Final Summary

- Python algorithms are remembered by module and task.
- Sorting uses `sorted` or `list.sort`.
- Binary search uses `bisect`.
- Priority queue uses `heapq`.
- Combinatorics uses `itertools`.
- Frequency uses `Counter`.
- Queue uses `deque`.
- Memoization uses `functools.cache`.
