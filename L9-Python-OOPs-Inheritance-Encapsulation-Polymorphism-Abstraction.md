# Python OOP: The Four Pillars
---

## Inheritance in Python

```python
class Vehicle:
    def __init__(self, wheels):
        self.wheels = wheels

    def honk(self):
        return "..."

class Car(Vehicle):                 # Car inherits from Vehicle
    def __init__(self, wheels, doors):
        super().__init__(wheels)    # run Vehicle's __init__ first
        self.doors = doors

    def honk(self):
        return "Beep"

c = Car(4, 2)
print(c.wheels, c.doors, c.honk())  # 4 2 Beep
```

| | C++ | Java | Python |
|---|---|---|---|
| Inherit | `class Car : public Vehicle` | `class Car extends Vehicle` | `class Car(Vehicle):` |
| Call the base constructor | `: Vehicle(w)` | `super(w);` | `super().__init__(wheels)` |

There is **one object**, not two. `Vehicle.__init__` fills `wheels` onto the same object, then `Car.__init__` adds `doors`.

### Which `__init__` runs first?

`Car(4, 2)` starts by running `Car.__init__`. Its first line, `super().__init__(wheels)`, runs `Vehicle.__init__` to completion, and then the rest of `Car.__init__` continues.

### How `c.honk()` finds its method

1. In the **object** `c`: no `honk` there.
2. In the **class** `Car`: yes. Stop, and it returns "Beep".

If `Car` had no `honk`, Python would continue to the parent class `Vehicle` and find `"..."` there. This happens because of the `(Vehicle)` in `class Car(Vehicle):`. It has nothing to do with `__init__`.

### You can sleep on the parent's constructor if you want

In C++ and Java, the base constructor always runs. In Python, `super().__init__(...)` is a normal call that **you** write.

```python
class Car(Vehicle):
    def __init__(self, wheels, doors):
        self.doors = doors          # forgot super().__init__(...)

c = Car(4, 2)
print(c.doors)                      # 2
print(c.wheels)                     # ✗ AttributeError: 'Car' object has no attribute 'wheels'
print(c.honk())                     # "..." still works
```

Methods are inherited through the class chain, but fields exist only if some `__init__` created them. The fix is to run the parent's `__init__` with `super().__init__(wheels)`. It's convention to put it first, but Python doesn't force that order.

---

## What `super()` is for

`super()` **runs the parent's version of a method**. There are no separate parent variables: the fields all live on the one object.

```python
class Car(Vehicle):
    def honk(self):
        return super().honk() + " Beep"   # run Vehicle's honk, then add to it

c = Car(4, 2)
print(c.honk())                           # ... Beep
```

#### Goal of `super`
1. Run a parent method the child **doesn't override**. Means the child doesn't have that method so you run the Parent's
2. Run the parent's version of a method the child **overrides**. If the child has that method, you run the parent's as well as the child's.

```python
class Vehicle:
    def __init__(self, wheels):
        self.wheels = wheels
    def describe(self):
        return f"{self.wheels} wheels"

class Car(Vehicle):
    def __init__(self, wheels, doors):
        super().__init__(wheels)
        self.doors = doors
    def describe(self):
        return super().describe() + f", {self.doors} doors"

c = Car(4, 2)
print(c.describe())                       # 4 wheels, 2 doors
```

`Car.describe` runs first, and inside it `Vehicle.describe` runs and returns `"4 wheels"`, then `Car` adds `", 2 doors"`.

---

## Types of inheritance

```python
class Vehicle:
    def honk(self): return "..."

class Car(Vehicle):                  # single: one parent
    pass

class SportsCar(Car):                # multilevel: a chain of parents
    pass

class Truck(Vehicle):                # hierarchical: many children, one parent
    def honk(self): return "HONK"
```

```
Single:        Vehicle ◄── Car
Multilevel:    Vehicle ◄── Car ◄── SportsCar
Hierarchical:  Vehicle ◄── Car
               Vehicle ◄── Truck
```

| Type | Shape |
|---|---|
| **Single** | one child, one parent |
| **Multilevel** | a chain: child → parent → grandparent |
| **Hierarchical** | several children share one parent |
| **Multiple** | one child with several parents |
| **Hybrid** | a mix of the above, such as the diamond |

### The MRO: the chain Python searches

```python
print([c.__name__ for c in SportsCar.__mro__])
# ['SportsCar', 'Car', 'Vehicle', 'object']
```

For `SportsCar().honk()`, Python checks the **object**, then `SportsCar`, `Car`, `Vehicle`, and finally `object`, and **stops at the first match**. Every chain ends at `object`, which is why every class gets a free `__str__`.

- `SportsCar().honk()` and `Car().honk()` return `"..."` (found in `Vehicle`).
- `Truck().honk()` returns `"HONK"` (found in `Truck` first).
- If `Car` also defined `honk` returning `"Beep"`, `SportsCar().honk()` would return `"Beep"`.

---

## Multiple inheritance

```python
class Car:
    def move(self): return "drive"

class Boat:
    def move(self): return "sail"
    def anchor(self): return "anchor down"

class Amphibian(Car, Boat):         # two parents
    pass

a = Amphibian()
print(a.move())                     # drive
print(a.anchor())                   # anchor down
print([c.__name__ for c in Amphibian.__mro__])
# ['Amphibian', 'Car', 'Boat', 'object']
```

Python searches left to right, **in the order you listed the parents**. Both parents define `move`, but `Car` is listed first, so `"drive"` wins. With `class Amphibian2(Boat, Car)`, `move()` returns `"sail"`.

It's still **one object**: the fields set by both parents' `__init__` land on the same object.

| | Multiple inheritance |
|---|---|
| **C++** | allowed. The object contains both base parts side by side in memory |
| **Java** | not allowed for classes (only for interfaces) |
| **Python** | allowed. The MRO decides, and the fields all end up on the one object |

### Check

```python
class A:
    def hello(self): return "A"

class B:
    def hello(self): return "B"
    def bye(self): return "bye from B"

class C(B, A):
    pass

c = C()
```

`c.hello()` is `"B"`, `c.bye()` is `"bye from B"`, and the MRO is `C`, `B`, `A`, `object`.

---

## The diamond (hybrid inheritance)

```python
class Vehicle:
    def info(self): return "vehicle"

class Car(Vehicle):
    pass

class Boat(Vehicle):
    def info(self): return "boat"

class Amphibian(Car, Boat):
    pass
```

```
        Vehicle
        ▲     ▲
      Car     Boat
        ▲     ▲
       Amphibian
```

```python
print([c.__name__ for c in Amphibian.__mro__])
# ['Amphibian', 'Car', 'Boat', 'Vehicle', 'object']

print(Amphibian().info())
# boat
```

`Vehicle` sits **after both** `Car` and `Boat`, because **a class always comes before its parents**. Trace `Amphibian().info()`: the object has none, `Amphibian` has none, `Car` has none of its own, and `Boat` overrides it, so the answer is `"boat"`. Python never reaches `Vehicle`'s version.

In C++, the same diamond contains **two separate `Vehicle` parts** unless you use `virtual` inheritance. In Python, each class appears **once** in the chain.

### Check

```python
class A:
    def who(self): return "A"

class B(A):
    pass

class C(A):
    def who(self): return "C"

class D(B, C):
    pass
```

The MRO of `D` is `D`, `B`, `C`, `A`, `object`, and `D().who()` returns `"C"`.

---

## Lookup order: the object comes first

The full lookup order for `x.name` is:

1. **The object itself**
2. **Each class in the MRO**, left to right, stopping at the first match

```python
a = Amphibian()
b = Amphibian()

a.info = lambda: "mine"     # gives object a its own "info"

print(a.info())             # mine
print(b.info())             # boat
```

`a` finds `info` in its own storage and stops. `b` has none, so it goes through the class chain and gets `"boat"`. After `del a.info`, `a.info()` returns `"boat"` again. A function stored on an object is not passed `self`, which is why the lambda took no arguments.

```python
class A:
    def hi(self): return "class A"

x = A()
y = A()
x.hi = lambda: "own"
```

`x.hi()` returns `"own"` (found in the object), and `y.hi()` returns `"class A"` (found in the class).

### Printing a class

A class is itself an object.

| Expression | What it printed | Meaning |
|---|---|---|
| `print(A)` | `<class '__main__.A'>` | the class object itself |
| `print(x)` | `<__main__.A object at 0x7f...>` | an **instance** of `A` |
| `print(type(x))` | `<class '__main__.A'>` | the class that `x` belongs to |
| `print(type(A))` | `<class 'type'>` | the class of `A` is `type` |
| `print(A.hi)` | `<function A.hi at 0x7f...>` | the method stored in the class |

`<class '...A'>` means the blueprint. `<... A object at 0x...>` means one object built from it. Also, `x.__class__ is A` is `True`.

---

## Encapsulation in Python

C++ and Java have compiler-checked `private`. **Python has no real `private`.** It has naming conventions.

```python
class Account:
    def __init__(self, balance):
        self.balance = balance      # public
        self._pin = 1234            # one underscore: "please don't touch"
        self.__secret = "x"         # two underscores: name-mangled

    def reveal(self):
        return self.__secret
```

```python
a = Account(100)
print(a.balance)       # 100
print(a._pin)          # 1234     ← works, just frowned upon
print(a.__secret)      # ✗ AttributeError: no attribute '__secret'
print(a.reveal())      # x        ← works, inside the class
```

| Name | Meaning | Enforced? |
|---|---|---|
| `balance` | public, use freely | n/a |
| `_pin` | "internal, please don't use from outside" | **No.** Python lets you read and change it |
| `__secret` | internal, and Python **renames** it | partly: the plain name stops working, but the renamed one still does |

### What "outside" means

- **Inside** = code written in the class body (`__init__`, `reveal`).
- **Outside** = any other code, such as the bottom of your file.

The underscore in `_pin` is a message from the class's author: "`_pin` is my own internal business. Don't depend on it from outside." Python doesn't stop you.

```python
a = Account()
a._pin = 0          # works, no error
```

| | C++ `private` | Python `_pin` |
|---|---|---|
| Reading or changing it from outside | **error** | **works** |
| What it is | a lock | a sign on an unlocked door |

### What the double underscore does

Python **renames the field** inside the class to `_Account__secret` (underscore, class name, then the original name). That's called **name mangling**.

```python
print(a.__dict__)
# {'balance': 100, '_pin': 1234, '_Account__secret': 'x'}

print(a._Account__secret)    # x   ← one single name, no dot in the middle
```

`a.__secret` fails because no field has that exact name. Inside the class, `self.__secret` is rewritten to `self._Account__secret`, which is why `reveal()` works.

In memory, all three kinds of field are ordinary fields. The differences are only in the names.

---

## `@property`: a controlled door

Python does what getters and setters do in C++ and Java with `@property`, and outside code still writes plain `a.balance`:

```python
class Account:
    def __init__(self, balance):
        self._balance = balance

    @property
    def balance(self):                  # runs when you READ a.balance
        return self._balance

    @balance.setter
    def balance(self, value):           # runs when you WRITE a.balance = ...
        if value < 0:
            raise ValueError("balance cannot be negative")
        self._balance = value
```

```python
a = Account(100)
print(a.balance)       # 100
a.balance = 50         # runs the setter, allowed
a.balance = -5         # ✗ ValueError: balance cannot be negative
```

### What really stores the number

A version that prints when each method runs:

```python
class Account:
    def __init__(self, balance):
        self._balance = balance

    @property
    def balance(self):
        print("getter runs")
        return self._balance

    @balance.setter
    def balance(self, value):
        print("setter runs with", value)
        self._balance = value

a = Account(100)
print("--- write")
a.balance = 50
print("--- read")
print(a.balance)
print(a.__dict__)
```

Output:

```
--- write
setter runs with 50
--- read
getter runs
50
{'_balance': 50}
```

1. `a.balance = 50`: Python sees `balance` is a **property** in the class, so it **calls the setter** instead of storing anything.
2. Inside the setter, `self._balance = value` stores the number in the object. **This line is the one that stores it.**
3. The object holds only `_balance`, and there is no field called `balance` at all.

```
a (object)               Account (class)
┌────────────────┐      ┌─────────────────────────┐
│ _balance ──► 50│      │ balance → getter, setter │
└────────────────┘      └─────────────────────────┘
```

This differs from `a.wheels = 6` in Lesson 8, which created a new field on `a`. Here the class's property intercepts the assignment first.

### Check

```python
a = Account(100)     # prints nothing: __init__ writes self._balance directly
a.balance = 70       # prints: setter runs with 70
x = a.balance        # prints: getter runs
```

Afterwards the object stores `_balance: 70`, and `x` is 70.

---

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

Python doesn't check *what type* the object is, only whether the **method exists**:

```python
class Rock:
    pass

Rock().move()      # ✗ AttributeError: 'Rock' object has no attribute 'move'
```

The error appears **at run time, only when that line runs**. In C++ and Java, the compiler would have refused the program earlier. This style is called **duck typing**: "if it walks like a duck and quacks like a duck, it's a duck."

| | C++ | Java | Python |
|---|---|---|---|
| Needs a shared base class or interface | yes | yes | **no** |
| Needs `virtual` | yes | no (default) | no |
| Wrong type found | compile error | compile error | error **at run time** |

Nothing new is stored in the objects. Python looks up `thing.move` at run time with the same rule as before: the object first, then its class chain. That lookup is the polymorphism, so the cost is a lookup and not a hidden `vptr`.

### Check

```python
for thing in [Car(), Rock(), Boat()]:
    print(thing.move())
```

Only `drive` prints. `Rock()` has no `move`, so Python raises an `AttributeError` and the **program stops**, and `Boat()` is never reached.

---

## Abstraction with `abc`

In C++, `virtual void speak() = 0;` made a class abstract. Python has the same thing in the **`abc` module** (abstract base classes):

```python
from abc import ABC, abstractmethod

class Vehicle(ABC):                 # inherit from ABC to make it abstract
    @abstractmethod
    def honk(self):                 # no body needed
        pass

class Car(Vehicle):
    def honk(self): return "Beep"   # Car supplies it

class Cart(Vehicle):
    pass                            # forgot honk
```

```python
Vehicle()      # ✗ TypeError: Can't instantiate abstract class Vehicle without an implementation for abstract method 'honk'
Cart()         # ✗ TypeError: same, because Cart is still abstract
Car().honk()   # Beep
```

| | C++ | Python |
|---|---|---|
| How you write it | `virtual void honk() = 0;` | `@abstractmethod` inside a class that inherits `ABC` |
| Building the abstract class | compile error | `TypeError` at run time, **when you create the object** |
| Child forgets the method | compile error | `TypeError` at run time, when you create the child |

The child must finish the job, and forgetting is caught loudly. The difference is that Python catches it only when the line `Cart()` runs, not before the program starts.

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
