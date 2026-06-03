---
title: C++ STL Algorithm Cheatsheet
description: Notes for quickly remembering common C++ STL algorithms from algorithm and numeric headers, grouped by common problem-solving use cases.
tags:
- cpp
- stl
- algorithm
- cheatsheet
- competitive-programming
---

## Question

- I wanted a quick list of commonly used C++ STL algorithms.
- The focus is practical usage for:
  - interviews
  - LeetCode
  - Codeforces
  - competitive programming
  - daily C++ development

- Most algorithms are from:

```cpp
#include <algorithm>
```

- Numeric helpers are from:

```cpp
#include <numeric>
```

## Sorting

### `sort`

- Sort a range.
- Usually implemented as introsort.
- Common use:

```cpp
sort(v.begin(), v.end());
```

- Descending order:

```cpp
sort(v.begin(), v.end(), greater<int>());
```

- Custom comparator:

```cpp
sort(v.begin(), v.end(), [](int a, int b) {
  return a % 10 < b % 10;
});
```

### `stable_sort`

- Stable sorting.
- Keeps the original relative order of equal elements.

```cpp
stable_sort(v.begin(), v.end());
```

### `partial_sort`

- Sort only the first `k` elements.
- Useful when only the smallest or largest `k` elements matter.

```cpp
partial_sort(v.begin(), v.begin() + k, v.end());
```

## Binary Search

- These functions require a sorted range.
- If the range is not sorted, the result is not meaningful.

### `binary_search`

- Check whether an element exists.

```cpp
bool found = binary_search(v.begin(), v.end(), x);
```

### `lower_bound`

- Find the first position where the value is greater than or equal to `x`.

```cpp
auto it = lower_bound(v.begin(), v.end(), x);
```

- Example:

```txt
v = [1, 2, 2, 2, 5]
lower_bound(v.begin(), v.end(), 2) -> first 2
```

### `upper_bound`

- Find the first position where the value is greater than `x`.

```cpp
auto it = upper_bound(v.begin(), v.end(), x);
```

### `equal_range`

- Get both lower and upper bounds.

```cpp
auto [l, r] = equal_range(v.begin(), v.end(), x);
```

- Count occurrences:

```cpp
int cnt = r - l;
```

## Max and Min

### `max` / `min`

- Compare two values.

```cpp
int a = max(x, y);
int b = min(x, y);
```

### `max_element`

- Find the iterator to the maximum element.

```cpp
auto it = max_element(v.begin(), v.end());
```

### `min_element`

- Find the iterator to the minimum element.

```cpp
auto it = min_element(v.begin(), v.end());
```

### `minmax_element`

- Find both minimum and maximum elements in one call.

```cpp
auto [mn, mx] = minmax_element(v.begin(), v.end());
```

## Counting

### `count`

- Count elements equal to a value.

```cpp
int cnt = count(v.begin(), v.end(), 5);
```

### `count_if`

- Count elements satisfying a condition.

```cpp
int cnt = count_if(v.begin(), v.end(), [](int x) {
  return x % 2 == 0;
});
```

## Finding

### `find`

- Find the first element equal to a value.

```cpp
auto it = find(v.begin(), v.end(), x);
```

### `find_if`

- Find the first element satisfying a condition.

```cpp
auto it = find_if(v.begin(), v.end(), [](int x) {
  return x > 100;
});
```

## Unique

### `unique`

- Remove consecutive duplicates.
- It does not erase elements from the container by itself.
- For full deduplication, sort first, then erase the tail.

```cpp
sort(v.begin(), v.end());

v.erase(unique(v.begin(), v.end()), v.end());
```

- Example:

```txt
1 2 2 2 3 3 5
|
v
1 2 3 5
```

## Reverse and Rotate

### `reverse`

- Reverse a range.

```cpp
reverse(v.begin(), v.end());
```

### `rotate`

- Move the middle iterator to the front.

```cpp
rotate(v.begin(), v.begin() + k, v.end());
```

- Example:

```txt
1 2 3 4 5
k = 2
3 4 5 1 2
```

## Permutation

### `next_permutation`

- Generate the next lexicographical permutation.
- Usually start from sorted order.

```cpp
sort(v.begin(), v.end());

do {
  // use v
} while (next_permutation(v.begin(), v.end()));
```

### `prev_permutation`

- Generate the previous lexicographical permutation.

```cpp
prev_permutation(v.begin(), v.end());
```

## Merge

### `merge`

- Merge two sorted ranges.
- The output range must have enough space.

```cpp
merge(a.begin(), a.end(),
      b.begin(), b.end(),
      c.begin());
```

## Set Operations

- These require sorted input ranges.

### `set_union`

- Compute union.

```cpp
set_union(a.begin(), a.end(),
          b.begin(), b.end(),
          back_inserter(c));
```

### `set_intersection`

- Compute intersection.

```cpp
set_intersection(a.begin(), a.end(),
                 b.begin(), b.end(),
                 back_inserter(c));
```

### `set_difference`

- Compute difference.

```cpp
set_difference(a.begin(), a.end(),
               b.begin(), b.end(),
               back_inserter(c));
```

## Numeric Algorithms

### `accumulate`

- Sum values.
- Requires:

```cpp
#include <numeric>
```

- Sum:

```cpp
int sum = accumulate(v.begin(), v.end(), 0);
```

- Product:

```cpp
int product = accumulate(v.begin(), v.end(), 1, multiplies<int>());
```

## Condition Checks

### `all_of`

- Check whether all elements satisfy a condition.

```cpp
bool ok = all_of(v.begin(), v.end(), [](int x) {
  return x > 0;
});
```

### `any_of`

- Check whether at least one element satisfies a condition.

```cpp
bool ok = any_of(v.begin(), v.end(), [](int x) {
  return x > 100;
});
```

### `none_of`

- Check whether no element satisfies a condition.

```cpp
bool ok = none_of(v.begin(), v.end(), [](int x) {
  return x < 0;
});
```

## Heap Algorithms

### `make_heap`

- Turn a range into a max heap.

```cpp
make_heap(v.begin(), v.end());
```

### `push_heap`

- Add a new element to an existing heap.

```cpp
v.push_back(x);
push_heap(v.begin(), v.end());
```

### `pop_heap`

- Move the heap top to the end.
- Then remove it from the container.

```cpp
pop_heap(v.begin(), v.end());
v.pop_back();
```

## Top 10 for Competitive Programming

| Algorithm | Purpose |
|---|---|
| `sort` | sorting |
| `lower_bound` | binary search, first `>= x` |
| `upper_bound` | binary search, first `> x` |
| `binary_search` | check existence |
| `max_element` | find maximum |
| `min_element` | find minimum |
| `count` | count value |
| `find` | find value |
| `unique` | deduplicate after sorting |
| `next_permutation` | enumerate permutations |

## Fast Memory Map

- Sort:
  - `sort`
  - `stable_sort`
  - `partial_sort`

- Binary search:
  - `binary_search`
  - `lower_bound`
  - `upper_bound`
  - `equal_range`

- Min/max:
  - `min`
  - `max`
  - `min_element`
  - `max_element`
  - `minmax_element`

- Count/find:
  - `count`
  - `count_if`
  - `find`
  - `find_if`

- Modify order:
  - `unique`
  - `reverse`
  - `rotate`
  - `next_permutation`
  - `prev_permutation`

- Sorted-range operations:
  - `merge`
  - `set_union`
  - `set_intersection`
  - `set_difference`

- Numeric:
  - `accumulate`

- Conditions:
  - `all_of`
  - `any_of`
  - `none_of`

- Heap:
  - `make_heap`
  - `push_heap`
  - `pop_heap`

## Final Summary

- Most useful STL algorithms can be remembered by task.
- Sorting starts with `sort`.
- Binary search starts with `lower_bound` and `upper_bound`.
- Deduplication is usually `sort` plus `unique` plus `erase`.
- Permutation enumeration uses `next_permutation`.
- Set operations require sorted ranges.
- Numeric accumulation uses `accumulate`.
- Heap algorithms operate on a normal container, usually `vector`.
- For interviews and contests, the top 10 algorithms cover most daily usage.
