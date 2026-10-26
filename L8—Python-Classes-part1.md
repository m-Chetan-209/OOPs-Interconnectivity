## Doubt Killer before Beginning

**Every Python object lives on the heap**

| What | Where | Notes |
|---|---|---|
| Your objects (lists, strings, instances of your classes) | Heap | confirmed: anonymous memory, not the Python program's own data area |
| Function code (the compiled bytecode of `def f()`) | Heap, inside a code object | code is data too, so even code is an object in Python |
| Names (`a`, `b`, `lst`) | In a table: a frame for each function call, a dict for globals | each entry is just an arrow to an object |
| `None`, `True`, `False`, small ints (-5 to 256), built-in types like `int` | Data area of the Python program itself | confirmed: these were in the interpreter's own data segment, created once at startup and shared |
| The interpreter's own machine code (written in C) | Code area | not your program, but the program that runs yours |

## Compare all three

| | C++ | Java | Python |
|---|---|---|---|
| Your objects | stack or heap (you choose) | always heap | always heap |
| Variables | hold the value or an address | hold the value (primitive) or reference | always a name (arrow) to an object |

---

# A class with `__init__` and `self`

You know the C++ version of `Point`:

```cpp
class Point {
public:
    int x, y;
    Point(int a, int b) { x = a; y = b; }
};
Point p(3, 4);
```

The Python version:

```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

p = Point(3, 4)
```

Compare the two:

- `class Point:` defines the blueprint (and the class is itself an object).
- `__init__` is Python's constructor. It's called automatically when the object is created, just like `Point(int a, int b)` in C++.
- `self` is the hidden `this` from Lesson 3, but Python makes you write it as the first parameter. `p` is the object, and inside `__init__` it arrives under the name `self`. `self` points to the object.
- `self.x = x` means "on this object, create the field `x` and point it at the value `x`". The left `x` belongs to the object, and the right `x` is the parameter, so it's the "field = parameter" pattern again.

`Point(3, 4)` does two steps, like C++:

1. Python creates an empty object on the heap.
2. It calls `__init__` with that object as `self`, plus 3 and 4, to fill it in.

---

## Methods

A method is a function defined inside the class, with `self` as the first parameter:

```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def move(self, dx):
        self.x += dx

p = Point(3, 4)
p.move(5)
```

What `self.x += dx` does ties back to Lesson 7:

1. `self.x` points to the int object 3, which is immutable.
2. `3 + 5` builds a new int object, 8.
3. The object's field `x` is re-pointed to the 8.

---

## Class variables vs instance variables

So far every field was created with `self.` inside `__init__`, so each object owns its own copy. Python also lets you put a variable in the class itself:

```python
class Car:
    wheels = 4                      # class variable: written in the class body

    def __init__(self, color):
        self.color = color          # instance variable: written on self

a = Car("red")
b = Car("blue")
```

- **Instance variable** (`self.color`): stored in each object, so `a` and `b` have separate ones.
- **Class variable** (`wheels`): stored once, in the class, and shared by every object.

This is the same idea as C++'s `static int wheels;` or Java's `static`. A static member isn't stored in each object. It exists once, which is why `sizeof` didn't include it.

### How Python finds `a.wheels`

When you write `a.wheels`, Python looks in order:

1. In the object `a` itself. Is there an instance variable named `wheels`? No.
2. In the class `Car`. Yes, `wheels` is there, so the answer is 4.

So `a.wheels` and `b.wheels` both reach the one 4 stored in the class.

### The trap: assigning through an object

```python
a.wheels = 6
```

This does not change the class variable. Assigning through `a` creates a new instance variable on `a` only:

```
Car                      a                    b
┌──────────────┐   ┌──────────────────┐   ┌────────────┐
│ wheels ──► 4 │   │ color ──► "red"  │   │ color ...  │
└──────────────┘   │ wheels ──► 6     │   └────────────┘
                   └──────────────────┘
```

Now `a.wheels` finds the object's own `wheels` first (6), while `b.wheels` still falls through to the class (4). This is re-pointing again: you didn't edit the shared 4, you gave `a` its own name for something else.

To change the shared value for everyone, assign on the class:

```python
Car.wheels = 5      # now a and b both see 5, unless they have their own wheels
```

### Check

```python
class Car:
    wheels = 4
    def __init__(self, color):
        self.color = color

a = Car("red")
b = Car("blue")
a.wheels = 6
Car.wheels = 5
```

1. What are `a.wheels`, `b.wheels`, and `Car.wheels` at the end?
2. Which of them is found in the class, and which in an object?

---

## The fix: create it in `__init__`

```python
class Cart:
    def __init__(self):
        self.items = []           # a NEW list for each object

    def add(self, thing):
        self.items.append(thing)

a = Cart()
b = Cart()
a.add("pen")
print(b.items)                    # []
```

Each call to `Cart()` runs `__init__`, which makes a separate list and stores it on that object.

**Rule of thumb:** use class variables for values shared by everyone (a constant like `wheels = 4`). Put anything that belongs to one object, especially lists and dicts, in `__init__` on `self`.

### Check

```python
class Cart:
    items = []
    def add(self, thing):
        self.items.append(thing)

a = Cart()
b = Cart()
a.add("pen")
b.add("book")
```

1. What are `a.items` and `b.items` at the end?
2. How many list objects exist in total?

---

## Printing an object

Try printing a `Point`:

```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

p = Point(3, 4)
print(p)
```

You get something like `<__main__.Point object at 0x7f...>`: the class name plus the object's address in memory (the number `id()` shows). Python doesn't know how you'd like a `Point` displayed.

You teach it by defining a **dunder method** (short for "double underscore", also called a magic method). Python calls these automatically in certain situations, just like `__init__` is called when you write `Point(3, 4)`:

```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __str__(self):
        return f"({self.x}, {self.y})"

p = Point(3, 4)
print(p)             # (3, 4)
```

- `print(p)` secretly calls `p.__str__()`, and prints whatever string it returns.
- `__str__` receives the object as `self`, like every method.
- The `f"..."` string puts the values of `self.x` and `self.y` into the text.
- `__str__` does not change the object's memory. It's a method stored once in the class, shared by every `Point`.

### Why the plain version shows `<__main__.Point object at 0x7f...>`

`print(p)` doesn't know how to display your object by itself. It calls `p.__str__()`, and Python looks that method up the same way it looked up `wheels`: object first, then class. And as with `__init__`, every class gets a `__str__` for free, from the class `object`.

| Your class | Which `__str__` is found | What it prints |
|---|---|---|
| No `__str__` written | the free one inherited from `object` | `<__main__.Point object at 0x7f...>` |
| Your own `__str__` written | yours, found in `Point` first | `(3, 4)` |

When you write your own `__str__` in `Point`, Python finds yours first and never reaches the one in `object`. This is overriding.

### Check

```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __str__(self):
        return f"({self.x}, {self.y})"

a = Point(1, 2)
b = Point(5, 6)
print(b)
```

1. What does the last line print?
2. Which method runs when `print(b)` executes, and what is `self` inside it?

---

## Doubt: What does it mean, "Object first, then class"?

There are two places where things are stored:

- **In the object:** the fields, `x` and `y`. Each object has its own.
- **In the class:** the methods, `move` and `__str__`. They're stored once and shared by every object. (Same as C++, where methods weren't inside each object.)

```
p (the object)         Point (the class)
┌──────────┐          ┌─────────────────┐
│ x ──► 3  │          │ __init__        │
│ y ──► 4  │          │ move            │
└──────────┘          │ __str__         │
                      └─────────────────┘
```

So when you write `p.x`, or `p.move(5)`, Python has to find the thing after the dot:

- `p.x`: look in `p`. Found there.
- `p.move`: look in `p`. Not there, since `p` only holds `x` and `y`. So look in the class `Point`. Found there.

---

## Making `==` work for your own class

Remember `==` vs `is` from Lesson 7? Try it with `Point`:

```python
a = Point(1, 2)
b = Point(1, 2)
print(a == b)       # False
print(a is b)       # False
```

You might expect `a == b` to be `True`, since both have the same contents. But `==` is also a dunder: Python secretly runs `a.__eq__(b)`. Just as with `__str__`, if you don't write one, Python uses a free default. That default only asks "is it the very same object?", which is `is`. These are two separate objects, so it says `False`.

You can teach Python what "equal" means for your class:

```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __eq__(self, other):
        return self.x == other.x and self.y == other.y

a = Point(1, 2)
b = Point(1, 2)
print(a == b)       # True
print(a is b)       # False
```

- In `a == b`, Python runs `a.__eq__(b)`. So `self` is `a` and `other` is `b`.
- The method compares the fields and returns `True` or `False`.
- `a is b` is still `False`. They're still two separate objects, and `is` can't be overridden.
- This is the same split as Java: `==` on objects asks "same object?" (Python's `is`), and `.equals()` asks "same contents?" (Python's `==`, once you write `__eq__`).
- Nothing changes in memory. `__eq__` is one method stored in the class, shared by all points.

### Check

```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __eq__(self, other):
        return self.x == other.x and self.y == other.y

a = Point(1, 2)
b = Point(1, 2)
c = a
```

1. What are `a == b`, `a is b`, and `a is c`?
2. In `a == b`, what are `self` and `other` inside `__eq__`?

---

## Making `+` work for your class

Try adding two points:

```python
a = Point(1, 2)
b = Point(10, 20)
c = a + b          # ✗ TypeError: Python doesn't know how to add two Points
```

Python runs `a.__add__(b)`, and `Point` has no `__add__`. Unlike `__str__` and `__eq__`, there's no free default for it. We can define one:

```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __add__(self, other):
        return Point(self.x + other.x, self.y + other.y)

    def __str__(self):
        return f"({self.x}, {self.y})"

a = Point(1, 2)
b = Point(10, 20)
c = a + b
print(c)           # (11, 22)
```

In `a + b`, `self` is `a` and `other` is `b`. The method builds a brand-new `Point` from the sums and returns it, so `c` points to a third object. `a` and `b` are untouched.

### Check

```python
a = Point(1, 2)
b = Point(3, 4)
c = a + b
```

1. How many `Point` objects exist afterwards, and what is `c`?
2. Inside `__add__`, what are `self` and `other`?

---

## The destructor, `__del__`

C++ had `~Point()`, which ran when an object died. Java had no destructor. Python has one, `__del__`, and it connects to the reference counting from Lesson 7: it runs at the moment the object's count hits 0.

```python
class Temp:
    def __init__(self, name):
        self.name = name
        print(self.name, "born")

    def __del__(self):
        print(self.name, "dies")

a = Temp("A")
b = a
del a
print("after del a")
del b
print("after del b")
```

I ran this on real Python 3.12, and the output was:

```
A born
after del a
A dies
after del b
```

Trace it:

1. `Temp("A")` creates the object, and `a` points to it (count = 1). `__init__` prints "A born".
2. `b = a` makes count = 2.
3. `del a` makes count = 1. The object is still alive, so nothing prints.
4. `print("after del a")` runs.
5. `del b` makes count = 0. Python calls `__del__` right then, printing "A dies", and frees the object.

### How it compares

| | C++ `~Point()` | Java | Python `__del__` |
|---|---|---|---|
| Exists? | yes | no | yes |
| Runs when | the object goes out of scope or you `delete` it | no fixed time | the count hits 0 (usually immediately) |

`__del__` runs only when an object's reference count hits 0.

---

## Class methods and static methods

So far every method took `self`, so it worked on one object. Sometimes you want a method that works on the class as a whole, like counting how many objects exist. Python has two kinds, and you write each one with a line above it called a **decorator** (`@...`).

```python
class Car:
    count = 0                       # class variable, shared by all cars

    def __init__(self, color):
        self.color = color
        Car.count += 1              # one more car exists

    @classmethod
    def how_many(cls):
        return cls.count

    @staticmethod
    def honk_sound():
        return "Beep"

a = Car("red")
b = Car("blue")
print(Car.how_many())               # 2
print(Car.honk_sound())             # Beep
```

I ran this on Python 3.12 and it printed `2` then `Beep`.

| Kind | First parameter | Gets | Can reach |
|---|---|---|---|
| normal method | `self` | the object it was called on | the object's fields and the class |
| `@classmethod` | `cls` | the class itself | the class variables |
| `@staticmethod` | (none) | nothing | neither, it's a plain function that sits in the class |

- In `Car.how_many()`, Python passes the class `Car` in as `cls`, just as it passes the object in as `self` for a normal method. So `cls.count` is `Car.count`.
- `honk_sound` has no `self` and no `cls`, because it needs neither. It lives in the class only to keep related code together.
- You can call both through the class (`Car.how_many()`) or through an object (`a.how_many()`).
- This is the same idea as `static` in C++ and Java. They are stored once, not in each object, so they add no bytes to your objects.
- Why `Car.count += 1` and not `self.count += 1`? From Step 3: assigning through `self` would create a new instance variable on that car, and the shared count would never change.

### Doubt killer

- Needs one object's data → normal method (`self`)
- Needs the class (class variables, or building objects) → `@classmethod`
- Needs neither → `@staticmethod`

Neither kind adds bytes to your objects. Both are stored once, in the class.

### Check

For each, say which kind fits best:

1. `def total_cars()` that returns how many cars exist, using `Car.count`
2. `def paint(self, new_color)` that changes one car's color
3. `def km_to_miles(km)` that converts a number and touches no car
