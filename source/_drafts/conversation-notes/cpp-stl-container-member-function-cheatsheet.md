---
title: C++ STL Container Member Function Cheatsheet
description: Notes for quickly remembering common C++ STL container member functions by grouping containers by their underlying data structure and operation patterns.
tags:
- cpp
- stl
- data-structure
- cheatsheet
- competitive-programming
---

## Question

- I wanted a fast way to remember C++ STL member functions.
- The goal is not to memorize the full reference.
- The goal is to remember enough for:
  - coding interviews
  - competitive programming
  - LeetCode
  - Codeforces
  - quick problem solving

- The better memory method is:
  - group containers by data structure
  - remember each container's ability
  - map the ability to member functions

## Core Idea

- Do not memorize STL as a flat function list.
- Memorize it as container families.
- Each family answers four questions:
  - how to insert
  - how to erase
  - how to access
  - how to search

## Common Functions Shared by Many Containers

- Many STL containers support:

```cpp
size()
empty()
clear()
begin()
end()
insert()
erase()
```

- Fast memory:
  - number of elements -> `size()`
  - is empty -> `empty()`
  - remove all -> `clear()`
  - iterate -> `begin()`, `end()`
  - add/remove -> `insert()`, `erase()`

## Big STL Memory Table

| Operation | Dynamic Array | Double-ended Dynamic Array | Doubly Linked List | Ordered Tree | Ordered Tree KV | Hash Table | Hash Table KV | Heap | Adapter | Adapter |
|---|---|---|---|---|---|---|---|---|---|---|
| STL container | `vector` | `deque` | `list` | `set` / `multiset` | `map` / `multimap` | `unordered_set` | `unordered_map` | `priority_queue` | `stack` | `queue` |
| Insert | `push_back` | `push_back`, `push_front` | `push_back`, `push_front` | `insert` | `insert`, `operator[]` | `insert` | `insert`, `operator[]` | `push` | `push` | `push` |
| Erase | `pop_back` | `pop_back`, `pop_front` | `pop_back`, `pop_front`, `remove` | `erase` | `erase` | `erase` | `erase` | `pop` | `pop` | `pop` |
| Access | `[]`, `at`, `front`, `back` | `[]`, `front`, `back` | `front`, `back` | none | `[]` | none | `[]` | `top` | `top` | `front`, `back` |
| Search | linear search | linear search | linear search | `find` | `find` | `find` | `find` | none | none | none |
| Count | none | none | none | `count` | `count` | `count` | `count` | none | none | none |
| Range query | none | none | none | `lower_bound`, `upper_bound` | `lower_bound`, `upper_bound` | none | none | none | none | none |
| Iterator | random access | random access | bidirectional | bidirectional | bidirectional | forward | forward | none | none | none |
| Special feature | `resize`, `reserve` | front operations | `sort`, `reverse`, `unique` | sorted keys | key-value | average O(1) lookup | average O(1) lookup | max first | LIFO | FIFO |

## Linear Containers

- The memory chain:

```txt
vector -> deque -> list
```

| Operation | Dynamic Array | Double-ended Dynamic Array | Doubly Linked List |
|---|---|---|---|
| STL container | `vector` | `deque` | `list` |
| Insert at back | `push_back` | `push_back` | `push_back` |
| Delete at back | `pop_back` | `pop_back` | `pop_back` |
| Insert at front | none | `push_front` | `push_front` |
| Delete at front | none | `pop_front` | `pop_front` |
| Access first | `front` | `front` | `front` |
| Access last | `back` | `back` | `back` |
| Random access | `operator[]`, `at` | `operator[]`, `at` | none |
| Resize | `resize` | `resize` | `resize` |
| Reserve capacity | `reserve` | none | none |
| Remove by value | none | none | `remove` |
| Sort inside container | use `std::sort` | use `std::sort` | `sort` |
| Reverse inside container | use `std::reverse` | use `std::reverse` | `reverse` |
| Remove consecutive duplicates | use algorithm after sorting | use algorithm after sorting | `unique` |
| Iterator | random access | random access | bidirectional |
| Main strength | cache-friendly indexed array | both-end operations | cheap insert/delete after position |
| Main weakness | slow front insert/delete | more complex than `vector` | no random access |

- Memory rule:
  - append at tail -> `push_back`
  - remove tail -> `pop_back`
  - first element -> `front`
  - last element -> `back`
  - indexed access -> `[]` or `at`
  - `vector` + front operations = `deque`
  - `deque` - random access focus + linked-list operations = `list`
  - linked-list-only value removal -> `remove`
  - linked-list-only internal sort -> `sort`
  - linked-list-only internal reverse -> `reverse`
  - linked-list-only consecutive duplicate removal -> `unique`

## Ordered Set Family

- `set` and `multiset` are ordered tree containers.

| Operation | Ordered Tree | Ordered Tree with Duplicates |
|---|---|---|
| STL container | `set` | `multiset` |
| Stores | unique keys | duplicate keys |
| Insert | `insert` | `insert` |
| Erase | `erase` | `erase` |
| Search | `find` | `find` |
| Count | `count` returns `0` or `1` | `count` can return more than `1` |
| Lower bound | `lower_bound` | `lower_bound` |
| Upper bound | `upper_bound` | `upper_bound` |
| Random access | none | none |
| Iterator | bidirectional | bidirectional |
| Ordering | sorted by key | sorted by key |
| Typical complexity | O(log n) | O(log n) |
| Main strength | unique sorted keys | sorted keys with duplicates |
| Main weakness | no direct index access | no direct index access |

- Fast memory:
  - insert -> `insert`
  - erase -> `erase`
  - find value -> `find`
  - count existence -> `count`
  - range query -> `lower_bound`, `upper_bound`

- Most competitive programming cases use:

```cpp
insert()
erase()
find()
lower_bound()
```

## Ordered Map Family

- `map` and `multimap` are ordered key-value tree containers.
- Think:

```txt
map = set + value
```

| Operation | Ordered Tree KV | Ordered Tree KV with Duplicates |
|---|---|---|
| STL container | `map` | `multimap` |
| Stores | unique key-value pairs | duplicate-key key-value pairs |
| Insert | `insert`, `operator[]` | `insert` |
| Erase | `erase` | `erase` |
| Search | `find` | `find` |
| Count | `count` returns `0` or `1` | `count` can return more than `1` |
| Lower bound | `lower_bound` | `lower_bound` |
| Upper bound | `upper_bound` | `upper_bound` |
| Value access by key | `operator[]`, `at` | none |
| Random access | none | none |
| Iterator | bidirectional | bidirectional |
| Ordering | sorted by key | sorted by key |
| Typical complexity | O(log n) | O(log n) |
| Main strength | sorted key-value lookup | sorted duplicate-key records |
| Main weakness | `operator[]` inserts missing keys | no `operator[]` |

- Common problem-solving patterns:

```cpp
mp[x]++;
mp[x]--;

if (mp.count(x)) {
  // key exists
}

auto it = mp.find(x);
```

## set vs map

| Function / Idea | `set` | `map` |
|---|---|---|
| `insert` | yes | yes |
| `erase` | yes | yes |
| `find` | yes | yes |
| `count` | yes | yes |
| `lower_bound` | yes | yes |
| `upper_bound` | yes | yes |
| stores | key only | key-value |
| common access | value itself | `mp[key]` |

- Memory rule:
  - `set` = key only
  - `map` = key + value

## Hash Family

- Hash containers are unordered versions of set/map.
- Main containers:

```cpp
unordered_set
unordered_map
```

- Common functions are similar to ordered containers:

```cpp
insert()
erase()
find()
count()
operator[] // unordered_map only
```

## Ordered vs Hash

| Feature | Ordered | Hash |
|---|---|---|
| set type | `set` | `unordered_set` |
| map type | `map` | `unordered_map` |
| sorted | yes | no |
| `lower_bound` | yes | no |
| `upper_bound` | yes | no |
| typical complexity | O(log n) | average O(1) |

- Memory rule:

```txt
unordered_xxx = xxx - sorted order + hash lookup
```

## Adapters

- Adapters expose a small interface.
- Most problems only need a few functions.

| Container | Main Idea | Insert | Remove | Access |
|---|---|---|---|---|
| `stack` | LIFO | `push` | `pop` | `top` |
| `queue` | FIFO | `push` | `pop` | `front`, `back` |
| `priority_queue` | heap | `push` | `pop` | `top` |

- Memory rule:
  - `stack` -> last in, first out
  - `queue` -> first in, first out
  - `priority_queue` -> largest element at `top` by default

## 90 Percent Competitive Programming Table

| Category | Must-remember functions |
|---|---|
| `vector` | `push_back()`, `pop_back()`, `size()`, `begin()`, `end()` |
| `deque` | `push_front()`, `push_back()`, `pop_front()`, `pop_back()` |
| `set` | `insert()`, `erase()`, `find()`, `lower_bound()` |
| `map` | `operator[]`, `find()`, `count()` |
| `unordered_map` | `operator[]`, `find()`, `count()` |
| `stack` | `push()`, `pop()`, `top()` |
| `queue` | `push()`, `pop()`, `front()` |
| `priority_queue` | `push()`, `pop()`, `top()` |

## Final Summary

- Remember the container's data structure first.
- Then remember its ability.
- Then map ability to member functions.
- `vector` is the dynamic array baseline.
- `deque` extends `vector` with front operations.
- `list` is the linked-list version.
- `set` stores ordered keys.
- `map` stores ordered key-value pairs.
- `unordered_*` removes ordering and uses hash lookup.
- `stack`, `queue`, and `priority_queue` are small adapters.
- After solving 20 to 30 problems, these member functions become natural.
