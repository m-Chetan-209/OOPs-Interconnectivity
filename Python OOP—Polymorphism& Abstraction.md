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
