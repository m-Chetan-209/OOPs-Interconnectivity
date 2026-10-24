# C++ Classes and Memory

A C++ class is the struct you already know, plus methods, access control, and automatic setup and cleanup. In short: **a class is a struct that carries its functions.**

```cpp
class Point {
public:
    int x;
    int y;

    Point(int a, int b) { x = a; y = b; }   // constructor
    void move(int dx) { x += dx; }          // method
};

int main() {
    Point p(3, 4);   // stack object (a variable inside a function)
    p.move(5);       // p is now {8, 4}
}
```

Methods are not stored inside the object. They live once in the program's code, shared by all objects.

## Type vs object

- **Type (class):** the blueprint. It takes no memory by itself and only describes the layout.
- **Object:** one actual variable built from the blueprint, with real bytes in memory.

```cpp
int main() {
    Point p(3, 4);
    //  │    │  └── constructor arguments
    //  │    └───── object (instance) name
    //  └────────── class name / type
}
```

Creating an object from a class is called **instantiation**.

---

## `this`: the `self` from C

```cpp
void move(int dx) { x += dx; }
```

The compiler turns this into the C version:

```c
void Point_move(Point *this, int dx) { this->x += dx; }
```

`p.move(5)` becomes `Point_move(&p, 5)`. Inside a method, writing `x` secretly means `this->x`. C++ makes this implicit.

```cpp
int main() {
    Point a(1, 2);
    Point b(10, 20);
    a.move(5);        // a is {6, 2}, b is still {10, 20}
}
```

- **`a` and `b` afterwards:** `a` is `{6, 2}`, `b` is still `{10, 20}`.
- **What `this` points to during `a.move(5)`:** `&a`.

---

## Constructors

A constructor is a function with the **same name as the class and no return type**. It runs automatically when the object is created.

```cpp
class Point {
public:
    int x;
    int y;

    Point(int a, int b) { x = a; y = b; }   // constructor
    void move(int dx) { x += dx; }
};

int main() {
    Point p(3, 4);   // p is the object
}
```

The constructor does not create the object's memory. The memory is reserved first, and the constructor then fills it in. The call `Point p(3, 4);` does this:

1. Reserve 8 bytes on the stack for `p`.
2. Copy 3 and 4 into the parameters `a` and `b`.
3. Run the body: `x = a;` and `y = b;` copy the values into `p`.
4. The constructor ends, and `a` and `b` vanish.

### Why `x = a` and `y = b`?

- `a` and `b` are **parameters**: throwaway boxes holding 3 and 4, living only while the constructor runs.
- `x` and `y` are the **fields**: the 8 bytes that belong to `p`.

`a` and `b` vanish when the constructor ends. If you don't copy them into `x` and `y`, `p` never receives them.

---

## Destructors

A destructor is a function that runs automatically **right before the object dies**. It is written like the constructor, with a `~` in front and no parameters.

```cpp
class Point {
public:
    int x;
    int y;

    Point(int a, int b) { x = a; y = b; }
    ~Point() { /* runs when p dies */ }
};

void f() {
    Point p(5, 6);   // constructor runs here
}                    // destructor runs here, then p's 8 bytes vanish
```

With several objects, each one runs its own destructor, in reverse order of creation:

```cpp
void f() {
    Point p(5, 6);   // p is born → constructor runs
    Point q(1, 2);   // q is born → constructor runs
}                    // function ends → q dies, then p → destructor runs for each
```

---

## Destructors and the heap

Stack memory vanishes automatically at `}`. Heap memory (from `new`) stays reserved until `delete`. A destructor is how an object frees heap memory it owns.

```cpp
class Holder {
public:
    int *p;                              // pointer field (8 bytes, on the stack with the object)

    Holder()  { p = new int(42); }       // constructor: ask heap for 4 bytes, remember the address
    ~Holder() { delete p; }              // destructor: give those 4 bytes back
};

void f() {
    Holder h;      // born
}                  // dies
```

`p` is on the **stack** because it is just a variable declared in the function. The `42` is on the **heap** because it was created by `new`.

At the closing `}`, the destructor runs `delete p;`, so the 4 heap bytes are freed.

**Without a destructor:** the pointer vanishes without `delete`, and nothing points to the 42 anymore. That is a **memory leak**.

---

## Example: array on the heap

```cpp
class Buffer {
public:
    int *data;
    Buffer(int n) { data = new int[n]; }   // the n ints (not the number n) go in the HEAP area
    ~Buffer()     { delete[] data; }       // release them
};

void f() {
    Buffer b(100);
}   // b goes out of scope here → destructor runs automatically
```

```
Stack                 Heap
┌─────────────┐      ┌────────────────┐
│ b.data ─────┼────► │ 100 ints       │
└─────────────┘      └────────────────┘
   (8 bytes)            (400 bytes)
```

`sizeof(Buffer)` is just 8, the pointer. The 400 bytes are separate, on the heap.

### What does `Buffer b(100);` really mean?

```cpp
Buffer b(100);
```

This line does two things:

1. **Declaration:** it reserves space for the object on the **stack**: 8 bytes, just the `data` pointer. That space holds garbage right now.
2. **Constructor call:** `100` is passed into `n`, and the constructor runs `data = new int[n];`. This reserves **400 bytes on the heap** and stores their address in `b.data`. Now the object is ready to use.

### Constructors with and without parameters

```cpp
Point(int a, int b) { ... }    // needs 2 values
Account() { balance = 0; }     // needs nothing
```

```cpp
int main() {
    Point   p(3, 4);   // constructor takes 2 values, you give them
    Account a;         // constructor takes 0 values, so no parentheses
}
```

In C you had to remember `free`. Here, when the stack object `b` ends, the destructor releases the heap memory for you. This idea, **tying heap memory to an object's lifetime**, is called **RAII** and is the heart of C++ memory management.

## `new`

- `new` reserves space on the **heap**.
- `new` **returns an address**.

```cpp
int main() {
    Point *q = new Point(1, 2);   // q holds the address of a Point, so *q is a Point
}
```

```
Stack              Heap (address 5000)
┌──────────┐      ┌─────────┐
│ q = 5000 │ ───► │ x = 1   │
└──────────┘      │ y = 2   │
  (8 bytes)       └─────────┘
                    (8 bytes)
```

`new Point(1, 2)` reserves 8 heap bytes, runs the constructor on them, and returns their address.

### Why `int *q` would be wrong

```cpp
int main() {
    int *q = new Point(1, 2);   // ✗ type mismatch
}
```

`new Point(1, 2)` returns the address of a `Point`. Putting that into an `int *` is a type mismatch. The type in front of the star must match what `new` creates.

---

## `delete`

Heap memory stays reserved until you free it. `delete` is how you free a heap object.

```cpp
int main() {
    Point *q = new Point(4, 6);
    // ... use q ...
    delete q;
}
```

`delete q;` does two things, **in this order**:

1. **Runs the object's destructor**, with `this` holding the address in `q`.
2. **Frees the 8 heap bytes**, giving them back.

For `Point`, the destructor does nothing visible, so step 1 is easy to miss. If the object held heap memory of its own (like `Buffer`), freeing it is the job of that destructor, triggered by `delete q`. That is why `delete` is safer than a bare `free`, which would not run the destructor.

|                   | Does                                         |
|-------------------|----------------------------------------------|
| `new Point(4, 6)` | reserve heap bytes, then run the constructor |
| `delete q`        | run the destructor, then free the heap bytes |

`delete` frees the **heap object**, not `q`. The pointer `q` is a stack variable and vanishes on its own at `}`.

Pairs: `new` goes with `delete`, and `new[]` goes with `delete[]`.

### Forgetting `delete`

```cpp
void f() {
    Point *q = new Point(4, 6);
}   // q vanishes, the 8 heap bytes stay reserved → memory leak
```

### What `q` holds after `delete`

```cpp
int main() {
    Point *q = new Point(4, 6);   // say q is 5000
    delete q;                     // q still holds 5000, but the memory is freed
}
```

`q` still holds the number 5000, but the memory there is no longer yours. Using `q->x` now is **undefined behavior**: it might crash, or it might silently read garbage. A pointer in this state is called a **dangling pointer**.

A common habit is to write `q = nullptr;` right after `delete q;`, so the pointer visibly points to nothing.

```cpp
void f() {
    Point *q = new Point(1, 2);
    delete q;      // q is now dangling: don't touch q->x
}                  // q vanishes, the dangling pointer is gone
```

Once the function closes with `}`, `q` itself is gone, so that dangling pointer no longer exists. (A **copy** of the address stored in a longer-lived pointer would still dangle.)

### Summary

- `new Point(1, 2)`: reserve heap bytes, run the constructor, return the address
- `delete q`: run the destructor, free the heap bytes
- Forget `delete`: memory leak
- Use `q` after `delete`: dangling pointer, undefined behavior

---

## `public` vs `private`

### The problem: everything public

```cpp
class Account {
public:
    int balance;
};

int main() {
    Account a;
    a.balance = 100;
    a.balance = -999999;   // nothing stops this
}
```

Anyone can put nonsense into `balance`. If the class wants a rule like "balance never goes negative," it has no way to enforce it.

### The fix: `private`

```cpp
class Account {
private:
    int balance;                 // only Account's own methods can touch this

public:
    Account()            { balance = 0; }
    void deposit(int n)  { if (n > 0) balance += n; }
    int  get()           { return balance; }
};
```

- **`private`**: only the class's own methods may read or write it.
- **`public`**: any code may use it.

Now from outside the class:

```cpp
int main() {
    Account a;
    a.deposit(50);       // ✓ allowed: deposit is public
    a.balance = -999;    // ✗ compile error: balance is private
}
```

The outside code must go through `deposit`, which checks `n > 0` first. The rule is enforced in one place: the class.

The test for any line: **who is writing it, a method of the class or someone else?** `balance = 0;` in the constructor is allowed, because the constructor belongs to the class. `a.balance = 5;` in outside code is not.

If you write neither word in a `class`, the default is `private`. In a `struct`, the default is `public`.

### Key points

- `public`: any code may use it. `private`: only the class's own methods may.
- `private` does **not** mean "nobody can change this." It means **only code inside the class can change it**.
- It is a **compile-time rule**. The compiler rejects illegal access, and the running program never sees the words `public` or `private`.
- It adds **no memory and no speed cost**. Fields take the same memory whether public or private, and methods take no per-object memory.
- It protects against accidental misuse, not deliberate tricks. It is not security.

---
