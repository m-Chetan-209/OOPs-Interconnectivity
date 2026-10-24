# Inheritance, Polymorphism and Memory

## Terms

| Term | Meaning |
|---|---|
| Instance | an object |
| Field / instance variable | a variable declared in the class body |
| Method | a function inside a class |
| Base / derived class | the class inherited from / the class with the colon |
| Encapsulation | `private` and bundling |
| Inheritance | derived class includes the base part |
| Polymorphism | same call, behavior chosen by the real object |
| Abstraction | abstract class says *what*, children say *how* |

---

## Inheritance as memory layout

**Inheritance means a new class starts as a copy of an existing class's layout, then adds more.**

```cpp
class Animal {
public:
    int age;
};

class Dog : public Animal {   // Dog inherits from Animal
public:
    int tricks;
};

int main() {
    Dog d;
}
```

`Animal` is the **base class** and `Dog` is the **derived class**.

**Memory:** the base part comes first, then the new fields, like a struct with a struct at its start.

```
Dog d (on the stack)
Offset  Field
0       age      ← the Animal part (inherited)
4       tricks   ← Dog's own field
```

`sizeof(Dog)` is 8. Nothing hidden is added. `d.age` works because `age` really is inside `d`.

---

## Where do objects go?

- Declared **inside a function** (even `main`): **stack**
- Created with **`new`**: **heap**
- Declared **outside every function**: **global area** (rare)

### Doubt killer

```cpp
#include <iostream>
using namespace std;

class Point {
public:
    int x;
    int y;
    Point(int a, int b) { x = a; y = b; }
};

Point g(9, 9);                  // line A: written outside every function

int main() {
    Point a(1, 2);              // line B
    Point *b = new Point(3, 4); // line C
    delete b;                   // line D
    return 0;
}
```

**Questions.** For each one, answer stack, heap, or global area, with a short reason (how was it created?):

1. Line A: `g`
2. Line B: `a`
3. Line C: the pointer `b`
4. Line C: the object that `b` points to
5. After line D runs, is the object `b` pointed to still reserved? Does `b` itself still exist?
6. How many `Point` objects get created in the whole program, and how many bytes is each?

**Answers:**

1. **Global area:** declared outside every function.
2. **Stack:** a variable inside `main`.
3. **Stack:** a variable inside `main`.
4. **Heap:** created by `new`.
5. The heap object is freed, but `b` still exists (holding a dangling address) until `main` ends.
6. **3 objects** (`g`, `a`, and the unnamed heap object), **8 bytes** each.

### Where is the object in `Point *b = new Point(3, 4);`?

The object is the part `new Point(3, 4)`. It has **no name**. `new` creates it on the heap and hands back its address, and the only way to reach it is through the pointer `b`.

```cpp
Point *b = new Point(3, 4);   // x = 3, y = 4
//     │      └── the object (unnamed, on the heap)
//     └───────── the pointer (named b, on the stack)
```

`new Point c(3, 4);` isn't valid, because `new` never takes a name. The name belongs to the pointer, not the object. You reach the object with `b->x` or `(*b).x`.

---

## Building a derived object

Last time we saw that a `Dog` is an `Animal` part (offset 0) followed by Dog's own fields. So building a `Dog` means building both parts.

**The base part is built first, then the derived part. Destruction goes in reverse.**

```cpp
#include <iostream>
using namespace std;

class Animal {
public:
    int age;
    Animal(int a) { age = a; cout << "Animal born\n"; }
    ~Animal()     { cout << "Animal dies\n"; }
};

class Dog : public Animal {
public:
    int tricks;
    Dog(int a, int t) : Animal(a) { tricks = t; cout << "Dog born\n"; }
    ~Dog()                        { cout << "Dog dies\n"; }
};

int main() {
    Dog d(3, 5);
    Dog e(1, 2);
}
```

**Mnemonic:** `(derived class) : (base class)`. The `: Animal(a)` part means "before my body runs, call Animal's constructor with `a`."

**Questions:**

1. How many lines print in total, and what are the first four?
2. Who dies first, and why (briefly)?

**Answers:**

1. 8 lines. The first four are `Animal born`, `Dog born`, `Animal born`, `Dog born`.
2. `e` dies first, because it was created last, and later objects may depend on earlier ones (like a stack of plates). Inside each object, `Dog dies` prints before `Animal dies`.

Full output:

```
Animal born
Dog born
Animal born
Dog born
Dog dies
Animal dies
Dog dies
Animal dies
```

---

## Inherited methods and overriding

Methods are inherited too, not just fields.

```cpp
#include <iostream>
using namespace std;

class Animal {
public:
    int age;
    void speak() { cout << "..."; }
};

class Dog : public Animal {
public:
    int tricks;
    void speak() { cout << "Woof"; }   // Dog's own version
};

int main() {
    Dog d;
    d.speak();
}
```

- If `Dog` has **no** `speak`, then `d.speak()` runs `Animal`'s version, with `this` pointing at `d`.
- If `Dog` defines its own `speak`, then `d.speak()` runs **Dog's** version. This is called **overriding**.

**Memory:** nothing changes. Methods are not stored in the object, so `sizeof(Dog)` is still 8 (`age` + `tricks`). Inheriting or overriding a method costs zero bytes per object.

The compiler picks the function when it compiles, by looking at the object's type: `d` is a `Dog`, so `d.speak()` becomes `Dog_speak(&d)`.

**Questions:**

1. What does `d.speak()` print?
2. If `Dog` had no `speak` at all, what would print, and what would `this` point to?
3. What is `sizeof(Dog)`?

**Answers:** 1. `Woof`. 2. `...`, and `this` points to `d`. 3. 8 bytes.

### Example 2: fields actually used

```cpp
#include <iostream>
using namespace std;

class Account {
private:
    int balance;
public:
    Account(int b)      { balance = b; }
    void deposit(int n) { balance += n; }
    int  get()          { return balance; }
};

class Savings : public Account {
private:
    int ratePercent;
public:
    Savings(int b, int r) : Account(b) { ratePercent = r; }
    void addInterest() { deposit(get() * ratePercent / 100); }
};

int main() {
    Savings s(1000, 5);
    s.addInterest();
    cout << s.get();     // prints 1050
}
```

**Check:**

```cpp
Savings s(200, 10);
s.addInterest();
```

1. What is the balance afterwards?
2. What is `sizeof(Savings)`, and which class does `deposit(...)` belong to?

**Answers:** 1. 220. 2. 8 bytes. `deposit` belongs to `Account`, the base class, where it's defined.

- `addInterest` belongs to `Savings`, the derived class.
- `addInterest` calls `deposit`. It doesn't own it.
- So the derived method is `addInterest`, and `deposit` and `get` are inherited.

`balance` is `private` to `Account`, so even `Savings` methods must go through `get()` and `deposit()`. The bytes are inside `s`, but the compiler blocks the access.

### What is stored in the object?

| Thing | Stored in the object? |
|---|---|
| Fields (`balance`, `ratePercent`) | Yes, they make up the size |
| Parameters (`n`, `b`, `r`) | No, they vanish when the function ends |
| Local variables (inside a function body) | No, they vanish when the function ends |
| Methods (`deposit`, `get`) | No, they live once in the program's code |

```cpp
class Rect {
    int w;
    int h;
public:
    Rect(int a, int b) { int area = a * b; w = a; h = b; }
};
```

**Check:** which names are fields, which are parameters, and which is a local variable? What is `sizeof(Rect)`?

**Answer:** `w` and `h` are fields, `a` and `b` are parameters, and `area` is a local variable (declared inside the constructor's `{ }` body, so it exists only while the constructor runs). `sizeof(Rect)` is 8.

---

## A pointer to the base type

```cpp
#include <iostream>
using namespace std;

class Animal {
public:
    int age;
    void speak() { cout << "..."; }
};

class Dog : public Animal {
public:
    int tricks;
    void speak() { cout << "Woof"; }   // Dog's own version
};

int main() {
    Dog d;
    d.speak();         // Woof
    Animal *p = &d;    // p is an Animal pointer, but holds the address of a Dog
    p->speak();        // which speak runs? ... (prints "...")
}
```

**The new thing:** a pointer to the base type is allowed, because a `Dog` starts with an `Animal` part. This works with or without `virtual`.

### The key idea: `p` is a window

`p` holds the address of `d`, but its **type says Animal**. So through `p`, the compiler treats the object as an `Animal` and can't see `tricks` or Dog's version of `speak`.

```
d in memory:    [ age | tricks ]
p's window:     [ age ]           ← p only "knows" this much
```

### Why it prints `...` and not `Woof`

The compiler decides which `speak` to call **before the program runs**. At that moment it only knows `p`'s declared type, `Animal *`. It doesn't know a Dog will be there.

```cpp
d.speak();    →  Dog_speak(&d)       // d's type is Dog
p->speak();   →  Animal_speak(p)     // p's type is Animal
```

### Why it's a problem

With a list of different animals, every call would be fixed to `Animal`'s version:

```cpp
Dog d;
Cat c;
Animal *list[2] = { &d, &c };   // both fit, each starts with an Animal part

for (int i = 0; i < 2; i++)
    list[i]->speak();           // we WANT: Woof, Meow. We GET: ..., ...
```

---

## The fix: `virtual`

```cpp
#include <iostream>
using namespace std;

class Animal {
public:
    int age;
    virtual void speak() { cout << "..."; }   // ← virtual
};

class Dog : public Animal {
public:
    int tricks;
    void speak() { cout << "Woof"; }
};

int main() {
    Dog d;
    Animal *p = &d;
    p->speak();      // now prints: Woof
}
```

| | Decided by | When |
|---|---|---|
| normal method | type of the pointer | compile time |
| `virtual` method | type of the actual object | run time |

This is **polymorphism**: the same line, `p->speak()`, does different things depending on the real object.

---

## Inside `virtual`: the vptr and the vtable

When a class has a virtual method, the compiler adds a **hidden pointer field** at the very start of every object. It's called the **vptr**. It holds the address of a small table, the **vtable**, that lists which functions that class uses.

```cpp
class Animal {
public:
    int age;
    virtual void speak() { cout << "..."; }
};

class Dog : public Animal {
public:
    int tricks;
    void speak() { cout << "Woof"; }
};

int main() {
    Dog d;
    Animal *p = &d;
    p->speak();
}
```

### The memory picture

```
d (a Dog, on the stack)                    Program's fixed area
Offset 0:   vptr  ─────────────────────►  Dog's vtable
Offset 8:   age                            ┌───────────────┐
Offset 12:  tricks                         │ speak → Dog's │
                                           └───────────────┘

Animal's vtable (separate, for real Animals):
                                           ┌─────────────────┐
                                           │ speak → Animal's│
                                           └─────────────────┘
```

Two facts about the table:

- There is **one vtable per class**, not per object. All `Dog` objects point to the same one.
- It's built by the compiler and lives in the program's fixed memory, not in the object.

### What `p->speak()` does now

1. Follow `p` to the object (the Dog).
2. Read the **vptr** at offset 0 of that object.
3. Follow the vptr to the vtable, and read the entry for `speak`.
4. Call that function, with `this` = `p`.

The type `Animal *` is only used to know *which slot* in the table to read. Which function sits in that slot depends on the **object's** vtable, so a Dog gives "Woof".

### The cost

- Every object grows by one **8-byte pointer** (on a 64-bit machine), plus any padding it causes.
- Each virtual call takes an extra step (read vptr, read table) instead of jumping straight to a fixed function.

This is the first time inheritance actually changes `sizeof`:

```cpp
class Animal { public: int age; };
// sizeof = 4

class Animal { public: int age; virtual void speak() {} };
// vptr 0-7, age 8-11, padding 12-15  →  sizeof = 16
```

For `Dog` (with `tricks`): `vptr` at 0, `age` at 8, `tricks` at 12, so `sizeof(Dog)` is 16.

### What you write vs what the compiler builds

| You write | The compiler builds |
|---|---|
| `virtual void speak()` | a vtable for each class with a slot for `speak` |
| `Dog d;` | an object with a hidden vptr at offset 0, pointed at Dog's vtable |
| `p->speak()` | "read the object's vptr, find the slot, call that function" |

### Counting

- **vtables = number of classes** that have virtual methods.
- **vptrs = number of objects** of those classes.

```cpp
class Animal { public: virtual void speak() {} };
class Dog : public Animal { public: void speak() {} };
class Cat : public Animal { public: void speak() {} };

int main() {
    Dog d1, d2;
    Cat c1;
}
```

Here there are **3 vtables** (`Animal`, `Dog`, `Cat`) and **3 vptrs** (`d1`, `d2`, `c1`).

---

## Why destructors need `virtual` too

Now that `p->speak()` works through a base pointer, the same question applies to `delete`.

```cpp
#include <iostream>
using namespace std;

class Animal {
public:
    ~Animal() { cout << "Animal dies\n"; }
};

class Dog : public Animal {
public:
    int *toys;
    Dog()  { toys = new int[10]; }          // 40 heap bytes
    ~Dog() { delete[] toys; cout << "Dog dies\n"; }
};

int main() {
    Animal *p = new Dog;
    delete p;
}
```

`p` is an `Animal *`. Just like `speak` earlier, the compiler decides which destructor to call from the **pointer's type**. So `delete p;` runs only `~Animal`, and `~Dog` never runs. The 40 bytes that `toys` points to are never freed: a **memory leak**. (Strictly, the C++ rules call this undefined behavior, and the leak is what you'd see in practice.)

Output:

```
Animal dies
```

### The fix: a virtual destructor

```cpp
class Animal {
public:
    virtual ~Animal() { cout << "Animal dies\n"; }   // ← virtual
};
```

Now `delete p;` reads the object's vptr, finds that it's really a `Dog`, and runs `~Dog` first, then `~Animal`:

```
Dog dies
Animal dies
```

The 40 bytes are freed.

**Rule of thumb:** if a class has any virtual method, or will be used through a base pointer, make its destructor `virtual`.

---

## Abstract classes (pure virtual)

Sometimes a base class makes no sense as a real object. What sound does a plain "Animal" make? There isn't one. C++ lets you say that directly:

```cpp
#include <iostream>
using namespace std;

class Animal {
public:
    int age;
    virtual void speak() = 0;          // "= 0" means: no body here
    virtual ~Animal() {}
};

class Dog : public Animal {
public:
    void speak() { cout << "Woof"; }   // Dog must provide it
};

int main() {
    // Animal a;           // ✗ compile error: Animal is abstract
    Dog d;                 // ✓ fine
    Animal *p = &d;        // ✓ fine: a pointer to Animal is allowed
    p->speak();            // Woof
}
```

- `= 0` makes `speak` a **pure virtual** method. It has no body in `Animal`.
- A class with at least one pure virtual method is **abstract**. You can't create objects of it, but you can use pointers to it.
- Every derived class must override all pure virtual methods, or it is abstract too.

This is **abstraction**.

### The problem

```cpp
class Animal {
public:
    virtual void speak() { cout << "..."; }
};

int main() {
    Animal a;       // allowed
    a.speak();      // prints "..."
}
```

This compiles, but it makes no sense. A plain "Animal" has no real sound. Only `Dog` and `Cat` do. We want C++ to **stop** anyone from writing `Animal a;`.

### The one change

Replace the body with `= 0`:

```cpp
class Animal {
public:
    virtual void speak() = 0;    // no body. Children must write one.
};
```

`= 0` means: "`Animal` doesn't know how to speak. Every child class must supply its own `speak`."

Because `Animal` now has a method with no body, it is **incomplete**, so C++ refuses to build an object from it:

```cpp
Animal a;    // ✗ compile error
```

A class like this is called **abstract**.

### Children fill it in

```cpp
class Dog : public Animal {
public:
    void speak() { cout << "Woof"; }    // Dog writes the body
};

Dog d;       // ✓ allowed: Dog is complete
```

`Dog` supplied the body, so `Dog` is complete and can be built.

### Why an error is the point

The error is a **safety net**. It catches your mistake while you're writing the program, instead of letting it hide until someone runs it.

Say you add a new `Cat` and forget to write its `speak`.

**Without `= 0`** (`Animal` has a normal `speak` that prints `...`):

```cpp
class Cat : public Animal {
public:
    int lives;
    // forgot speak!
};

Cat c;
c.speak();    // compiles fine, silently prints "..."
```

No error. The program runs, and a cat prints `...`. The bug is hidden, and you may only notice after it has shipped.

**With `= 0`:**

```cpp
Cat c;        // ✗ compile error: Cat is still abstract
```

The compiler stops you immediately, and you find out **while compiling**, before the program ever runs. `= 0` costs nothing in memory. It only changes what the compiler allows.

### Pointers to an abstract class still work

```cpp
Animal a;          // ✗ builds a NEW, separate Animal object. Not allowed.
Animal *p = &d;    // ✓ builds nothing. Points at the Dog that already exists.
```

`Animal *p = &d;` **still works**, because `d` was created as a complete `Dog`, and `p` is only an address. The `Animal` part exists only as the first bytes inside `d`. This also works without `virtual`, because a `Dog` always starts with an `Animal` part.

---
