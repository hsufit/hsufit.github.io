---
title: Python Property Descriptor and Attribute Tracking Notes
description: Notes about printing and checking Python attribute changes with __setattr__, property, and descriptors.
tags:
- python
- descriptor
- property
- class
- oop
---

## Question

- I wanted to know how to print changes and check content whenever a Python object changes.
- I also remembered a special syntax:
  - it is written inside a class
  - it is not inside another `def`
  - but behind the syntax, it is still related to a function
- The likely concepts are:
  - `__setattr__`
  - `property`
  - descriptor

## Main Idea

- If I want to catch every attribute assignment, use `__setattr__`.
- If I want one field to behave like an attribute but run getter/setter logic, use `property`.
- If I want reusable field behavior across many attributes or classes, use a descriptor.

## Catch Every Attribute Change with __setattr__

- `__setattr__` is called whenever an attribute is assigned.
- It is useful when I want to print or validate every change.
- The formal behavior is described in the Python data model reference listed in the References section.

```python
class Person:
    def __setattr__(self, key, value):
        print(f"{key} changed to {value}")
        super().__setattr__(key, value)
```

- Example usage:

```python
p = Person()
p.name = "Tom"
p.age = 18
```

- Output:

```txt
name changed to Tom
age changed to 18
```

## Validate Before Setting

- `__setattr__` can also check values before saving them.

```python
class Person:
    def __setattr__(self, key, value):
        if key == "age" and value < 0:
            raise ValueError("age cannot be negative")

        print(f"{key} -> {value}")
        super().__setattr__(key, value)
```

- This is good for broad object-level tracking.
- It can intercept many fields without writing a separate setter for each one.

## property

- `property` lets a method behave like an attribute.
- It is usually written with decorators:
  - `@property`
  - `@name.setter`
- The related video in the References section is the original link for this `property` and descriptor discussion.

```python
class Person:
    def __init__(self):
        self._name = ""

    @property
    def name(self):
        print("read name")
        return self._name

    @name.setter
    def name(self, value):
        print(f"name changed to {value}")
        self._name = value
```

- Example usage:

```python
p = Person()
p.name = "Tom"
print(p.name)
```

- Output:

```txt
name changed to Tom
read name
Tom
```

## Why property Looks Special

- `@property` is written inside the class body.
- It is not written inside another function.
- But it decorates a function and turns it into a class attribute.
- That class attribute controls attribute access.

```python
class A:
    @property
    def x(self):
        return 123
```

- `x` looks like a normal attribute from outside:

```python
a = A()
print(a.x)
```

- But internally, `x` is backed by a function.

## Descriptor

- A descriptor is a lower-level Python attribute protocol.
- An object is a descriptor if its class defines one or more of:
  - `__get__`
  - `__set__`
  - `__delete__`
- The Python Descriptor HOWTO in the References section is the best next reading for this mechanism.

```python
class MyDescriptor:
    def __get__(self, instance, owner):
        print("get")
        return 123

    def __set__(self, instance, value):
        print("set", value)
```

- Put the descriptor on a class:

```python
class A:
    x = MyDescriptor()
```

- Use it:

```python
a = A()
print(a.x)
a.x = 10
```

- Output:

```txt
get
123
set 10
```

## How Descriptor Lookup Works

- When Python evaluates:

```python
a.x
```

- Python does not always directly read a stored value.
- It first checks whether `x` on the class is a descriptor.
- Roughly:
  - look at `A.__dict__["x"]`
  - if it has `__get__`, call `x.__get__(a, A)`
  - if assignment happens and it has `__set__`, call `x.__set__(a, value)`

## property Is a Built-in Descriptor

- `property` is implemented using the descriptor protocol.
- This means:
  - descriptor is the lower-level mechanism
  - `property` is a convenient built-in descriptor

```python
class A:
    @property
    def x(self):
        return 123
```

- This is conceptually similar to:

```python
class PropertyLike:
    def __init__(self, fget):
        self.fget = fget

    def __get__(self, instance, owner):
        return self.fget(instance)


class A:
    def x(self):
        return 123

    x = PropertyLike(x)
```

- The real `property` also supports:
  - setter
  - deleter
  - docstring

## property vs Descriptor

| Concept | Best For | Main Idea |
|---|---|---|
| `property` | one field on one class | make methods look like attributes |
| descriptor | reusable field behavior | define attribute access rules once and reuse them |
| `__setattr__` | whole-object tracking | intercept all attribute assignments |

## property Is Usually Per Field

- `property` is commonly written for one specific field.

```python
class User:
    @property
    def age(self):
        return self._age

    @age.setter
    def age(self, value):
        if value < 0:
            raise ValueError("age cannot be negative")
        self._age = value
```

- This is direct and readable.
- But if many fields need the same validation, repeating properties can become noisy.

## Descriptor Is Reusable Field Logic

- A descriptor can represent a reusable field type.
- Example:

```python
class IntegerField:
    def __set_name__(self, owner, name):
        self.name = "_" + name

    def __get__(self, instance, owner):
        if instance is None:
            return self
        return getattr(instance, self.name)

    def __set__(self, instance, value):
        if not isinstance(value, int):
            raise TypeError("value must be int")
        setattr(instance, self.name, value)
```

- Use it in a model:

```python
class User:
    age = IntegerField()
    score = IntegerField()
```

- Now both `age` and `score` share the same validation rule.

## Why Frameworks Use Descriptors

- Descriptors are common in frameworks.
- They are useful for:
  - ORM fields
  - validation systems
  - lazy loading
  - dependency injection
  - automatic tracking
  - computed attributes
- Django-style and SQLAlchemy-style model fields are common examples of this pattern.

- In an ORM-like class:

```python
class User(Model):
    age = IntegerField()
```

- `age` looks like a normal variable.
- But internally:
  - `age.__get__()` may load data
  - `age.__set__()` may validate data
  - the framework may track changes

## Relationship Summary

- Descriptor is the protocol.
- `property` is one implementation of that protocol.
- So:
  - every `property` is a descriptor
  - not every descriptor is a `property`

## Which One Should I Use?

- Use `__setattr__` when:
  - I want to log every attribute assignment
  - I want object-wide validation
  - I want simple change tracking
- Use `property` when:
  - one specific attribute needs getter/setter logic
  - I want `obj.name` syntax backed by functions
  - I want simple validation for one field
- Use a descriptor when:
  - many fields need the same behavior
  - I am building a framework-like field system
  - I want reusable validation or tracking logic

## One-Line Memory Aids

- `__setattr__`:
  - intercept every assignment on the object.
- `property`:
  - make a method look like an attribute.
- descriptor:
  - define how attribute access works at the class level.

## References

- Related video from the original discussion:
  - https://www.youtube.com/watch?v=QbFjc9c0B08
- Python Descriptor HOWTO:
  - https://docs.python.org/3/howto/descriptor.html
- Python `property()` documentation:
  - https://docs.python.org/3/library/functions.html#property
- Python data model `object.__setattr__` documentation:
  - https://docs.python.org/3/reference/datamodel.html#object.__setattr__

## Final Summary

- To print every change, start with `__setattr__`.
- To control one attribute with getter/setter logic, use `property`.
- To reuse the same attribute behavior across many fields, use a descriptor.
- `property` is not separate from descriptors.
- `property` is a built-in descriptor.
- Descriptor is the deeper mechanism behind many Python attribute tricks.
