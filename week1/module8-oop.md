# Module 8: Object-Oriented Programming — My Learning Notes

*Notes from working through Module 8 of [Krish Naik's Complete Python Bootcamp](https://github.com/krishnaik06/Complete-Python-Bootcamp) — this covers classes & objects, inheritance, and the core OOP pillars (polymorphism, abstraction, encapsulation).*

---

## Classes and Objects — The Foundation

The mental shift here: a **class** is a blueprint, an **object** is an actual thing built from that blueprint. Everything else in OOP is built on top of that one idea.

```python
class Car:
    def __init__(self, make, model, year):
        self.make = make
        self.model = model
        self.year = year

    def start_engine(self):
        print("The engine has started.")
```

`__init__` is the constructor — it runs automatically the moment I create an object, and `self` is how each object refers to its own data.

---

## Encapsulation — Protecting an Object's Internal State

This was the concept that made OOP feel less academic and more practical. The idea: an object should control access to its own data, not just expose everything freely.

**Private attributes** (double underscore prefix) signal "don't touch this directly from outside the class":
```python
class BankAccount:
    def __init__(self, account_number, balance=0):
        self.__account_number = account_number
        self.__balance = balance

    def deposit(self, amount):
        self.__balance += amount

    def withdraw(self, amount):
        if amount > self.__balance:
            print("Insufficient balance!")
        else:
            self.__balance -= amount

    def check_balance(self):
        return self.__balance
```

**Properties** take this further — they let me expose an attribute like a normal variable (`account.balance`) while still running validation logic behind the scenes:
```python
class BankAccount:
    def __init__(self, account_number, balance=0):
        self.__balance = balance

    @property
    def balance(self):
        return self.__balance

    @balance.setter
    def balance(self, amount):
        if amount < 0:
            print("Balance cannot be negative!")
        else:
            self.__balance = amount
```

**Why this actually matters:** without this, anyone could do `account.balance = -9999` directly and silently corrupt the data. The setter gives me a single controlled gateway for every change to that value — this is the difference between an object that just *stores* data and one that *protects* its own rules.

---

## Inheritance — Reusing and Extending Behavior

Inheritance is how a class can build on top of another class instead of repeating its logic.

```python
class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age

class Employee(Person):
    def __init__(self, name, age, employee_id):
        super().__init__(name, age)   # reuse Person's setup logic
        self.employee_id = employee_id
```

`super()` is the key piece here — it lets the child class call the parent's version of a method (usually `__init__`) instead of duplicating that logic.

**Overriding a method** — the child class can redefine behavior it inherited:
```python
class Car(Vehicle):
    def start(self):
        print("Car starting...")
        super().start()   # still call the parent's version too, if I want
```

**Multiple inheritance** — a class can inherit from more than one parent:
```python
class Walker:
    def walk(self):
        print("Walking...")

class Runner:
    def run(self):
        print("Running...")

class Athlete(Walker, Runner):
    pass
```

**The diamond problem** is the one gotcha I had to actually sit with: if two parent classes both define the same method, which one wins?
```python
class A:
    def show(self):
        print("A's show method")

class B(A):
    def show(self):
        print("B's show method")

class C(A):
    def show(self):
        print("C's show method")

class D(B, C):
    pass

d = D()
d.show()  # "B's show method" — not C, not A
```
Python resolves this using **MRO (Method Resolution Order)** — it checks the parent classes left to right in the order they're listed (`D(B, C)` → checks `D`, then `B`, then `C`, then `A`). This is why the order I list parent classes in matters, not just which classes I pick.

---

## Polymorphism — Same Interface, Different Behavior

Polymorphism means different classes can be used interchangeably as long as they share the same method name — each one just does its own thing when called.

```python
class Shape:
    def area(self):
        pass

class Circle(Shape):
    def __init__(self, radius):
        self.radius = radius
    def area(self):
        return math.pi * self.radius ** 2

class Square(Shape):
    def __init__(self, side):
        self.side = side
    def area(self):
        return self.side ** 2

shapes = [Circle(5), Square(4)]
for shape in shapes:
    print(shape.area())  # works for both, no need to check the type
```

**Why this is powerful:** I can write a function like `describe_shape(shape)` that works on *any* shape, current or future, as long as it implements `.area()`. I don't need if/else chains checking "is this a Circle or a Square" — the object itself knows how to behave.

**Operator overloading** is polymorphism applied to built-in operators — I can define what `+` means for my own class:
```python
class Vector:
    def __init__(self, x, y):
        self.x = x
        self.y = y
    def __add__(self, other):
        return Vector(self.x + other.x, self.y + other.y)
    def __str__(self):
        return f"Vector({self.x}, {self.y})"

v3 = Vector(2, 3) + Vector(4, 5)  # Vector(6, 8)
```

---

## Abstraction — Forcing a Contract Without Providing the Implementation

An **abstract base class** defines *what* every subclass must implement, without saying *how*. This is enforced by Python, not just a naming convention — trying to instantiate an abstract class directly will raise an error.

```python
from abc import ABC, abstractmethod

class Vehicle(ABC):
    @abstractmethod
    def start_engine(self):
        pass

    def fuel_type(self):     # concrete method — shared by all subclasses
        return "Generic Fuel"

class Car(Vehicle):
    def start_engine(self):
        print("Car engine started")
    def fuel_type(self):
        return "Petrol"
```

**Why I'd actually use this:** it guarantees that every subclass of `Vehicle` *must* implement `start_engine()` — if I forget to, Python won't let me create the object at all. It's a way of designing a contract upfront, which matters a lot once multiple people (or my future self) are extending the same base class.

Abstract classes can also have **abstract properties**, combining `@property` and `@abstractmethod`:
```python
class Appliance(ABC):
    @property
    @abstractmethod
    def power(self):
        pass
```

---

## Extra Object Patterns Worth Remembering

**Class variables vs instance variables** — shared state across every object of a class, tracked using `@classmethod`:
```python
class Counter:
    count = 0
    def __init__(self):
        Counter.count += 1

    @classmethod
    def get_count(cls):
        return cls.count
```

**Static methods** — a method that belongs to the class conceptually but doesn't need `self` or `cls` at all:
```python
class MathOperations:
    @staticmethod
    def sqrt(x):
        return math.sqrt(x)

MathOperations.sqrt(16)   # called without ever creating an object
```

**Composition** — instead of inheriting, a class can simply *hold* another object as an attribute. I now think of this as "has-a" versus inheritance's "is-a":
```python
class Person:
    def __init__(self, name, age, address):
        self.name = name
        self.age = age
        self.address = address   # a Person "has an" Address, not "is an" Address
```

**Context managers** — implementing `__enter__` and `__exit__` lets my own class work with the `with` statement, just like `open()` does:
```python
class FileManager:
    def __enter__(self):
        self.file = open(self.filename, self.mode)
        return self.file
    def __exit__(self, exc_type, exc_value, traceback):
        self.file.close()
```

**Method chaining** — having each method return `self` so calls can be strung together fluently:
```python
class Calculator:
    def add(self, amount):
        self.value += amount
        return self

calc = Calculator()
calc.add(10).subtract(3).multiply(2)   # chained calls
```

---

## What I'm Taking Away From This Module

- Encapsulation is about controlling *how* data changes, not just hiding it — properties are the real tool here, not just double underscores
- `super()` is how inheritance avoids duplicating logic — I should reach for it by default in a subclass `__init__`
- MRO matters the moment I use multiple inheritance — the order of parent classes isn't cosmetic
- Polymorphism is what lets me write code against a shared interface instead of checking types everywhere — this is genuinely how "clean" OOP code avoids long if/else chains
- Abstract base classes are how I enforce a contract at the language level, not just through documentation or convention
- OOP isn't a separate topic from what I learned in file handling and exceptions — context managers and custom exceptions are both things I can build myself using these same class mechanics

---

*Personal study notes from Module 8 of the [Complete Python Bootcamp by Krish Naik](https://github.com/krishnaik06/Complete-Python-Bootcamp).*
