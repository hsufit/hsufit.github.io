---
title: Python Container Interface Cheatsheet
description: Notes for quickly remembering common Python container operations by grouping containers by their role and operation patterns.
tags:
- python
- container
- cheatsheet
- data-structure
- interview
---

## Question

- I wanted a Python version of the C++ STL container interface note.
- Python does not have one STL namespace.
- The useful tools are spread across:
  - built-in containers
  - built-in functions
  - `collections`
  - `heapq`

## Core Idea

- Do not memorize every method.
- Remember the container role first.
- Then map the operation to the Python API.

Questions to ask:

- add value?
- remove value?
- access value?
- search value?
- count value?
- iterate?

## Big Python Container Memory Table

| Operation | Dynamic Array | Immutable Sequence | String | Hash Map | Hash Set | Deque | Frequency Map | Default Map | Heap |
|---|---|---|---|---|---|---|---|---|---|
| Python tool | `list` | `tuple` | `str` | `dict` | `set` | `collections.deque` | `collections.Counter` | `collections.defaultdict` | `heapq` over `list` |
| Add | `append`, `extend`, `insert` | none | create new string | `d[k] = v`, `update` | `add`, `update` | `append`, `appendleft` | `update` | auto-create on access | `heappush` |
| Remove | `pop`, `remove`, `clear` | none | create new string | `pop`, `del`, `clear` | `remove`, `discard`, `pop` | `pop`, `popleft` | `subtract`, `del` | `pop`, `del` | `heappop` |
| Access | `a[i]` | `a[i]` | `s[i]` | `d[k]`, `get` | membership only | ends with `dq[0]`, `dq[-1]` | `cnt[x]` | `d[k]` | `heap[0]` |
| Search | `x in a` | `x in a` | substring or char `in` | `k in d` | `x in s` | `x in dq` | `x in cnt` | `k in d` | none |
| Count | `count` | `count` | `count` | usually manual | membership only | `count` | `cnt[x]` | usually manual | none |
| Iterate | `for x in a` | `for x in a` | `for ch in s` | `keys`, `values`, `items` | `for x in s` | `for x in dq` | `items`, `most_common` | `items` | iterate list order, not sorted order |
| Sort | `sort`, `sorted` | `sorted` | `sorted` | sorted keys/items | sorted values | `sorted` | `most_common` | sorted keys/items | heap order only |
| Best use | general sequence | fixed record | text | key-value lookup | membership/dedup | queue/BFS | frequency counting | grouping/default values | min-priority queue |

## Common Built-in Operations

- Most Python containers support:
  - number of elements -> `len(x)`
  - empty check -> `if not x`
  - membership -> `in`
  - iteration -> `for item in x`

- Many mutable containers support:
  - remove all -> `clear()`
  - remove one -> `pop()`

- Special cases:
  - `tuple` and `str` are immutable.
  - `heapq` is a module, not a container class.
  - `Counter` and `defaultdict` are `dict`-like tools from `collections`.

## Sequence Family

| Operation | `list` | `tuple` | `str` |
|---|---|---|---|
| Mutable | yes | no | no |
| Access by index | `a[i]` | `a[i]` | `s[i]` |
| Slice | `a[l:r]` | `a[l:r]` | `s[l:r]` |
| Append | `append` | none | create new string |
| Extend | `extend` | none | create new string |
| Remove | `pop`, `remove` | none | create new string |
| Count | `count` | `count` | `count` |
| Find index | `index` | `index` | `find`, `index` |
| Sort | `sort`, `sorted` | `sorted` | `sorted` |

Memory rule:

- `list` = mutable dynamic array.
- `tuple` = immutable sequence.
- `str` = immutable text sequence.

## Mapping and Set Family

| Operation | `dict` | `set` | `Counter` | `defaultdict` |
|---|---|---|---|---|
| Main role | key-value lookup | membership and dedup | frequency map | dict with default factory |
| Add | `d[k] = v` | `add` | `update` | access creates default |
| Remove | `pop`, `del` | `remove`, `discard` | `subtract`, `del` | `pop`, `del` |
| Access | `d[k]`, `get` | membership only | `cnt[x]` | `d[k]` |
| Search | `k in d` | `x in s` | `x in cnt` | `k in d` |
| Iterate | `keys`, `values`, `items` | values | `items`, `most_common` | `items` |

Memory rule:

- `dict` = key -> value.
- `set` = unique keys only.
- `Counter` = value -> frequency.
- `defaultdict` = key -> auto-created default value.

## Queue and Heap Family

| Operation | `deque` | `heapq` |
|---|---|---|
| Main role | queue / stack / BFS | min-priority queue |
| Push back | `append` | `heappush` |
| Push front | `appendleft` | none |
| Pop back | `pop` | none |
| Pop front | `popleft` | none |
| Peek | `dq[0]`, `dq[-1]` | `heap[0]` |
| Best use | FIFO queue | smallest item first |

Memory rule:

- BFS queue -> `deque`.
- Min heap -> `heapq`.
- Max heap -> push negative values or use custom wrapper.

## Final Summary

- Python containers are remembered by role.
- `list` is the default sequence.
- `dict` is the default key-value lookup.
- `set` is the default membership/dedup tool.
- `deque` is the default queue.
- `Counter` is the default frequency map.
- `defaultdict` is the default grouping map.
- `heapq` is the default priority queue tool.
