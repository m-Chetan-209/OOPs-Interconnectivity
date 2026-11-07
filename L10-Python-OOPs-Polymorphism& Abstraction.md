# Python OOP- Polymorphism & Abstraction

## Polymorphism and duck typing

In C++ you needed `virtual` and a shared base class so that `p->speak()` could run different code for different objects. In Java, every object needed a shared parent or interface.

**Python needs neither.** A call only needs the object to have a method **with that name**:

```python
class Car:
    def move(self): return "drive"

class Boat:
    def move(self): return "sail"

for thing in [Car(), Boat()]:
    print(thing.move())
```

Output:

```
drive
sail
```

`Car` and `Boat` have no common parent (apart from `object`), and there's no `virtual`. The same line, `thing.move()`, does different things depending on the real object. That is **polymorphism**.

Python doesn't check *what type* the object is, only whether the **method exists**, this is **Duck Typing**: if it walks like a duck and quacks like a duck, it's a duck."

```python
class Rock:
    pass

Rock().move()      # ✗ AttributeError: 'Rock' object has no attribute 'move'
```


| | C++ | Java | Python |
|---|---|---|---|
| Needs a shared base class or interface | yes | yes | **no** |
| Needs `virtual` | yes | no (default) | no |
| Wrong type found | compile error | compile error | error **at run time** |

Nothing new is stored in the objects. Python looks up `thing.move` at run time with the same rule as before: the object first, then its class chain. That lookup is the polymorphism, so the cost is a lookup and not a hidden `vptr`.


### Check

- In C++, what happens if you forget `virtual` and call `honk()` through a `Vehicle *` that points to a `Car`?
- In Python and Java, why isn't any keyword needed?

### Check

```python
for thing in [Car(), Rock(), Boat()]:
    print(thing.move())
```

Only `drive` prints. `Rock()` has no `move`, so Python raises an `AttributeError` and the **program stops**, and `Boat()` is never reached.

---

## Abstraction in Python 
### The Problem

```python
class Vehicle:
    def honk(self):
        return "..."

class Car(Vehicle):
    def honk(self):
        return "Beep"

class Cart(Vehicle):
    pass                    # the programmer forgot to write honk

print(Cart().honk())        # ...
```

`Cart` was supposed to have its own sound, but the programmer forgot. Python didn't complain. It quietly used the parent's `...`.

The mistake is hidden. Nobody knows something went wrong until later, if at all.

### What abstraction is for

Abstraction is a way to say:

> "Every vehicle must write its own `honk`. If someone forgets, stop and tell them."

#### Mark the method as "must be written by every child"

```python
from abc import ABC, abstractmethod

class Vehicle(ABC):
    @abstractmethod
    def honk(self):
        pass
```

Only three things are new here:

- `from abc import ABC, abstractmethod` just brings two ready-made tools into your file, like `import sys` did earlier.
- `(ABC)` after `Vehicle` is a label that says "this class is a rulebook for other classes".
- `@abstractmethod` above `honk` says "every child must write its own `honk`". The body is just `pass`, which means "nothing here".

### Piece 2: a child that follows the rule

```python
class Car(Vehicle):
    def honk(self):
        return "Beep"

print(Car().honk())        # Beep
```

`Car` wrote `honk`, so everything works as before.

### Piece 3: a child that forgets

```python
class Cart(Vehicle):
    pass                    # forgot honk

Cart()                      # ✗ TypeError
```

Now Python does warn us, at the moment we try to create the object, and says exactly what's missing. That's the safety net we wanted, so the forgotten `honk` can't hide any more.

### Check

1. **Which of `Car()` and `Cart()` raises the error?**
   `Cart()`. `Car` wrote `honk`, but `Cart` did not.
2. **What does the error tell us?**
   It is a `TypeError` saying Python can't create `Cart` because it has not written the abstract method `honk`, so the missing method is named right away.

### One more piece: the rulebook itself can't be created

```python
Vehicle()          # ✗ TypeError: Can't instantiate abstract class Vehicle ...
```

`Vehicle` has an unfinished method (`honk` has no real body), so Python refuses to build an object from it. It only exists so that other classes can inherit from it and finish the job. That matches what you learned in C++ with `= 0`.

## An abstract class can also hold normal methods

It can mix must-write methods with ones that are already finished:

```python
from abc import ABC, abstractmethod

class Vehicle(ABC):
    def __init__(self, wheels):
        self.wheels = wheels

    def describe(self):                  # a normal method, already complete
        return f"{self.wheels} wheels"

    @abstractmethod
    def honk(self):                      # every child must write this one
        pass

class Car(Vehicle):
    def honk(self):
        return "Beep"

c = Car(4)
print(c.describe())    # 4 wheels  (inherited from Vehicle)
print(c.honk())        # Beep      (Car's own)
```

- `describe` and `__init__` are inherited as usual.
- `honk` has to be written by every child, or the child can't be created.

## Python vs C++

|                              | C++                          | Python                                          |
|------------------------------|------------------------------|-------------------------------------------------|
| How you mark it              | `virtual string honk() = 0;` | `@abstractmethod` in a class that inherits `ABC` |
| Creating the abstract class  | compile error                | `TypeError` when you create it                  |
| A child forgets the method   | compile error                | `TypeError` when you create the child           |

### Check

Using the last code above:

1. **What does `c.describe()` print, and where is `describe` defined?**
   It prints `4 wheels`. `describe` is defined in `Vehicle` and `Car` inherits it.
2. **What would happen if you wrote `class Bus(Vehicle): pass` and then `Bus(6)`?**
   `TypeError`. `Bus` did not write the abstract method `honk`, so Python refuses to create the object.

### Check

```python
class Vehicle(ABC):
    @abstractmethod
    def honk(self): pass

    @abstractmethod
    def start(self): pass

class Car(Vehicle):
    def honk(self): return "Beep"
```

1. **Can you create `Car()`? Why or why not?**
   No. `Car` only wrote `honk` and left out `start`, so one abstract method is still unfinished and Python raises a `TypeError`.
2. **What single addition would fix it?**
   Add a `start` method to `Car`:

   ```python
   def start(self): return "Vroom"
   ```

### Duck typing vs `abc`

- **Duck typing:** no promises. Any object with a `move()` works, and a missing method fails only when it's called.
- **`abc`:** a promise. "Every `Vehicle` **must** have `honk`", enforced when the object is created.

The class `Vehicle` takes no extra memory per object. It's one class object, like any other.

### Check

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self):
        pass

class Square(Shape):
    def __init__(self, s):
        self.s = s
    def area(self):
        return self.s * self.s

class Blob(Shape):
    pass
```

`Shape()` and `Blob()` raise a `TypeError` (`Shape` is abstract, and `Blob` never supplied `area`). `Square(3).area()` is 9.

---

## COMMON DOUBT:-

| What the name is | How you use it | Why |
|---|---|---|
| A normal method | `a.get_plain()`, with `()` | the parentheses mean "run it now" |
| A property (`@property`) | `a.balance`, no `()` | Python runs the getter for you |
| A plain field | `a.color`, no `()` | it's just a value |

### Check

```python
class Car:
    def __init__(self):
        self._speed = 60

    @property
    def speed(self):
        return self._speed

    def honk(self):
        return "Beep"
```

For each, say whether it works or raises an error:

```python
print(Car().speed)
print(Car().speed())
print(Car().honk())
print(Car().honk)
```



## The four pillars compared

| Pillar | C++ | Java | Python |
|---|---|---|---|
| **Inheritance** | `class Car : public Vehicle` | `class Car extends Vehicle` | `class Car(Vehicle):`, and multiple parents allowed (the MRO decides) |
| **Encapsulation** | `private`, enforced by the compiler | `private`, enforced by the compiler | `_name` is a note and `__name` is a rename (neither is a lock), plus `@property` for checked access |
| **Polymorphism** | needs `virtual` and a shared base | every method is virtual, needs a shared parent or interface | **duck typing**: any object with the right method name works |
| **Abstraction** | `= 0` | `abstract` classes and interfaces | `ABC` and `@abstractmethod` |

## Summary

1. `super()` runs the **parent's version of a method**. Fields live on the one object, so there's nothing to fetch from the parent.
2. Methods come from the class chain, but fields exist only if some `__init__` created them. That's why forgetting `super().__init__(...)` loses the parent's fields.
3. The lookup order for `x.name` is **the object first, then each class in the MRO**, stopping at the first match.
4. A property's setter is what stores the value (`self._balance = value`). `a.balance = 50` only triggers it.
5. Errors in Python appear **at run time**, not at compile time: a missing method during duck typing, or creating an abstract class.
