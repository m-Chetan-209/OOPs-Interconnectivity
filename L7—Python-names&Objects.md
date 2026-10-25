# Names and objects

**EVERYTHING IN PYTHON IS AN OBJECT.**

Compare the three languages on one line:

```
int a = 5;          // C++: a IS a 4-byte box holding 5
int a = 5;          // Java: same, a primitive holds the value
Point p = new Point(1, 2);   // Java: p is a reference to a heap object
a = 5               # Python: a is a name pointing to an object
```

Python has no primitives. The 5 is an object on the heap, and a is a name (a reference, like Java's p) that points to it.

No declared types. You never write `int a`. The name `a` has no type. The object it points to has one.

A name is not a box that holds a value. It's a pointer to an object, so `a = 5` means "make `a` point to the object 5".

```python
a = 5
b = a
```

`b = a` doesn't copy the 5. It makes `b` point to the same object, exactly like `Point t = s;` in Java. In Python, `id(a) == id(b)` is True, because `id()` shows which object a name points to.

---

## Why call 5 an object?

An "object" is just a block of memory with a header and some data in it. The Point you built in C++ is that, and so is 5 in Python:

Python does the work for you. When Python reads the 5 in your code, it creates this block for you, with no constructor call and no `new`.

```
int object for 5
┌─────────────────────────┐
│ reference count         │  ← header (bookkeeping)
│ type pointer → "int"    │  ← header (what kind of thing I am)
│ value: 5                │  ← data
└─────────────────────────┘   28 bytes in total
```

It also behaves like an object, with a type and methods:

```python
type(5)           # <class 'int'>   → 5 belongs to the class int
(5).bit_length()  # 3               → a method called on the object 5
(5).__add__(3)    # 8               → even + is a method call underneath
```

In Java, the number 5 is a plain value, and Integer is the object version, so you have to ask for the object.
In Python, the object version is the only version.

---

## Reassigning a name

What does `b = 10` do?

```python
a = 5
b = a        # both names point to the 5 object
b = 10       # b now points to a NEW object, 10
```

`a` is still 5.

## Why Python has no "t.x = 99" for numbers

You can't change the 5 object itself, because Python's int objects are immutable: once made, their value can never change. `b = 10` therefore can only re-point.

### Check

```python
# Case 1
a = [1, 2]
b = a
b.append(3)

# Case 2
c = [1, 2]
d = c
d = [9]
```

1. What are `a` and `b` after Case 1?
2. What are `c` and `d` after Case 2?

---

|           | Can the object itself be changed? | Examples                                          |
|-----------|-----------------------------------|---------------------------------------------------|
| Immutable | No, never                         | `int`, `float`, `str`, `bool`, `tuple`            |
| Mutable   | Yes, in place                     | `list`, `dict`, `set`, and your own class objects |

## The trap: `+=` looks like it changes the object

```python
x = 5
x += 1
```

It looks like the 5 became 6. What really happens is `x = x + 1`:

1. Python computes 5 + 1 and creates a new int object, 6.
2. It re-points `x` to it.

```
Before:  x ──► [ 5 ]
After:   x ──► [ 6 ]       the 5 object is untouched
```

Strings do the same:

```python
s = "hi"
s += "!"
```

A new string "hi!" is built, and `s` is re-pointed to it. The old "hi" is not modified. If no name points to it any more, its memory is freed.

---

## Who frees objects in Python?

In C++ you wrote `delete`. In Java, the garbage collector freed unreachable objects at an unknown time. Python usually does it immediately, using a number stored in the header of every object: the reference count.

The rules:

- A new name pointing at the object: count goes up by 1.
- A name re-pointed away, or deleted: count goes down by 1.
- When the count reaches 0, nobody can reach the object, so Python frees it right away.

```python
a = [1, 2]       # list object created, count = 1
b = a            # count = 2
b = None         # b re-pointed away, count = 1
a = None         # count = 0 → freed immediately
```

`None` is Python's null. It's a real object that means "nothing here."

## The `del` statement

`del x` removes the name `x`. It does not directly destroy the object:

```python
x = [1, 2]       # count = 1
y = x            # count = 2
del x            # count = 1 → the list is still alive, y reaches it
del y            # count = 0 → freed now
```

|            | C++           | Java              | Python                    |
|------------|---------------|-------------------|---------------------------|
| Who frees  | you (`delete`) | garbage collector | reference count (usually) |
| When       | when you say  | some unknown time | the moment the count hits 0 |

---

## `==` vs `is`

Python has two ways to compare, and they ask different questions:

- `==`: do the two objects have the same contents?
- `is`: are they the very same object? (the same address, as `id()` shows)

- `a == b` is True: same contents.
- `a is b` is False: two different objects.
- `a is c` is True: one object, two names.

### Check

```python
a = [1, 2]
b = [1, 2]
c = a
```

1. What are `a == b`, `a is b`, and `a is c`?
2. If you run `a.append(3)`, does `c` change? Does `b`?

---

## Passing objects to functions

```python
def add(lst):
    lst.append(9)       # follows the pointer, changes the object

def rebind(lst):
    lst = [0]           # re-points the local name only

a = [1]
add(a)                  # a is now [1, 9]
rebind(a)               # a is still [1, 9]
```

Re-pointing means erasing the arrow and drawing a new one to a different object.

### Check

```python
def add(lst):
    lst.append(9)

def rebind(lst):
    lst = [0]

def inc(n):
    n += 1

a = [1]
x = 5
add(a)
rebind(a)
inc(x)
```

1. What are `a` and `x` at the end?
2. Which of the three functions changed an object, and which only re-pointed a name?

### Check

```python
def f(lst):
    lst = lst + [5]

def g(lst):
    lst += [5]

a = [1]
b = [1]
f(a)
g(b)
```

1. After both calls, what are `a` and `b`?
2. Which function changed the object, and which re-pointed?

## The trap

`+=` behaves differently depending on the type of the object:

| Type                              | `x += something`                  | Why                                 |
|-----------------------------------|-----------------------------------|-------------------------------------|
| `int`, `str`, `tuple` (immutable) | re-points `x` to a new object     | the old object can't be changed     |
| `list` (mutable)                  | edits the object in place         | the list can be changed             |

---

## Copying

### The catch: a list holds references

A list doesn't contain its items. It contains references to its items, just like Java's `Point[]`. So what does a copy copy?

```python
a = [[1, 2], [3, 4]]      # a list of two lists
b = a.copy()
```

`.copy()` makes a new outer list, but copies only the references inside it.

This is a shallow copy: one level deep.

What we want is a new list unaffected by changing the other one.

### The fix: deep copy

```python
import copy
b = copy.deepcopy(a)
```

`deepcopy` copies every level, so the inner lists become new objects too. After `a[0].append(99)`, `b` stays `[[1, 2], [3, 4]]`.

| Way                         | New outer list?   | New inner lists?      |
|-----------------------------|-------------------|-----------------------|
| `b = a`                     | no (same object)  | no                    |
| `b = a.copy()` (shallow)    | yes               | no, shared            |
| `copy.deepcopy(a)` (deep)   | yes               | yes                   |

### Check

```python
a = [[1, 2], [3, 4]]
b = a.copy()
b.append([5, 6])
b[1].append(0)
```

1. What are `a` and `b` afterwards?
2. Which of the two changes (`b.append([5, 6])` or `b[1].append(0)`) affected `a`, and why?

---

## Reference cycles, and the backup garbage collector

Reference counting has one blind spot. Look at this:

```python
a = [1]
b = [2]
a.append(b)      # list A holds a reference to list B
b.append(a)      # list B holds a reference back to list A
del a
del b
```

```
list A ──► list B
  ▲          │
  └──────────┘
```

After the two `del` lines, no name can reach either list. But each list is still pointed to by the other, so both counts are 1, not 0. Reference counting alone would never free them: a leak.

This is where Python's cyclic garbage collector comes in. It runs now and then, looks for groups of objects that nothing from outside can reach, and frees the whole group. I tested it on real Python: after the two `del` lines the objects were still alive. After forcing a collection, they were gone.

| Case                | Freed by                                                |
|---------------------|---------------------------------------------------------|
| Normal objects      | reference count reaching 0, immediately                 |
| Objects in a cycle  | the cyclic garbage collector, later, at an unknown time |

### Check

```python
a = [1]
a.append(a)      # the list holds a reference to itself
del a
```

1. After `del a`, what is the list's reference count: 0, or 1?
2. So who frees it: reference counting, or the cyclic garbage collector?
