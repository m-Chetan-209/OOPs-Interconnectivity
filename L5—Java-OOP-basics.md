# Java: References and the Heap

## Running the examples

Java statements must live inside a method. Every snippet below that shows loose statements is meant to sit **inside `main`**, with the `Point` class available:

```java
class Point {
    int x;
    int y;
    Point(int a, int b) { x = a; y = b; }
}

public class Demo {
    public static void main(String[] args) {
        // ← the snippets marked "inside main" go here
    }
}
```

`public static void main(String[] args)` is just Java's required wrapper. Think of it as the same `main()` from C++, and ignore the extra words for now.

---

## A Java variable is a pointer you can't see

The C++ line you already know:

```cpp
Point *p = new Point(3, 4);   // p on the stack, the object on the heap
```

The Java version:

```java
class Point {
    int x;
    int y;
    Point(int a, int b) { x = a; y = b; }
}

// inside main
Point p = new Point(3, 4);
```

The memory picture is identical:

```
Stack              Heap
┌──────────┐      ┌─────────┐
│ p = 5000 │ ───► │ x = 3   │
└──────────┘      │ y = 4   │
 (a reference)    └─────────┘
```

Three differences in how you write it:

- **No `*` in the declaration.** In Java, a variable of a class type is *always* this kind of pointer, so there's nothing to mark. It's called a **reference**.
- **No `->`.** You always write `p.x`, and Java follows the reference for you.
- **No `delete`.** You never free anything yourself. A garbage collector does it (see below).

Also, Java has no `Point p(3, 4);` form. **Every object is created with `new`, so every object lives on the heap.** There are no stack objects like in C++.

In C++ there were two ways to make a `Point`:

```cpp
Point p(3, 4);                // way 1: the object sits directly in the variable (stack)
Point *q = new Point(3, 4);   // way 2: the object is on the heap, q holds its address
```

**Java has only way 2.**

```java
// inside main
Point q = new Point(3, 4);
```

1. `new Point(3, 4)` creates the object on the **heap** and gives back its address.
2. `q` is a variable on the **stack**, and it stores that address.

> **Reference = a pointer that Java hides from you.**

---

## Reference vs object: counting

A declaration alone makes a reference. Only `new` makes an object.

```java
// inside main
Student s;                // line 1: reference s created, no object
s = new Student();        // line 2: object #1 created, s points to it
Student t = new Student();// line 3: reference t created AND object #2 created
```

```
After line 1:               After line 3:
Stack        Heap           Stack          Heap
┌────────┐                  ┌──────────┐   ┌────────────┐
│ s = ?  │  (nothing)       │ s = 5000 │──►│ Student #1 │
└────────┘                  ├──────────┤   ├────────────┤
                            │ t = 7000 │──►│ Student #2 │
                            └──────────┘   └────────────┘
```

**Count the `new`s to count the objects. Count the declared names to count the references.**

Every `new` makes one object. There are two `new`s, so there are two objects.

---

## Assigning one reference to another

```java
// inside main
Student s = new Student();
Student t = s;           // no new → no new object
```

Only the **address** is copied into `t`. `s` and `t` are two separate boxes holding the same address, so they are two names for the same object.

```
Stack            Heap
┌──────────┐    ┌─────────────┐
│ s = 5000 │───►│ Student     │
├──────────┤ ┌─►│ (one object)│
│ t = 5000 │─┘  └─────────────┘
└──────────┘
```

> The **boxes** `s` and `t` are at different locations. What they **hold** is the same address.

---

## Changing the object through one reference

If two references point to the same object, changing it through one is visible through the other.

```java
// inside main
Point s = new Point(3, 4);
Point t = s;       // same object, no copy
t.x = 99;
```

`t.x = 99;` means: follow the address in `t`, then set `x` to 99. That is the object `s` also points to. So `s.x` is 99 as well.

```
Stack            Heap
┌──────────┐    ┌──────────┐
│ s = 5000 │───►│ x = 99   │
├──────────┤ ┌─►│ y = 4    │
│ t = 5000 │─┘  └──────────┘
└──────────┘
```

Compare it with the C++ pointer version you already know:

```cpp
Point *s = new Point(3, 4);
Point *t = s;      // copies the address only
t->x = 99;         // s->x is also 99
```

---

## How to tell a value from an address, with no `*`?

**The declared type decides what a variable holds.**

| Declared type | The variable holds | Example |
|---|---|---|
| **Primitive**: `int`, `double`, `char`, `boolean`, `long`... | the **value itself** | `int a = 5;` → `a` holds 5 |
| **Class**: `Point`, `Student`, `String`... | an **address** (a reference) | `Point s = ...;` → `s` holds 5000 |

| C++ | Java | Meaning |
|---|---|---|
| `p` | `s` | the pointer (box holding the address) |
| `p->x` or `(*p).x` | `s.x` | follow the address, then pick the field |
| `p = q;` | `s = t;` | copy the address, so both point to one object |
| `*p` (the whole object) | *(no equivalent)* | Java never gives you the whole object as a value |

Two things work differently from C++:

- There's no `*s`. The dot `.` always follows the reference for you.
- You can't print or do math on the address (no `p + 1`). Java keeps the address hidden, but it's there in the memory picture.

### Why it matters: copying

```java
// inside main
int a = 5;
int b = a;       // copies the VALUE: b gets its own 5
b = 10;          // a is still 5

Point s = new Point(1, 2);
Point t = s;     // copies the ADDRESS: both point to one object
t.x = 10;        // s.x is now 10 too
```

Both look like `b = a` and `t = s`, but the type decides whether a value or an address is copied.

---

## Who frees the heap object? The garbage collector

In C++, forgetting `delete` leaked the heap object. Java has no `delete`. A built-in helper called the **garbage collector (GC)** frees objects for you.

An object is **reachable** if at least one reference still points to it. When **no reference** points to it, nobody can ever use it again, so the GC is allowed to free it. The GC runs on its own schedule, so you don't know exactly when.

```java
// inside main
Point s = new Point(1, 2);   // object #1 created, s points to it
s = new Point(3, 4);         // object #2 created, s now points to it
```

```
Stack              Heap
┌──────────┐      ┌─────────┐
│ s = 7000 │ ───► │ (3, 4)  │  ← reachable
└──────────┘      └─────────┘
                  ┌─────────┐
                  │ (1, 2)  │  ← nothing points here: GC can free it
                  └─────────┘
```

### Check

```java
// inside main
Point s = new Point(1, 2);
Point t = new Point(3, 4);
s = t;
```

1. How many objects were created in total?
2. After line 3, how many of them are reachable?

**Answers:** 1. 2 objects. 2. 1 reachable (the `(3, 4)` object, which both `s` and `t` now point to). The `(1, 2)` object has no reference left.

---

## `null`

A reference can hold **no address at all**. Java calls that value `null`.

```java
// inside main
Point s = null;     // s holds "nothing"
```

```
Stack              Heap
┌──────────┐
│ s = null │       (no object)
└──────────┘
```

In C++ terms, this is `Point *s = nullptr;`.

If you try to follow a `null` reference, Java stops the program with a **NullPointerException**:

```java
// inside main
Point s = null;
s.x = 5;            // ✗ crash: there is no object to follow
```

C++ would give undefined behavior here (a crash or silent garbage). Java gives a clear, predictable error instead.

`null` has a second use. Setting a reference to `null` can make an object unreachable, which lets the garbage collector free it:

```java
// inside main
Point s = new Point(1, 2);   // object exists, s points to it
s = null;                    // nothing points to the object now → GC may free it
```

### Check

```java
// inside main
Point s = new Point(1, 2);
Point t = s;
s = null;
```

1. Is the object still reachable after line 3?
2. Does `t.x = 9;` work or crash?
3. What happens with `s.x = 9;`?

**Answers:** 1. Yes, `t` still points to it. 2. Works, the object becomes `{9, 2}`. 3. Crash (`NullPointerException`), because `s` is `null`.

---

## Passing an object to a method

Java always copies **whatever the variable holds**. For a class type, that's the address. So the method gets a copy of the address, which leads to the same object.

```java
public class Demo {
    static void move(Point p, int dx) {
        p.x += dx;
    }

    public static void main(String[] args) {
        Point s = new Point(3, 4);
        move(s, 5);
        System.out.println(s.x);    // 8
    }
}
```

```
main                       move
┌──────────┐              ┌──────────┐
│ s = 5000 │──┐           │ p = 5000 │──┐
└──────────┘  │           └──────────┘  │
              └──► ┌────────────┐ ◄─────┘
                   │ x = 3 → 8  │
                   │ y = 4      │
                   └────────────┘
```

- `s` and `p` are two separate boxes holding the same address.
- `p.x += dx;` follows that address and changes the one object, so `s.x` becomes 8.

### For a primitive

```java
static void change(int n) { n = 100; }

// inside main
int a = 5;
change(a);        // a is still 5
```

`n` gets a copy of the **value** 5, so changing `n` can't touch `a`.

### Changing the object vs changing the box

- **Changing the object through `p`** (`p.x = ...`) is visible to the caller, because both boxes lead to the same object.
- **Changing the box `p` itself** (`p = ...`) is **not** visible to the caller, because `p` is only a copy of the box.

### Check

```java
static void move(Point p, int dx) { p.x += dx; }
static void reset(Point p) { p = new Point(0, 0); }
static void zero(Point p) { p.x = 0; p.y = 0; }

// inside main
Point s = new Point(3, 4);
reset(s);
zero(s);
```

1. After `reset(s);`, what is `s`?
2. After `zero(s);`, what is `s`, and why is it different from `reset`?

**Answers:**

1. `s` still points to the **original** `{3, 4}` object. `reset` only overwrote its own copy of the box (`p`), and the new `(0, 0)` object was lost when the method ended.
2. `s` holds the **same address as before**, but the object's contents are now `{0, 0}`. `zero` changed the object through the reference, and no new object was created.

> **Address in the box** changes only when you assign to the variable itself (`s = ...`). **Contents of the object** change when you assign through a field (`s.x = ...`).

---

## Arrays of objects

In C++, an array of `Point` holds the objects themselves, side by side:

```cpp
Point arr[3];     // 3 Points in a row, 24 bytes, all in one block
```

In Java, a class-type array holds **references**, not objects:

```java
// inside main
Point[] arr = new Point[3];
```

This creates **one** thing: an array of 3 boxes, each holding `null`. **No `Point` objects exist yet.**

```
Stack            Heap
┌──────────┐    ┌──────────────┐
│ arr=5000 │───►│ null         │
└──────────┘    │ null         │
                │ null         │
                └──────────────┘
```

To fill it, create each object:

```java
// inside main
arr[0] = new Point(1, 2);
arr[1] = new Point(3, 4);
```

```
arr (5000)          Separate heap objects
┌──────────────┐
│ ref ─────────┼───► { 1, 2 }
│ ref ─────────┼───► { 3, 4 }
│ null         │
└──────────────┘
```

So the objects are scattered across the heap, not in one block like C++. That matters later for speed, and we'll come back to it in the optimization lesson.

If a box is still `null`, following it crashes:

```java
arr[2].x = 9;     // ✗ NullPointerException: arr[2] is null
```

Create the object first: `arr[2] = new Point(0, 0);`. Also, `x` is an `int`, so it can never be `null`. Only references can be `null`.

### Check

```java
// inside main
Point[] arr = new Point[3];
arr[0] = new Point(1, 2);
arr[1] = new Point(1, 2);
arr[2] = arr[0];
arr[1].x = 50;
```

1. How many `Point` objects exist?
2. After the last line, what are `arr[0].x`, `arr[1].x`, and `arr[2].x`?

**Answers:**

1. **2** `Point` objects (two `new Point(...)`), plus 1 array object that holds the references.
2. `arr[0].x` is **1**, `arr[1].x` is **50**, and `arr[2].x` is **1**. `arr[2] = arr[0];` copied the address, so boxes 0 and 2 share one object. Box 1 has its own object.

```
arr
┌──────────────┐
│ [0] ref ─────┼──┬──► Point #1 { 1, 2 }
│ [1] ref ─────┼──┼──► Point #2 { 50, 2 }
│ [2] ref ─────┼──┘
└──────────────┘
```

---
