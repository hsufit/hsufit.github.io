---
title: Python Container Data Structure and Complexity Notes
description: Notes for understanding Python containers by their underlying data structures, time complexity, and problem-solving use cases.
tags:
- python
- container
- data-structure
- complexity
- interview
---

## Question

- I wanted a Python version of the C++ container data structure and complexity note.
- The goal is to understand the cost behind common operations.
- This helps with:
  - LeetCode
  - interviews
  - algorithm analysis
  - choosing the right Python tool

## Complexity Table

| Python tool | Mental data structure | Access | Search / membership | Insert / add | Delete / remove |
|---|---|---|---|---|---|
| `list` | dynamic array | index O(1) | value O(n) | append O(1) amortized, middle O(n) | end O(1), middle O(n) |
| `tuple` | immutable array | index O(1) | value O(n) | none | none |
| `str` | immutable character sequence | index O(1) | substring search varies, often O(n) | create new string | create new string |
| `dict` | hash table | key O(1) avg | key O(1) avg | O(1) avg | O(1) avg |
| `set` | hash table | none | value O(1) avg | O(1) avg | O(1) avg |
| `deque` | block-linked deque | end O(1), middle O(n) | O(n) | both ends O(1) | both ends O(1) |
| `Counter` | dict subclass | key O(1) avg | key O(1) avg | update O(n) over input | O(1) avg per key |
| `defaultdict` | dict subclass | key O(1) avg | key O(1) avg | O(1) avg | O(1) avg |
| `heapq` | binary heap in `list` | top O(1) | O(n) | push O(log n) | pop O(log n) |

## Notes About Complexity

- `dict` and `set` are average O(1).
- Their worst case can degrade, but average O(1) is the normal mental model.
- `list.append()` is amortized O(1).
- `list.insert(0, x)` is O(n).
- `deque.appendleft()` is O(1).
- `deque.popleft()` is O(1).
- `heapq` uses a normal list as heap storage.

## list

- `list` is a dynamic array.
- Memory is array-like and cache-friendly.

```txt
[1][2][3][4][5]
```

- Good for:
  - random access
  - append at end
  - iteration
  - sorting

- Bad for:
  - insert at front
  - delete from front
  - frequent middle insert/delete

## tuple

- `tuple` is an immutable sequence.
- It is like a fixed record.

```python
point = (3, 4)
```

- Good for:
  - fixed groups of values
  - dictionary keys
  - returning multiple values

- Bad for:
  - mutation
  - append/remove

## str

- `str` is immutable.
- Every modification creates a new string.

```python
s = "abc"
s2 = s + "d"
```

- Good for:
  - text
  - slicing
  - search with `in`, `find`, `startswith`

- Bad for:
  - repeated concatenation in loops

Use this pattern for many pieces:

```python
"".join(parts)
```

## dict

- `dict` is a hash table.
- It maps key to value.

```python
d = {"a": 1, "b": 2}
```

- Good for:
  - key-value lookup
  - frequency maps
  - memo tables
  - graph adjacency maps

- Average operations:
  - lookup O(1)
  - insert O(1)
  - delete O(1)

## set

- `set` is a hash table for unique values.

```python
seen = {1, 2, 3}
```

- Good for:
  - membership test
  - deduplication
  - visited set

- Average operations:
  - membership O(1)
  - add O(1)
  - remove O(1)

## deque

- `deque` is a double-ended queue.
- It is implemented with blocks, not one flat dynamic array.

```txt
[block] <-> [block] <-> [block]
```

- Good for:
  - BFS queue
  - sliding window
  - append/pop on both ends

- Key operations:
  - `append` O(1)
  - `appendleft` O(1)
  - `pop` O(1)
  - `popleft` O(1)

## Counter

- `Counter` is a dict subclass for frequency counting.

```python
from collections import Counter

cnt = Counter(nums)
```

- Good for:
  - count frequency
  - compare multisets
  - get most common values

Useful API:

```python
cnt[x]
cnt.update(items)
cnt.most_common()
```

## defaultdict

- `defaultdict` is a dict with automatic default values.

```python
from collections import defaultdict

g = defaultdict(list)
g[u].append(v)
```

- Good for:
  - grouping
  - adjacency lists
  - counters with `int`
  - sets with `set`

## heapq

- `heapq` is a binary heap over a list.
- Python's heap is a min heap.

```python
import heapq

heapq.heappush(heap, x)
smallest = heapq.heappop(heap)
```

- Good for:
  - priority queue
  - Dijkstra
  - top-k problems

- Key operations:
  - `heap[0]` O(1)
  - `heappush` O(log n)
  - `heappop` O(log n)

## Fast Memory

- Need random access -> `list`
- Need immutable record -> `tuple`
- Need key-value lookup -> `dict`
- Need membership/dedup -> `set`
- Need queue -> `deque`
- Need frequency -> `Counter`
- Need grouping -> `defaultdict`
- Need priority queue -> `heapq`

## Final Summary

- Python's default containers are fewer than C++ STL containers.
- `list` covers dynamic array use cases.
- `dict` and `set` cover hash lookup use cases.
- `deque` covers queue use cases.
- `heapq` covers priority queue use cases.
- `Counter` and `defaultdict` cover common dict patterns.
