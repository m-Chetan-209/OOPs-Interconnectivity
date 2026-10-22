# C++ Classes and Memory

A C++ class is the struct you already know, plus methods, access control, and automatic setup and cleanup.
In short: **a class is a struct that carries its functions.**

```cpp
class Point {
public:
    int x;
    int y;

    Point(int a, int b) { x = a; y = b; }   // constructor
    void move(int dx) { x += dx; }          // method
};

Point p(3, 4);   // stack object
p.move(5);       // p is now {8, 4}
```

Methods are not stored inside the object. They live once in the program's code, shared by all objects.

## Type vs object

- **Type (class):** the blueprint. It takes no memory by itself and only describes the layout.
- **Object:** one actual variable built from the blueprint, with real bytes in memory.

```cpp
Point p(3, 4);
//  │    │  └── constructor arguments
//  │    └───── object (instance) name
//  └────────── class name / type
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
Point a;  a.x = 1;  a.y = 2;
Point b;  b.x = 10; b.y = 20;
a.move(5);
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

Point p(3, 4);   // p is the object
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
    Buffer(int n) { data = new int[n]; }   // acquire heap memory
    ~Buffer()     { delete[] data; }       // release it
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

In C you had to remember `free`. Here, when the stack object `b` ends, the destructor releases the heap memory for you. This idea, **tying heap memory to an object's lifetime**, is called **RAII** and is the heart of C++ memory management.
