# C++ Copying, Moving and Ownership

All C++ in these notes was compiled with g++ and run, and the outputs shown are real. Java behavior is standard Java (not run here).

---

## 1. What `Buffer b2 = b1;` copies

```cpp
class Buffer {
public:
    int *data;
    Buffer(int n) { data = new int[n]; }
};

Buffer b1(3);
b1.data[0] = 7;
Buffer b2 = b1;          // make b2 as a copy of b1
```

If you write no copying code, C++ copies an object **field by field**. `Buffer` has one field, the pointer `data` (8 bytes), so C++ copies **those 8 bytes**: the **address**, not the 12 bytes of heap data it points to.

```
b1 (stack)               Heap
┌────────────┐          ┌────────────┐
│ data = 5000│ ───────► │ 7, 0, 0    │
└────────────┘     ▲    └────────────┘
b2 (stack)         │
┌────────────┐     │
│ data = 5000│ ────┘
└────────────┘
```

Output of checking it:

```
same address? yes
b1.data[0] = 99        ← after b2.data[0] = 99
```

There are **two `Buffer` objects** (on the stack, 8 bytes each) but **one heap array**. This is a **shallow copy**: the surface is copied, and the heap data is shared.

---

## 2. The crash: a double delete

Add the destructor from Lesson 3:

```cpp
class Buffer {
public:
    int *data;
    Buffer(int n) { data = new int[n]; }
    ~Buffer()     { delete[] data; }
};

void f() {
    Buffer b1(3);
    Buffer b2 = b1;       // both hold the same address
}                         // } → both objects die
```

Running it, with a print added to the constructor and destructor:

```
allocated 0x55a60c5352b0
freeing   0x55a60c5352b0
freeing   0x55a60c5352b0
free(): double free detected in tcache 2
Aborted
```

1. `b1` is built, and `new` allocates one array.
2. `b2 = b1` copies the pointer, so nothing new is allocated (only one "allocated" line).
3. At `}`, `b2` dies first (reverse order), and its destructor frees the array.
4. Then `b1` dies, and its destructor frees **the same address again**. The program aborts.

The order doesn't matter: whichever object dies second does the double free.

### What the destructor deletes

`delete[] data;` frees **the heap array**, not the address. The pointer `data` is a normal field inside the object and **keeps holding the same number**:

```
Before delete[]:   data = 5000 ──► [ array ]
After delete[]:    data = 5000 ──► (freed memory, no longer yours)    ← a dangling pointer
```

`b1` still held 5000, because it had its own copy of the address and nobody told it the memory was gone. The pointer itself disappears when its object dies.

### The other languages

| | A shallow copy |
|---|---|
| **Java** | no crash: one object, and the garbage collector frees it once |
| **Python** | no crash: the reference count tracks the sharing, and the object is freed once at 0 |
| **C++** | **crash**: nobody is counting, and each destructor frees blindly |

---

## 3. The copy constructor: a deep copy

Tell C++ how to copy a `Buffer`: allocate a **new** array and copy the numbers into it.

```cpp
class Buffer {
public:
    int *data;
    int n;                                    // the copy needs the size

    Buffer(int n) : n(n) { data = new int[n]; for (int i = 0; i < n; i++) data[i] = 0; }

    Buffer(const Buffer &other) {             // copy constructor
        n = other.n;
        data = new int[n];                    // a NEW array for this copy
        for (int i = 0; i < n; i++)
            data[i] = other.data[i];          // copy the numbers over
    }

    ~Buffer() { delete[] data; }
};
```

- `Buffer(const Buffer &other)` takes **another `Buffer`**. The `&` makes `other` just **another name for the original**, like a Java reference, so passing it copies nothing. Without the `&`, calling the copy would need a copy, forever. `const` promises not to change the original.
- `int n` is new: the size must be stored. That's why `sizeof(Buffer)` is now **16** (8-byte pointer, 4-byte int, 4 bytes of padding).

```
same address? no
b1.data[0] = 7, b2.data[0] = 99
```

```
b1                        Heap
┌────────────┐           ┌────────────┐
│ data = 5000│ ────────► │ 7, 0, 0    │
│ n = 3      │           └────────────┘
└────────────┘
b2                        ┌────────────┐
┌────────────┐           │ 99, 0, 0   │   ← b2's OWN array
│ data = 6000│ ────────► └────────────┘
│ n = 3      │
└────────────┘
```

Two objects, two separate arrays, and two `delete[]` calls, each on its own address, so it's safe. This is a **deep copy**, the same idea as Python's `copy.deepcopy`.

---

## 4. Move: transferring ownership

A deep copy is safe but expensive. I timed copying a `Buffer` of 20 million ints (80 MB): about **40 to 50 ms**. Often the original isn't needed any more, for example when a function returns a big `Buffer` that is about to be destroyed anyway.

A **move** takes the heap array from the old object instead of duplicating it:

```cpp
Buffer(Buffer &&other) {          // move constructor
    n = other.n;
    data = other.data;            // take over the pointer: no new array
    other.data = nullptr;         // the old object now owns nothing
    other.n = 0;
}

Buffer copyB = b1;                // copy constructor (b1 stays intact)
Buffer moveB = std::move(b1);     // move constructor (b1 is emptied)
```

```
Before:                          After the move:
b1.data ──► [ 80 MB array ]      b1.data = nullptr
                                 moveB.data ──► [ the same 80 MB array ]
```

On the same buffer, the move took about **0.00007 ms**, effectively instant.

- **`other.data = nullptr` is essential.** When `b1` dies, its destructor runs `delete[] nullptr`, which is safe and does nothing. Without it, both objects would hold the same address, and we would be back to the double delete.
- `Buffer &&other` means "another `Buffer` that we are **allowed to steal from**".
- `std::move(b1)` moves nothing itself. It is a **label**: "I am finished with `b1`, you may steal from it."
- After a move, `b1` is valid but **empty** (`data` is `nullptr`, `n` is 0). Don't use its data again.

### What a move actually copies

A move still builds a new object and copies a few small things: the 4-byte size and the 8-byte address, about **12 bytes**. What it does **not** copy is the 80 MB the address points to. That array never moves, and only the right to own it changes hands.

| | After a copy | After a move |
|---|---|---|
| New object | has its own full data | has the data |
| Old object | still has its data, unchanged | **emptied** |
| Heap arrays | **two** | **one** |

> **The crux:** in a move, the big data stays exactly where it is. Only the **ownership** (the pointer) passes to the new object.

### How the other languages compare

| | Cost of `t = s` for a huge object |
|---|---|
| **Java** | copies only the reference, always cheap, like a move |
| **Python** | copies only the arrow, always cheap, like a move |
| **C++** | a **copy** by default, and you choose when to **move** |

---

## 5. Smart pointers

`Buffer` needed a destructor, a copy constructor, and a move constructor, and getting any one wrong leaks or crashes. C++ ships ready-made classes that do it correctly: **smart pointers**, in `<memory>`. A smart pointer holds a pointer and **frees it automatically** with the same destructor trick as `Buffer` (RAII, Lesson 3).

### `unique_ptr`: one owner

```cpp
unique_ptr<Noisy> a = make_unique<Noisy>();   // creates the object on the heap
unique_ptr<Noisy> b = std::move(a);           // ownership moves to b
```

```
  Noisy built
  a is empty, b is owning
  Noisy freed             ← freed automatically at the closing }
```

- `make_unique<Noisy>()` is the safe replacement for `new Noisy`.
- **Exactly one owner.** `unique_ptr<int> b = a;` is a **compile error** ("use of deleted function"). You can only **move**, so two owners can never exist and the double delete can't happen.
- It costs **8 bytes**, the same as a raw pointer.

### `shared_ptr`: many owners, counted

```cpp
shared_ptr<Noisy> s1 = make_shared<Noisy>();
shared_ptr<Noisy> s2 = s1;                    // copying is allowed: the count goes up
```

```
  count = 1
  count = 2             ← s1 and s2 share it
  count = 1 (s2 gone)
  Noisy freed           ← freed when the count reaches 0
```

It costs **16 bytes** (the pointer plus a pointer to the counter).

| | Python | C++ `shared_ptr` |
|---|---|---|
| Count lives | in the object's header | in a control block next to the object |
| A new owner | count goes up | count goes up |
| An owner goes away | count goes down | count goes down |
| At 0 | freed immediately | freed immediately |

### The cycle problem

`shared_ptr` has the same blind spot as Python's reference counting. Two nodes holding `shared_ptr`s to each other keep each other's counts above 0:

```
no cycle:
  Node freed
  Node freed
cycle:
end of program         ← no "Node freed" at all: a leak
```

Python has a backup cyclic garbage collector, and **C++ does not**. In C++ you break such cycles with `weak_ptr`, a non-owning pointer that doesn't add to the count.

### Which to use

| Situation | Use |
|---|---|
| One clear owner (the normal case) | `unique_ptr` |
| Several parts of the program truly share ownership | `shared_ptr` |
| Rarely, when you need manual control | a raw pointer with `new` / `delete` |

Modern C++ rarely writes `new` and `delete` by hand. The raw version is what you learn so you understand what smart pointers do for you.

---

## 6. Copying across all four languages

| | What `b = a` copies | A real copy of the object |
|---|---|---|
| **C** | the struct's bytes (a pointer field is copied as an **address**) | you write the copying code |
| **C++** | the whole object, field by field (a pointer field copies the **address**) | copy constructor (deep), `std::move` (steal), smart pointers |
| **Java** | only the **reference** (one object, two names) | a copy constructor or `clone()`, and arrays of objects need care |
| **Python** | only the **arrow** (one object, two names) | `copy.copy` (shallow) or `copy.deepcopy` (deep) |

The common thread:

- **Sharing is cheap but dangerous** (one object, many names).
- **Copying is safe but expensive** (new memory, all fields copied).
- **Shallow** copies the surface and shares what's inside. **Deep** copies every level.
- **Move** hands over ownership, with no duplication of the data.

C++ makes you choose explicitly. Java and Python pick "share" for you and free the memory automatically, so sharing never crashes there.

### A gap to know about

`Buffer b2 = b1;` makes a **new** object. A different line, `b2 = b1;` on a `b2` that **already exists**, uses the **copy assignment operator**. The default one has the same shallow-copy problem, and it also leaks `b2`'s old array. A class that manages its own memory needs the destructor, copy constructor, and copy assignment (plus the move versions) written together. That's the **rule of three** (or **five**, with the moves). Smart pointers let you avoid writing any of it.

---

## Summary

1. The default copy of a pointer field copies the **address**, so two objects share one heap array (**shallow**).
2. With a destructor, a shallow copy causes a **double delete** and a crash.
3. `delete[] data` frees the array, not the address. The pointer keeps its number and dangles.
4. A **copy constructor** allocates a new array and copies the numbers (**deep copy**). The class must store the size.
5. A **move constructor** copies the pointer and sets the old one to `nullptr`. It's instant, and safe because `delete[] nullptr` does nothing. The data stays put, and only ownership changes.
6. `unique_ptr` allows one owner (8 bytes, move only). `shared_ptr` counts owners like Python (16 bytes). A `shared_ptr` cycle leaks.
7. In Java and Python, `b = a` copies only the reference, so it's always cheap and shared.
