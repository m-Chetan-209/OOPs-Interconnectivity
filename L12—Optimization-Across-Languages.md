# Optimization Across C++, Java and Python

## 1. Why layout affects speed

The CPU is fast, but **reading from RAM is slow**. So the CPU fetches memory in **chunks of about 64 bytes** (a cache line) and keeps them in a small, fast cache. This is the same chunk idea behind alignment in Lesson 2.

Analogy: books across town.

- **Neighbors:** the books you need are on one shelf. One trip brings them all.
- **Scattered:** every book is in a different building, so you make a trip per book.

The trips are the slow part, so what decides speed is: **are the things I read next to each other in memory?**

### The experiment (C++, 5 million ints, adding them up)

```
Contiguous:   one array of 5 million ints, side by side
              [ 7 ][ 3 ][ 9 ][ 1 ] ...

Scattered:    5 million pointers, each to an int in its own heap block,
              visited in random order
              [ ptr ]──► 7   (somewhere)
              [ ptr ]──► 3   (somewhere else)
```

| Layout | Time |
|---|---|
| Contiguous array | about **5 ms** |
| Scattered via pointers | about **80 to 90 ms** |

Same data and same answer, about 16 to 18 times slower. I shuffled the order to force the worst case. Objects created one after another are often placed fairly close together, but you can't count on that once a program has run for a while.

### Is C++ simply the fastest?

No. C++ gives you **control**, and the control only helps if you use it:

| Version | Time to sum 5M numbers |
|---|---|
| C++, contiguous array | about 5 ms |
| C++, scattered via pointers | about 80 to 90 ms |
| Python, `sum(list)` | about 21 ms |
| Python, `for` loop on a list | about 80 ms |

Badly laid out C++ was as slow as a Python loop. Python is slower for a second reason that has nothing to do with cache: each step of a Python loop costs many operations (looking up names, checking types, creating objects). `sum(list)` is quicker because the loop runs inside Python's own C code.

---

## 2. Arrays of objects in each language

An array of 1,000 `Point`s, same count and same data:

```
C++       [x y][x y][x y][x y] ...        one contiguous block

Java      [ref][ref][ref] ...             1,000 references
            │    │    │
            ▼    ▼    ▼
          Point Point Point               1,000 separate objects on the heap

Python    [arrow][arrow][arrow] ...       1,000 arrows (8 bytes each)
             │      │      │
             ▼      ▼      ▼
           Point  Point  Point            1,000 separate objects
```

| | Layout | Total for 1,000 points |
|---|---|---|
| **C++** | one contiguous block | **8,000** bytes (checked with `sizeof`) |
| **Java** | references to scattered objects | about 28,000 (estimate) |
| **Python with `__slots__`** | arrows to scattered objects | about 57,000 |
| **Python normal** | arrows to scattered objects | about 92,000 |

C++ wins twice: **less memory**, and **everything side by side**. The Python list is about 11.5× bigger because each point is about 10× bigger (80 vs 8 bytes), and the list also needs an 8-byte arrow per point.

---

## 3. What a virtual call costs

From Lesson 4, a virtual call reads the object's vptr, then the vtable, then jumps. I ran 5 million calls in C++, with identical layout so only the call type changed:

| Version | Time |
|---|---|
| Non-virtual call | about **12 ms** |
| Virtual call, all one type | about **18 ms** |
| Virtual call, two types randomly mixed | about **40 ms** |

Why it's slower:

1. **Extra reads** of the vptr and vtable.
2. **No inlining.** The compiler can't paste the function body into the loop when it doesn't know which function runs.
3. **Jump prediction fails.** The CPU guesses where a jump goes. That's easy when the type never changes and often wrong when types alternate, which is the jump from about 18 to about 40 ms.

Python shows the same pattern, on 5 million iterations:

| Version | Time |
|---|---|
| A loop with no method call | about 123 ms |
| Method call, objects of **one class** | about 171 ms |
| Method call, objects of **two classes** mixed | about 262 ms |

Python's interpreter speeds up repeated lookups when the type stays the same, and loses that speedup when types alternate.

**When it matters:** a tiny function called millions of times in a hot loop. In ordinary code a few nanoseconds don't matter, and the clarity of polymorphism is worth more. Measure first.

---

## 4. Field order and object size

### C++: order matters

From Lesson 2, padding depends on the order:

```cpp
struct Bad  { char a; double b; char c; };    // 24 bytes
struct Good { double b; char a; char c; };    // 16 bytes
```

For 1 million objects that is **24 MB vs 16 MB**. Another pair: `{char, int, char}` is 12 bytes and `{int, char, char}` is 8.

```cpp
struct S { char a; int b; char c; double d; };   // 24 bytes
```

| Offset | Field |
|---|---|
| 0 | `a` |
| 1-3 | padding (the `int` needs a multiple of 4) |
| 4-7 | `b` |
| 8 | `c` |
| 9-15 | padding (the `double` needs a multiple of 8) |
| 16-23 | `d` |

Reordered as `{ double d; int b; char a; char c; }`, it is **16**. Smaller objects also mean more of them fit in one 64-byte chunk, so scanning them is faster too.

### Java: the JVM reorders for you

You don't control the order. You control the **number of fields** and their **types**.

### Python: order doesn't matter, the count does

Each field is an 8-byte arrow, so there is no padding to tune. Bytes per object, with small shared numbers:

| Fields | Normal class | With `__slots__` |
|---|---|---|
| 1 | 73 | 41 |
| 2 | 81 | 49 |
| 4 | 97 | 65 |
| 8 | 137 | 97 |

Each extra field costs about **8 bytes** with `__slots__` (8 to 10 for a normal class). The fixed starting cost is about 33 bytes for a slots object and about 65 for a normal one.

---

## 5. What Python objects cost, and what `getsizeof` counts

Every Python object has a **16-byte header** (an 8-byte reference count and an 8-byte type pointer, Lesson 7), plus whatever data it holds. Measured with `sys.getsizeof`:

| Object | Bytes | Why |
|---|---|---|
| `None` | 16 | just the header |
| `object()` | 16 | just the header |
| `3.14` (float) | 24 | header + 8 for the number |
| `5` (int) | 28 | header + a little for the digits |
| `2**30` | 32 | bigger ints need more room |
| `2**60` | 36 | |
| `2**100` | 40 | |
| `'a'` (str) | 42 | each extra character adds 1 |
| `'hello'` (str) | 46 | |
| `()` (empty tuple) | 40 | |
| `[]` (empty list) | 56 | |
| `{}` (empty dict) | 64 | |

A C++ `double` is 8 bytes with no header, and a Python float is 24. Python ints have no upper limit, so they grow with the value, unlike a C++ `int`. Python also rounds allocations up, so an int takes about 32 bytes of real memory while `getsizeof(5)` says 28.

### What `getsizeof` counts

It counts **the object's own block**. The arrows stored inside a container are counted, but **the objects at the other end of the arrows are not**.

```
list (this block is what getsizeof counts)          Elsewhere (NOT counted)
┌───────────────────────────┐
│ header                    │
│ arrow ────────────────────┼──────────────►  [ item 1 ]
│ arrow ────────────────────┼──────────────►  [ item 2 ]
│ arrow ────────────────────┼──────────────►  [ item 3 ]
└───────────────────────────┘
```

| Object | `getsizeof` | What adds up to it |
|---|---|---|
| list of 1,000 numbers | 8,056 | 56 (header) + 1,000 arrows × 8 |
| list holding that list | 64 | 56 + 1 arrow × 8 |
| list of 2 long strings | 72 | 56 + 2 arrows × 8 (the strings alone are 1,582) |
| a list of 3 long strings | 80 | 56 + 3 × 8, **without** the strings |

| Container | Counted | Left out |
|---|---|---|
| list, tuple, set | its own array of arrows | the items |
| dict | the table | keys and values |
| your own object | its own block | the values its fields point to (and, for a normal object, part of the field storage) |

To add up everything, follow the arrows:

```python
import sys

def deep_size(o, seen=None):
    seen = seen if seen is not None else set()
    if id(o) in seen:
        return 0                          # don't count the same object twice
    seen.add(id(o))
    size = sys.getsizeof(o)
    if isinstance(o, dict):
        size += sum(deep_size(k, seen) + deep_size(v, seen) for k, v in o.items())
    elif isinstance(o, (list, tuple, set)):
        size += sum(deep_size(i, seen) for i in o)
    return size
```

On my test, `deep_size` of a list holding a 1,000-number list was **36,120** bytes, against `getsizeof` at 64. It also counts shared small numbers that exist anyway. For the measurements in these notes, I used `tracemalloc`, which reports memory actually allocated.

---

## 6. Choosing the right structure

**For lots of small items, store the values themselves, side by side, not arrows to separate objects.**

### Python: a million numbers

| Structure | Memory | Time to sum |
|---|---|---|
| `list` of ints | **40.0 MB** | 19 ms (`sum`), 95 ms (`for` loop), on 5M |
| `array.array('i')` | **4.1 MB** | not timed |
| NumPy `int32` | **4.0 MB** | **1.7 ms** on 5M |
| `list` of floats | **32.4 MB** | |
| NumPy `float64` | **8.0 MB** | |

A list needs an 8-byte arrow plus a separate object per number (about 32 bytes for an int and 24 for a float). A NumPy array stores each number directly, side by side.

**Use NumPy's own functions, not Python's built-in `sum()`.** On 1,000,000 floats:

| Way | Time |
|---|---|
| `arr.sum()` | **0.33 ms** |
| `sum(list)` | 8.6 ms |
| `sum(arr)` (built-in on a NumPy array) | **53 ms** |

The built-in `sum()` walks the array one element at a time and creates a Python object for each, so it is slower than summing a plain list.

### Python: a record with two fields

100,000 records, small shared numbers, bytes per record:

| Representation | Bytes |
|---|---|
| `__slots__` class | **48** |
| tuple `(x, y)` | 56 |
| namedtuple | 64 |
| normal class | 80 |
| dict `{"x": .., "y": ..}` | **184** |

A dict is the heaviest, because every record carries its own table of names.

### All three languages

| Language | Do this | Not this |
|---|---|---|
| **C++** | `vector<Point>` | `vector<Point*>` |
| **Java** | `int[]` (4 bytes per number) | `Integer[]` (about 20 bytes per number, scattered) |
| **Python** | NumPy or `array` for numbers, `__slots__` for many small objects | a list of ints, a list of dicts |

---

## Summary

1. The CPU reads memory in ~64-byte chunks, so items **side by side** are much faster than scattered ones (5 ms vs 80 to 90 ms in C++).
2. C++ arrays of structs are contiguous. Java object arrays and Python lists hold **arrows to scattered objects**, which are bigger and slower.
3. Badly laid out C++ can be as slow as a Python loop. Layout and overhead decide speed, not the language name.
4. A virtual call costs a few nanoseconds more, and mixed types make it worse. It only matters in hot loops, so measure first.
5. In C++ field **order** changes size (24 vs 16 bytes). In Java the JVM reorders. In Python the **number** of fields matters (about 8 bytes each).
6. A Python object costs **16 bytes of header** plus its data. `getsizeof` counts a container's arrows but **not what they lead to**.
7. For bulk data, store values directly: `vector<Point>` in C++, `int[]` in Java, NumPy or `array` in Python, and call NumPy's own `.sum()`.
