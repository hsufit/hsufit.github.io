---
title: C++ STL Container Data Structure and Complexity Notes
description: Notes for understanding C++ STL containers from their underlying data structures, time complexity, and simple memory-layout diagrams.
tags:
- cpp
- stl
- data-structure
- complexity
- interview
---

## Question

- I wanted to understand STL containers from a lower-level view.
- The focus is:
  - underlying data structure
  - random access complexity
  - search complexity
  - insert complexity
  - erase complexity

- This is useful for:
  - interviews
  - algorithm contests
  - LeetCode
  - complexity analysis

## Complexity Table

| STL container | Underlying data structure | Random access | Search | Insert | Erase |
|---|---|---|---|---|---|
| `vector` | dynamic array | O(1) | O(n) | back O(1) amortized, middle O(n) | middle O(n) |
| `deque` | segmented dynamic array | O(1) | O(n) | front/back O(1), middle O(n) | front/back O(1), middle O(n) |
| `list` | doubly linked list | O(n) | O(n) | O(1) if position is known | O(1) if position is known |
| `set` | red-black tree | none | O(log n) | O(log n) | O(log n) |
| `multiset` | red-black tree | none | O(log n) | O(log n) | O(log n) |
| `map` | red-black tree | none | O(log n) | O(log n) | O(log n) |
| `multimap` | red-black tree | none | O(log n) | O(log n) | O(log n) |
| `unordered_set` | hash table | none | O(1) average, O(n) worst | O(1) average, O(n) worst | O(1) average, O(n) worst |
| `unordered_map` | hash table | none | O(1) average, O(n) worst | O(1) average, O(n) worst | O(1) average, O(n) worst |
| `priority_queue` | binary heap | `top` O(1) | none | O(log n) | O(log n) |
| `stack` | adapter, `deque` by default | `top` O(1) | none | O(1) | O(1) |
| `queue` | adapter, `deque` by default | none | none | O(1) | O(1) |

## Notes About Complexity

- `vector` back insertion is amortized O(1).
- `vector` may reallocate when capacity is full.
- `unordered_*` operations are average O(1).
- `unordered_*` operations can become O(n) in the worst case.
- `list` insert/erase is O(1) only when the iterator position is already known.
- Finding that position is still O(n).
- `priority_queue::pop()` removes the top element and re-heapifies in O(log n).

## vector

- `vector` is a dynamic array.
- Memory is contiguous.

```txt
[1][2][3][4][5]
```

- Mental model:

```cpp
T* data;
size_t size;
size_t capacity;
```

- Strength:
  - fast random access
  - cache-friendly
  - simple memory layout

- Weakness:
  - slow middle insertion
  - slow front insertion
  - possible reallocation

## deque

- `deque` is a segmented dynamic array.
- It supports efficient operations at both ends.

```txt
map
 |
 v
+----+----+----+
| *  | *  | *  |
+----+----+----+
  |    |    |
  v    v    v
[1 2 3][4 5 6][7 8]
```

- Mental model:

```cpp
T** blocks;
```

- It uses:
  - fixed-size blocks
  - an index table to those blocks

- Strength:
  - O(1) push at front
  - O(1) push at back
  - random access is still O(1)

- Weakness:
  - not fully contiguous like `vector`
  - more complex memory layout

## list

- `list` is a doubly linked list.
- Each node points to the previous and next node.

```txt
NULL
 |
 v
[A] <-> [B] <-> [C]
                 |
                 v
               NULL
```

- Node model:

```cpp
struct Node {
  Node* prev;
  Node* next;
  T value;
};
```

- Strength:
  - O(1) insert after known position
  - O(1) erase at known position
  - stable node addresses

- Weakness:
  - no random access
  - poor cache locality
  - search is O(n)

## set and multiset

- `set` and `multiset` are usually implemented as red-black trees.
- They are balanced binary search trees.

```txt
        8
      /   \
     4     12
    / \    / \
   2   6  10 14
```

- `set` stores unique keys.
- `multiset` allows duplicate keys.

- Strength:
  - sorted order
  - O(log n) search
  - O(log n) insert
  - O(log n) erase
  - supports `lower_bound` and `upper_bound`

- Weakness:
  - no random access
  - slower than hash table for pure lookup on average

## map and multimap

- `map` and `multimap` are also usually red-black trees.
- They store key-value pairs.

```txt
        (8,A)
       /     \
   (4,B)   (12,C)
```

- Node value model:

```cpp
pair<const Key, T>
```

- `map` stores unique keys.
- `multimap` allows duplicate keys.

- Strength:
  - sorted key-value lookup
  - O(log n) search
  - O(log n) insert
  - O(log n) erase
  - supports `lower_bound` and `upper_bound`

- Weakness:
  - no random access
  - `map::operator[]` inserts a missing key
  - `multimap` has no `operator[]`

## unordered_set

- `unordered_set` is a hash table.
- It stores keys in buckets.

```txt
bucket
 0 -> NULL
 1 -> 21 -> 31
 2 -> 12
 3 -> 43 -> 13
```

- Mental model:
  - array of buckets
  - hash function
  - collision handling

- Strength:
  - average O(1) lookup
  - average O(1) insert
  - average O(1) erase

- Weakness:
  - no sorted order
  - worst-case O(n)
  - no `lower_bound`
  - no `upper_bound`

## unordered_map

- `unordered_map` is a hash table with key-value pairs.

```txt
bucket
 |
 v
(10,A)
 |
 v
(20,B)
```

- Node value model:

```cpp
pair<const Key, T>
```

- Strength:
  - average O(1) key-value lookup
  - useful for frequency counting
  - useful for memoization

- Weakness:
  - no sorted order
  - worst-case O(n)
  - no range query by key order

## priority_queue

- `priority_queue` is usually a binary heap.

```txt
        100
       /   \
     50     70
    / \    /
   20 10 30
```

- Actual storage is usually an array:

```txt
[100, 50, 70, 20, 10, 30]
```

- Mental model:

```txt
priority_queue<int> = vector<int> + heap operations
```

- Strength:
  - `top()` is O(1)
  - `push()` is O(log n)
  - `pop()` is O(log n)

- Weakness:
  - cannot search efficiently
  - cannot iterate in sorted order directly
  - only exposes the top element

## stack

- `stack` is a container adapter.
- The default underlying container is usually `deque`.

```txt
top
 |
 v
[1][2][3]
```

- It only exposes:

```cpp
push()
pop()
top()
```

- Mental model:

```txt
stack = LIFO
```

- LIFO means last in, first out.

## queue

- `queue` is a container adapter.
- The default underlying container is usually `deque`.

```txt
front -> [1][2][3] <- back
```

- It only exposes:

```cpp
push()
pop()
front()
back()
```

- Mental model:

```txt
queue = FIFO
```

- FIFO means first in, first out.

## Interview Memory Version

```txt
vector         = Dynamic Array
deque          = Segmented Array
list           = Doubly Linked List

set            = Red-Black Tree
multiset       = Red-Black Tree

map            = Red-Black Tree KV
multimap       = Red-Black Tree KV

unordered_set  = Hash Table
unordered_map  = Hash Table KV

priority_queue = Binary Heap

stack          = deque adapter
queue          = deque adapter
```

## Fast Complexity Memory

- `vector`
  - O(1) random access
  - O(1) amortized back insertion
  - O(n) middle insertion/erase

- `deque`
  - O(1) random access
  - O(1) front/back insertion
  - O(n) middle insertion/erase

- `list`
  - O(1) insert/erase if position is known
  - O(n) search

- `set` / `map`
  - O(log n) search
  - O(log n) insert
  - O(log n) erase

- `unordered_*`
  - O(1) average search
  - O(1) average insert
  - O(1) average erase
  - O(n) worst case

- `priority_queue`
  - O(1) top
  - O(log n) push
  - O(log n) pop

## Final Summary

- STL complexity becomes easier when tied to data structure.
- `vector` is a dynamic array.
- `deque` is a segmented array.
- `list` is a doubly linked list.
- `set` and `map` are red-black trees.
- `unordered_set` and `unordered_map` are hash tables.
- `priority_queue` is a binary heap.
- `stack` and `queue` are adapters.
- For most algorithm problems, remember:
  - `vector` has O(1) random access
  - `deque` has O(1) front/back operations
  - `list` has O(1) known-position insert/erase
  - `set` and `map` are O(log n)
  - `unordered_*` is O(1) average
  - `priority_queue` has O(log n) push/pop
