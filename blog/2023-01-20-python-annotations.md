---
title: "Examining the Problem of Forward References in Python Type Annotations"
description: >-
  A few interesting observations on Python type annotations, forward
  reference handling, and future imports.
tags:
  - python
---

## Type Hints

Python 3 introduced a new syntax for adding type hints to variables and functions:

```python
class Container:
    def __init__(self, name: str):
        """Initializes an empty container with the specified name"""
        self.name: str = name
        self.items: list[str] = []
```

Python is not a strongly typed language, so type hints are not checked at runtime
and mismatches do not cause an error. Rather, the annotations are intended to be processed
by third-party static analyzers like [Mypy][mypy], to detect bugs as part of the CI process.
Type hints also serve as an indicator of a library developer's intent and can be used to generate
richer API documentation.

## Forward References

One limitation of the annotation syntax is the ability to handle forward references.

```python
class Container:
    def transfer(self, other: 'Container') -> None:
        """Moves the contents of this container to the other container"""
        other.items.extend(self.items)
        self.items = []
```

<!-- truncate -->

In the above method, we cannot reference `Container` in a type annotation before the class is fully defined.
To avoid a name error, the annotation must be specified as a string.

Many developers find this syntax awkward, so an alternative was proposed in [PEP 563][pep563].
In this standard, the interpreter no longer evaluates the annotations at definition time.
Instead, annotations are stored behind the scenes in an `__annotations__` dictionary,
for postponed evaluation.

As a result, this code becomes legal:

```python
from __future__ import annotations

class Container:
    def transfer(self, other: Container) -> None:
        """Moves the contents of this container to the other container"""
        other.items.extend(self.items)
        self.items = []
```

Unless you are developing your own type checker, the evaluation of type hints should be
a low-level implementation detail. All we need to worry about is including the `__future__` import
so we can use the more convenient type hint syntax. What could go wrong?

## The Breaking Change

As it turns out, there are documented interfaces in the standard library that allow you
to inspect type hints at runtime, as a rudimentary form of reflection.
Consider this example, which looks at the [Field][field] objects on a dataclass:

```python
from dataclasses import dataclass, fields

@dataclass
class Item:
    name: str
    owner: str
    quantity: int

def _main():
    for f in fields(Item):
        print(f'{f.name} {f.type}')

if __name__ == '__main__':
    _main()
```

```
name <class 'str'>
owner <class 'str'>
quantity <class 'int'>
```

Putting a `from __future__ import annotations` at the top of the script changes the output to:

```
name str
owner str
quantity int
```

In other words, instead of returning a type object, `f.type` now returns an _annotation string for postponed evaluation_.
So to retain the original behavior of our code, we need to `eval()` that string:

```python
from __future__ import annotations
from dataclasses import dataclass, fields

@dataclass
class Item:
    name: str
    owner: str
    quantity: int

def _main():
    for f in fields(Item):
        print(f'{f.name} {eval(f.type)}')

if __name__ == '__main__':
    _main()
```

Although this usage of type objects is a bit uncommon, changing the behavior could break some third-party libraries,
both in terms of code and performance. (Calling `eval()` does incur some overhead.)

## The Future

PEP 563 was originally scheduled for full adoption in Python 3.10. However, it has been [postponed] several times.
A competing proposal [PEP 649][pep649] is also up for discussion, with its own set of benefits and drawbacks.
After years of debate, both proposals have been deemed insufficient by the Python Steering Council,
so it is unclear what will happen next.

For now, if you choose to use `from __future__ import annotations` to work around the forward reference problem,
just beware of the risks. `__future__` behaviors may seem straightforward, but there can be weird surprises lurking
beneath the surface.


[field]: https://docs.python.org/3.11/library/dataclasses.html#dataclasses.Field
[mypy]: https://github.com/python/mypy
[pep563]: https://peps.python.org/pep-0563/
[pep649]: https://peps.python.org/pep-0649/
[postponed]: https://mail.python.org/archives/list/python-dev@python.org/message/VIZEBX5EYMSYIJNDBF6DMUMZOCWHARSO/
