# Python Objects in Memory

All numbers were measured on CPython 3.12 (64-bit). Sizes in Java are estimates from the 12-byte header layout (Lesson 6), because Java was not run.

---

## Where an object keeps its fields: `__dict__`

In C++, `Point` with two `int`s is 8 bytes, with `x` at offset 0 and `y` at offset 4, fixed before the program runs. Python has no fixed offsets. Each object carries a **dictionary** that matches field **names** to values (arrows). Python calls it `__dict__`:

```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

p = Point(3, 4)
print(p.__dict__)       # {'x': 3, 'y': 4}
```

```
p (the object)
┌──────────────────────────────┐
│ __dict__ ──► { "x" ──► 3,    │
│               "y" ──► 4 }    │
└──────────────────────────────┘
```

"Look in the object first" (Lesson 8) means "look in the object's `__dict__` first".

### Fields can be added to one object

```python
a = Point(1, 2)
b = Point(5, 6)
a.color = "red"
a.x = 10

print(a.__dict__)   # {'x': 10, 'y': 2, 'color': 'red'}
print(b.__dict__)   # {'x': 5, 'y': 6}
b.color             # ✗ AttributeError: 'Point' object has no attribute 'color'
```

- `a.color = "red"` **added** an entry, to `a` only.
- `a.x = 10` did not add a second `x`. The entry existed, so it was **re-pointed**.

**Rule:** `obj.name = value` adds `name` to that object's dictionary if it isn't there, and re-points it if it is.

### Why this works

A Python class **doesn't declare fields**. Nothing in `class Point` says "a Point has `x` and `y`". They exist only because `__init__` ran `self.x = x` and `self.y = y`. Each object starts empty, and assignment fills its table:

```
Point(1, 2) starts as:     { }
self.x = x      →          { "x": 1 }
self.y = y      →          { "x": 1, "y": 2 }
a.color = "red" →          { "x": 1, "y": 2, "color": "red" }   (a only)
```

| | C++ / Java | Python |
|---|---|---|
| Fields decided | by the class, before the program runs | by the code that runs on each object |
| Add a field to one object later | not possible | possible |

The price of this flexibility is memory, since a dictionary stores the names as well as the values.

---

## What one Point costs

I created 100,000 points and measured the memory per point:

| Point | Bytes per point |
|---|---|
| `Point(1, 2)`, small numbers | **80** |
| `Point(1000, 2000000)`, big numbers | **144** |

### Shared number objects

Python creates the integers from **-5 to 256** once, at startup, and reuses them. Writing `1` or `44` doesn't build a new object. It hands out the existing one:

```python
p = Point(1, 2)
q = Point(1, 2)
p.x is q.x          # True: both fields point to the very same object, 1
```

```
p.x ──┐
      ├──► [ int 1 ]     (one object, made at startup, used by everyone)
q.x ──┘
```

So `a = Point(44, 54)` creates **one new object**, the `Point`. The numbers 44 and 54 already existed. Two `Point` calls always make two separate Points, but the small numbers inside them are shared.

Numbers outside -5 to 256 are built **fresh each time**, about 32 bytes each:

```python
a = Point(int("1000"), int("2000"))
b = Point(int("1000"), int("2000"))
a.x is b.x          # False: two separate int objects holding 1000
```

(Don't rely on `is` with numbers. Python may merge repeated constants written in the code.)

### Reading the cost

```
p (the Point object, 80 bytes)           separate number objects
┌───────────────────────┐
│ header                │
│ x: an arrow ──────────┼──────────────►  [ int 1000 ]      (its own ~32 bytes)
│ y: an arrow ──────────┼──────────────►  [ int 2000000 ]   (its own ~32 bytes)
└───────────────────────┘
```

| Point | The Point itself | New number objects | Total |
|---|---|---|---|
| `Point(10, 20)` | 80 | none | **80** |
| `Point(10, 3000)` | 80 | one (the 3000), 32 | **112** |
| `Point(1000, 2000000)` | 80 | two, 32 each | **144** |

`sys.getsizeof(p)` reports only 48 here. It leaves out the separate block that holds the fields, so it undercounts. The 80 comes from measuring the memory actually allocated.

### The same Point in four forms

| | Bytes |
|---|---|
| C++ | 8 |
| Java (estimate) | 24 |
| Python with `__slots__` | 48 |
| Python normal | 80 |

---

## `__slots__`: removing the per-object table

If you know the fields in advance, you can tell Python to reserve room for just those, and no table:

```python
class Point:
    __slots__ = ("x", "y")          # this class has exactly these two fields

    def __init__(self, x, y):
        self.x = x
        self.y = y
```

| Class | Bytes per `Point` |
|---|---|
| normal | **80** |
| with `__slots__` | **48** |

That saves 32 bytes per object, so 32 MB for a million points. Older Python versions saved more.

### The price: no new fields

```python
p = Point(1, 2)
p.z = 5                  # ✗ AttributeError: 'Point' object has no attribute 'z'
hasattr(p, "__dict__")   # False
```

### The rules

- **`__slots__` goes in the class body**, at the same level as the `def` lines. Where in the body doesn't matter (it still worked when placed after `__init__`), but the top is the convention.
- **It does nothing inside `__init__`.** `self.__slots__ = (...)` there is just an ordinary field, and the object keeps its table. Python needs the fields before any object is built, and `__init__` runs too late.
- **Every field you assign with `self.` must be listed.** Anything missing fails at once:

```python
class Point:
    __slots__ = ("x", "y")
    def __init__(self, x, y):
        self.x = x
        self.y = y
        self.z = 0       # ✗ AttributeError: z is not in __slots__
```

The fix is to add `"z"` to the list.

- **Parameters don't matter.** `__slots__` lists fields, not parameters. A constructor with three parameters and two slots works fine, because parameters vanish when `__init__` ends.

### Inheritance

```python
class A:
    __slots__ = ("x",)

class B(A):              # no __slots__ of its own
    pass

class C(A):
    __slots__ = ("y",)
```

| Class | Has the per-object table again? |
|---|---|
| `B` | **yes**, and it accepts new fields |
| `C` | **no** |

A child that doesn't write its own `__slots__` gets the table back, which undoes the saving. To keep it, every class in the chain needs `__slots__`.

### When to use it

| Situation | Use it? |
|---|---|
| Millions of small objects (points, records, nodes) | **yes** |
| A few objects | no, the saving is tiny |
| You need to add fields on the fly | no, it forbids that |
| A library needs `__dict__` | no |

---

## Summary

1. Python objects keep their fields in a **per-object table** (`__dict__`), so fields can be added to one object at any time.
2. `obj.name = value` adds the name or re-points it.
3. A `Point(1, 2)` costs about **80 bytes** here, against 8 in C++.
4. Numbers from -5 to 256 are **shared**, so they add nothing. Bigger ones add about 32 bytes each.
5. `__slots__` removes the table, saving **32 bytes per object**, but forbids new fields.
6. List **every** field in `__slots__`, put it **in the class body**, and write it in **every class** of an inheritance chain.
