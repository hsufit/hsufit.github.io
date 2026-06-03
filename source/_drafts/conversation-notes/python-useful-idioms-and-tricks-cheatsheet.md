---
title: Python Useful Idioms and Tricks Cheatsheet
description: Practical Python idioms for fast recall, including zip/unzip, star unpacking, double-star mapping expansion, comprehensions, slicing, and loop helpers.
tags:
- python
- idioms
- cheatsheet
- unpacking
- interview
---

## Question

- I wanted a Python version of useful tips similar to the C++ STL notes.
- Some tips are not exactly containers or algorithms.
- They are Python expression patterns.
- One important example is `zip` and unzip.
- Another important example is `*` and `**` expansion.

## zip and unzip

- `zip` combines multiple iterables element by element.

```python
names = ["a", "b"]
scores = [10, 20]

pairs = list(zip(names, scores))
```

- Result:

```python
pairs == [("a", 10), ("b", 20)]
```

- Use `zip(*pairs)` to unzip.

```python
pairs = [("a", 10), ("b", 20)]

names, scores = zip(*pairs)
```

- Result shape:

```python
names == ("a", "b")
scores == (10, 20)
```

- Memory rule:

```txt
zip(a, b)  -> combine
zip(*data) -> unzip
```

## Star and Double-Star Unpacking

- Python has two related unpacking operators.
- `*` works with positional values.
- `**` works with keyword or mapping values.
- Mapping expansion means turning a dict-like object into keyword arguments.

Memory rule:

```txt
*  -> expand or collect positional values
** -> expand or collect keyword/mapping values
```

## Expand Positional Arguments with `*`

- `*iterable` expands positional arguments.
- Use it when a function expects separate positional parameters.
- But the data is already stored in a list or tuple.

```python
def add(a, b):
    return a + b

nums = [1, 2]
add(*nums)
```

- This call is equivalent to:

```python
add(1, 2)
```

## Expand Keyword Arguments with `**`

- `**mapping` expands keyword arguments.
- Use it when a function expects named parameters.
- But the data is already stored in a dict.

```python
def connect(host, port):
    ...

config = {"host": "localhost", "port": 5432}
connect(**config)
```

- This call is equivalent to:

```python
connect(host="localhost", port=5432)
```

## Collect Arguments with `*args` and `**kwargs`

- `*args` collects positional arguments in a function definition.
- `**kwargs` collects keyword arguments in a function definition.

```python
def log_event(*args, **kwargs):
    print(args)
    print(kwargs)

log_event("login", user="hsufit", success=True)
```

- Result shape:

```python
args == ("login",)
kwargs == {"user": "hsufit", "success": True}
```

## Dictionary Merging with `**`

- `**` can also unpack dictionaries into a new dictionary.

```python
base = {"host": "localhost"}
override = {"port": 5432}

config = {**base, **override}
```

- Later keys override earlier keys.

```python
{**{"x": 1}, **{"x": 2}}  # {"x": 2}
```

## enumerate

- `enumerate` gives index and value together.

```python
for i, x in enumerate(nums):
    print(i, x)
```

- Use it instead of manually maintaining an index.

## Comprehensions

- List comprehension:

```python
squares = [x * x for x in nums]
```

- Set comprehension:

```python
unique_lengths = {len(s) for s in words}
```

- Dict comprehension:

```python
index = {value: i for i, value in enumerate(nums)}
```

- Filter inside comprehension:

```python
evens = [x for x in nums if x % 2 == 0]
```

## Slicing

- Copy list:

```python
copy = nums[:]
```

- Reverse list:

```python
rev = nums[::-1]
```

- Take every second element:

```python
every_two = nums[::2]
```

- Slice range:

```python
middle = nums[l:r]
```

## Sorting with Key

- Sort by derived value:

```python
words.sort(key=len)
```

- Sort by multiple fields:

```python
items.sort(key=lambda x: (x[0], -x[1]))
```

- Memory rule:

```txt
key= tells Python what value to compare by
```

## dict.get and setdefault

- Use `get` for a fallback value.

```python
cnt[x] = cnt.get(x, 0) + 1
```

- Use `setdefault` to initialize a missing key.

```python
groups.setdefault(k, []).append(v)
```

- In many cases, `defaultdict` is cleaner for repeated grouping.

```python
from collections import defaultdict

groups = defaultdict(list)
groups[k].append(v)
```

## Swap and Multiple Assignment

- Swap two values:

```python
a, b = b, a
```

- Return or assign multiple values:

```python
x, y = point
```

## Boolean Shortcuts

- Empty containers are falsy.

```python
if not nums:
    ...
```

- Non-empty containers are truthy.

```python
if nums:
    ...
```

- Use `all` and `any` for condition checks.

```python
all(x > 0 for x in nums)
any(x == target for x in nums)
```

## Final Summary

- `zip(a, b)` combines iterables.
- `zip(*pairs)` unzips pairs.
- `*` is for positional values.
- `**` is for keyword or mapping values.
- `*iterable` expands a list or tuple into positional arguments.
- `**mapping` expands a dict into keyword arguments.
- `*args` collects positional arguments.
- `**kwargs` collects keyword arguments.
- `{**a, **b}` merges dictionaries, with later keys winning.
- `enumerate` gives index and value.
- comprehensions build collections compactly.
- slicing is the quick tool for copy, reverse, and subranges.
