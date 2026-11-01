# Java: What an Object Looks Like in Memory

Snippets marked `// inside main` are meant to sit inside a `main` method, like in Lesson 5.

---

## The hidden header

In C++, a `Point` with two `int`s was **8 bytes**: just `x` and `y`. In Java, the same class is **24 bytes**:

```java
class Point {
    int x;
    int y;
}
```

Every Java object starts with a hidden **header**, before any of your fields. On a typical 64-bit JVM, the header is 12 bytes:

```
One Point object on the heap
Offset 0-7    mark word     (8 bytes)  ← GC info, lock info, cached hash code
Offset 8-11   class pointer (4 bytes)  ← points to info about "Point"
Offset 12-15  x             (4 bytes)
Offset 16-19  y             (4 bytes)
Offset 20-23  padding       (4 bytes)
                              total = 24 bytes
```

- The **mark word** (8 bytes) holds information for the garbage collector, locks, and the cached hash code.
- The **class pointer** (4 bytes) works like the C++ **vptr**: a hidden pointer at the start of every object saying "I am a `Point`". Java needs it for **every** object, not only virtual ones.
- **Padding:** Java rounds every object up to a multiple of 8 bytes. Here 20 rounds up to 24.

The 8 bytes of real data cost 24 bytes. The extra is the price of garbage collection and runtime type information. Exact sizes vary between JVMs and settings, but 12 bytes of header is the common case.

### Why does the JVM always need the type?

In C++, a plain object carries no type information, because the compiler already knew every type while compiling. Java keeps the type with the object because the JVM uses it constantly:

- **Garbage collector:** to free or move an object, it must know which fields are references.
- **Method calls:** Java looks up the method by the object's real class.
- **Casts and `instanceof`:** these check the real type, and a wrong cast throws `ClassCastException`.

The class pointer isn't literally a vptr. It points to the class's metadata, which contains the table of methods (the vtable). The idea is the same, with one extra hop.

### Check

```java
class Counter {
    int n;
}
```

Size of one `Counter`: header 12 + `n` 4 = 16, already a multiple of 8, so no padding. **16 bytes.**

```java
class Three {
    int a;
    int b;
    int c;
}
```

Size of one `Three`: 12 + 12 = **24 bytes**.

---

## `int` vs `Integer`

Java has primitives (`int`) and class types (`Integer`, the object version of an int). A primitive holds the value itself, and a class type holds an address.

```java
// inside main
int a = 5;          // 4 bytes: the value sits directly in the variable
Integer b = 5;      // a reference (4 bytes) to an object on the heap
```

```
Integer object on the heap
Offset 0-11   header   (12 bytes)
Offset 12-15  value    (4 bytes)   → total 16 bytes
```

The number 5 costs **4 bytes** as an `int`, but **4 + 16 = 20 bytes** as an `Integer` (the reference plus the object).

### Why it matters: arrays

```java
// inside main
int[] a = new int[1000];          // about 4,000 bytes: values side by side
Integer[] b = new Integer[1000];  // 1000 references + 1000 separate objects
```

The `Integer[]` needs about 4,000 bytes of references plus 1000 objects × 16 bytes, so about **20,000 bytes**, around 5 times more, and scattered across the heap. For 1000 numbers, `int[]` uses far less memory.

---

## Java reorders fields for you

In C++ (Lesson 2), the order you wrote the fields decided the padding, so you had to put them in a good order. In Java, the JVM is free to **rearrange the fields inside the object** to waste less space, and it usually does.

What you still control is **how many fields and what types** they are, because every field takes real bytes and the header is added on top.

```java
class Flags {
    int count;
    byte a;
    byte b;
}
```

```
Offset 0-11   header   (12 bytes)
Offset 12-15  count    (4 bytes)
Offset 16     a        (1 byte)
Offset 17     b        (1 byte)
Offset 18-23  padding  (round 18 up to a multiple of 8)
                         total = 24 bytes
```

In C++, the same fields would be 8 bytes. In Java it's 24, because the header adds 12 and the total rounds up to a multiple of 8.

### Check

```java
class Pair {
    int first;
    int second;
    byte tag;
}
```

12 + 4 + 4 + 1 = 21, rounded up to a multiple of 8: **24 bytes**, with 3 bytes of padding.

---

## Inheritance: every method is virtual

In C++ (Lesson 4), a method was only looked up by the real object if you wrote `virtual`. In Java, **every ordinary method is virtual by default**.

```java
class Vehicle {
    int wheels;
    void honk() { System.out.println("..."); }
}

class Car extends Vehicle {         // "extends" is Java's ": public"
    int doors;
    void honk() { System.out.println("Beep"); }
}

public class Demo {
    public static void main(String[] args) {
        Vehicle p = new Car();      // a Vehicle reference holding a Car's address
        p.honk();                   // prints: Beep
    }
}
```

The same setup in C++ without `virtual` printed `...`, because the pointer's type decided:

```cpp
class Vehicle { public: void honk() { cout << "..."; } };     // no virtual
class Car : public Vehicle { public: void honk() { cout << "Beep"; } };
Car c;
Vehicle *p = &c;
p->honk();      // prints "..." (decided by the pointer's type)
```

In Java, the **real object** decides, and it needs no extra per-object cost. In C++, `virtual` added a hidden 8-byte vptr to every object. In Java, every object already has the class pointer in its header, and the JVM uses it to find the right method.

### The layout

Base fields come first, then the derived ones, after the header:

```
Car object
Offset 0-11   header   (12 bytes)
Offset 12-15  wheels   (4 bytes)   ← the Vehicle part
Offset 16-19  doors    (4 bytes)   ← Car's own field
Offset 20-23  padding  (4 bytes)
                         total = 24 bytes
```

### Check

1. What prints in the code above, and what decides it? **"Beep", decided by the type of the real object.**
2. What is the size of one `Car`? **24 bytes.**

---

## Constructors and `super`

In C++ (Lesson 4), building a derived object ran the base constructor first, with the colon syntax:

```cpp
Dog(int a, int t) : Animal(a) { tricks = t; }
```

Java does the same thing with the word `super`:

```java
class Vehicle {
    int wheels;
    Vehicle(int w) {
        wheels = w;
        System.out.println("Vehicle built");
    }
}

class Car extends Vehicle {
    int doors;
    Car(int w, int d) {
        super(w);               // call Vehicle's constructor
        doors = d;
        System.out.println("Car built");
    }
}

public class Demo {
    public static void main(String[] args) {
        Car c = new Car(4, 2);
    }
}
```

Output:

```
Vehicle built
Car built
```

`new Car(4, 2)` creates **one** 24-byte object on the heap, and **two** constructors fill in different parts of it.

### Why the base part is built first

```
Car object (one object on the heap)
header
wheels   ← Vehicle part (Vehicle's constructor fills this)
doors    ← Car part     (Car's constructor fills this)
```

`Car`'s constructor is allowed to **use** the Vehicle part. If it ran first, it could read `wheels` while it still held garbage. So the parent constructor must **finish** before the child touches the parent part.

### The hidden `super()`

If you don't write `super(...)`, Java inserts `super();` for you. That only works if the parent has a constructor with **no parameters**:

```java
class Car extends Vehicle {
    int doors;
    Car(int d) {            // no super(...) written
        doors = d;
    }                       // ✗ error: Java tries super(), but Vehicle needs an int
}
```

C++ has the same rule: without `: Animal(a)`, the base class needs a default constructor.

### There is no destructor

Java has no `~Car()`. Objects are freed by the garbage collector at an unknown time, so there is no moment you can rely on for "cleanup runs now". For things that must be released promptly (files, connections), Java uses a different tool, which we'll cover later.

---

## Statements before `super(...)` (Java 25+)

Up to Java 24, `super(...)` had to be the **first line** of a constructor. In **Java 25 and newer** (flexible constructor bodies), statements may appear before it, as long as they **don't use the object being built**.

```java
class Car extends Vehicle {
    int doors;
    Car(int d, int w) {
        System.out.println("Car built");   // before super: allowed in Java 25
        doors = d;                         // writing to the Car part: allowed
        super(w);
    }
}
```

Output of `new Car(4, 2)` here:

```
Car built
Vehicle built
```
