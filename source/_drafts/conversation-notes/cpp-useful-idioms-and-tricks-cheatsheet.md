---
title: C++ Useful Idioms and Tricks Cheatsheet
description: Practical C++ idioms for fast recall, including auto, range-based loops, structured binding, emplace, erase-remove, lambda comparators, min-heaps, and fast I/O.
tags:
- cpp
- idioms
- cheatsheet
- competitive-programming
- interview
---

## Question

- I wanted a C++ version of useful Python-style idioms and tricks.
- This topic is separate from:
  - container member functions
  - container data structures and complexity
  - STL algorithms

- This note is for expression-level and coding-pattern-level recall.
- The default style is C++17.
- C++17 matters because structured binding is very useful.

## `auto` and Type Inference

- Use `auto` when the type is obvious from the right-hand side.
- It is especially useful for iterators and STL return types.

```cpp
auto it = mp.find(x);
auto n = v.size();
```

- Use explicit types when the type carries important meaning.
- Avoid using `auto` if it hides a surprising copy or conversion.

## `const auto&` to Avoid Copies

- Use `const auto&` when iterating large objects.
- This avoids copying each element.

```cpp
for (const auto& item : items) {
    // no copy
}
```

- For small values like `int`, plain `auto` is fine.

```cpp
for (auto x : nums) {
    // copy is cheap
}
```

## Range-Based `for`

- Use range-based `for` for direct iteration.

```cpp
for (int x : nums) {
    ...
}
```

- Use reference when modifying values.

```cpp
for (auto& x : nums) {
    x *= 2;
}
```

- Use `const auto&` when reading large elements.

```cpp
for (const auto& s : words) {
    ...
}
```

## Structured Binding

- Structured binding is available from C++17.
- It is useful for pairs, tuples, maps, and returned multiple values.

```cpp
for (auto [key, value] : mp) {
    // use key and value
}
```

- Use `const auto&` to avoid copying each pair.

```cpp
for (const auto& [key, value] : mp) {
    // no pair copy
}
```

- Memory rule:

```txt
auto [a, b] = pair_or_tuple;
```

## `pair` and `tuple` Unpacking

- `pair` is useful for two related values.

```cpp
pair<int, int> p = {3, 4};
auto [x, y] = p;
```

- `tuple` is useful for more than two values.

```cpp
tuple<int, string, double> t = {1, "a", 3.14};
auto [id, name, score] = t;
```

## `tie`

- `tie` can unpack values.

```cpp
int x, y;
tie(x, y) = p;
```

- `tie` can also create lexicographical comparisons.

```cpp
return tie(a.x, a.y) < tie(b.x, b.y);
```

- With C++17, structured binding is usually clearer for unpacking.
- `tie` is still useful for comparison logic.

## Lambda Comparator Patterns

- Use lambda comparators for custom sorting.

```cpp
sort(v.begin(), v.end(), [](const auto& a, const auto& b) {
    return a.second < b.second;
});
```

- Sort by multiple fields.

```cpp
sort(v.begin(), v.end(), [](const auto& a, const auto& b) {
    if (a.first != b.first) return a.first < b.first;
    return a.second > b.second;
});
```

- Shorter tuple-style comparison:

```cpp
sort(v.begin(), v.end(), [](const auto& a, const auto& b) {
    return tie(a.first, a.second) < tie(b.first, b.second);
});
```

## `emplace_back` and `emplace`

- `emplace_back` constructs an element in place.
- It is often cleaner than constructing a temporary object first.

```cpp
vector<pair<int, int>> v;
v.emplace_back(1, 2);
```

- For maps and sets:

```cpp
mp.emplace(key, value);
s.emplace(x);
```

- Memory rule:

```txt
push -> insert an existing object
emplace -> construct in place
```

## Erase-Remove Idiom

- `remove` only moves unwanted values to the end.
- It does not shrink the container.
- Use `erase` after `remove`.

```cpp
v.erase(remove(v.begin(), v.end(), x), v.end());
```

- For condition-based removal:

```cpp
v.erase(remove_if(v.begin(), v.end(), [](int x) {
    return x < 0;
}), v.end());
```

- Memory rule:

```txt
remove/remove_if + erase = real deletion from vector
```

## Iterator Erase While Looping

- When erasing while iterating, use the iterator returned by `erase`.

```cpp
for (auto it = s.begin(); it != s.end(); ) {
    if (*it < 0) it = s.erase(it);
    else ++it;
}
```

- This pattern avoids using an invalidated iterator.

## `iota` for Initializing Ranges

- `iota` fills a range with increasing values.
- It comes from `<numeric>`.

```cpp
vector<int> idx(n);
iota(idx.begin(), idx.end(), 0);
```

- Useful for:
  - index arrays
  - permutation bases
  - DSU parent initialization

## Optional `all(x)` Macro-Style Shorthand

- Competitive programming often uses shorthand for full ranges.

```cpp
#define all(x) (x).begin(), (x).end()
```

- Then:

```cpp
sort(all(v));
reverse(all(v));
```

- This is optional.
- It is convenient in contests.
- In production code, explicit `begin()` and `end()` are usually clearer.

## Fast I/O

- Useful for competitive programming.

```cpp
ios::sync_with_stdio(false);
cin.tie(nullptr);
```

- Put it near the start of `main`.

```cpp
int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    ...
}
```

## `using` Aliases

- Use `using` to simplify long types.

```cpp
using ll = long long;
using pii = pair<int, int>;
using Graph = vector<vector<int>>;
```

- Prefer `using` over old-style `typedef`.

## Vector Initialization Patterns

- Fixed size with default value:

```cpp
vector<int> v(n, 0);
```

- 2D vector:

```cpp
vector<vector<int>> grid(n, vector<int>(m, 0));
```

- From initializer list:

```cpp
vector<int> v = {1, 2, 3};
```

## Sorting Pairs and Structs

- Pairs sort lexicographically by default.

```cpp
vector<pair<int, int>> v;
sort(v.begin(), v.end());
```

- Custom struct sorting:

```cpp
struct Node {
    int x;
    int y;
};

sort(nodes.begin(), nodes.end(), [](const Node& a, const Node& b) {
    if (a.x != b.x) return a.x < b.x;
    return a.y < b.y;
});
```

## Min-Heap with `priority_queue`

- Default `priority_queue<int>` is a max heap.
- Use `greater<int>` for min heap.

```cpp
priority_queue<int, vector<int>, greater<int>> pq;
```

- For pairs:

```cpp
priority_queue<
    pair<int, int>,
    vector<pair<int, int>>,
    greater<pair<int, int>>
> pq;
```

## Common Map and Set Lookup Idioms

- Check existence:

```cpp
if (mp.count(key)) {
    ...
}
```

- Find and use iterator:

```cpp
auto it = mp.find(key);
if (it != mp.end()) {
    cout << it->second;
}
```

- Insert if not exists:

```cpp
mp.emplace(key, value);
```

- Frequency counting:

```cpp
mp[x]++;
```

- Set lower bound:

```cpp
auto it = s.lower_bound(x);
```

## Final Summary

- C++ useful tricks are mostly expression and pattern tools.
- `auto` keeps iterator-heavy code readable.
- `const auto&` avoids unnecessary copies.
- structured binding makes pairs and maps easier to read.
- lambdas are the main custom comparator tool.
- `emplace` constructs values in place.
- erase-remove is the common vector deletion pattern.
- `iota` builds index ranges.
- fast I/O matters in contests.
- min heap requires a custom `priority_queue` type.
