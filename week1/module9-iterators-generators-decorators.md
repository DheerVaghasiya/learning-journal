# Module 9: Iterators, Generators & Decorators — My Learning Notes

*Notes from working through Module 9 of [Krish Naik's Complete Python Bootcamp](https://github.com/krishnaik06/Complete-Python-Bootcamp) — this is where "advanced Python" really started to feel advanced.*

---

## Iterators — Understanding What a `for` Loop Actually Does

Before this module, I used `for item in something` without ever thinking about what makes that possible. Now I know: **any object with `__iter__` and `__next__` methods can be looped over.**

```python
class Countdown:
    def __init__(self, start):
        self.current = start

    def __iter__(self):
        return self

    def __next__(self):
        if self.current <= 0:
            raise StopIteration
        self.current -= 1
        return self.current

for number in Countdown(5):
    print(number)
```

**The two pieces, broken down:**
- `__iter__` returns the iterator object itself (usually just `self`)
- `__next__` returns the next value each time, and raises `StopIteration` when there's nothing left — that's literally the signal Python's `for` loop is listening for to know when to stop

Building my own version of the built-in `range()` function using this exact pattern (`MyRange`) is what made it click — `range`, lists, strings, dictionaries... they're all just objects that implement this same protocol underneath.

I also built an **infinite iterator** — one that never raises `StopIteration` on its own:
```python
class InfiniteCounter:
    def __init__(self, start):
        self.current = start
    def __iter__(self):
        return self
    def __next__(self):
        self.current += 1
        return self.current
```
This only works safely with `next()` calls or a bounded loop (`for _ in range(10)`) — a plain `for` loop over it would run forever, which is a good reminder that iterators put the "when to stop" logic entirely in my hands.

---

## Generators — The Lazy, Memory-Friendly Way to Iterate

This is the concept that actually reshaped how I write loops. A **generator function** looks like a normal function, except it uses `yield` instead of `return` — and that one keyword changes everything about how it runs.

```python
def fibonacci(n):
    a, b = 0, 1
    for _ in range(n):
        yield a
        a, b = b, a + b

for num in fibonacci(10):
    print(num)
```

**What actually happens under the hood:** calling `fibonacci(10)` doesn't run the function immediately — it returns a generator object. Each time I ask for the next value, the function runs *up to the `yield`*, hands back that value, and then pauses — remembering exactly where it left off. Next call, it resumes right after that `yield`.

**Why this matters for real work:** I don't have to build the entire sequence in memory upfront. For something like Fibonacci numbers, or reading massive files, this is the difference between a program that scales and one that runs out of memory.

**Generator expressions** are the lazy version of a list comprehension — same syntax, but with `()` instead of `[]`:
```python
squares = (x * x for x in range(1, 11))
```

**Chaining generators together** — this is the pattern that felt the most "advanced" to me, feeding one generator's output into another, like a pipeline:
```python
def even_numbers(limit):
    for i in range(limit + 1):
        if i % 2 == 0:
            yield i

def squares(numbers):
    for number in numbers:
        yield number * number

even_gen = even_numbers(20)
square_gen = squares(even_gen)   # generator built on top of another generator
```
Nothing actually computes until I iterate over `square_gen` — each value flows through the whole pipeline one at a time, lazily. I built a 3-stage version of this too (`integers → doubles → negatives`), and it's a genuinely elegant way to compose data transformations without ever holding the full dataset in memory.

**Stateful generators** — a generator can hold state across calls indefinitely, which is a clean way to build something like a counter:
```python
def counter(start):
    current = start
    while True:
        yield current
        current += 1

count = counter(0)
next(count)  # 0
next(count)  # 1
```

**Exception handling inside a generator** works the same as anywhere else — I can catch an error on one `yield` without killing the whole generator:
```python
def safe_divide(numbers, divisor):
    for number in numbers:
        try:
            yield number / divisor
        except ZeroDivisionError:
            yield "Error: Division by zero"
```

---

## Decorators — Wrapping a Function Without Changing Its Code

Decorators were genuinely confusing at first, until I realized the core idea is simple: **a decorator is a function that takes a function and returns a new function that wraps it.**

```python
import time

def time_it(func):
    def wrapper(*args, **kwargs):
        start_time = time.time()
        result = func(*args, **kwargs)
        end_time = time.time()
        print(f"Execution time: {end_time - start_time} seconds")
        return result
    return wrapper

@time_it
def factorial(n):
    return 1 if n == 0 else n * factorial(n - 1)
```

`@time_it` above `def factorial` is just shorthand for `factorial = time_it(factorial)`. Once I saw that equivalence, decorators stopped feeling like magic.

**Decorators that take their own arguments** need one extra layer of nesting — a function that returns the actual decorator:
```python
def repeat(n):
    def decorator(func):
        def wrapper(*args, **kwargs):
            for _ in range(n):
                func(*args, **kwargs)
        return wrapper
    return decorator

@repeat(3)
def print_message(message):
    print(message)
```

**Stacking multiple decorators** — they apply bottom-up, closest to the function first:
```python
def uppercase(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs).upper()
    return wrapper

def exclaim(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs) + "!"
    return wrapper

@uppercase
@exclaim
def greet(name):
    return f"Hello, {name}"

greet("Alice")  # "HELLO, ALICE!" — exclaim runs first, then uppercase wraps that result
```

**Decorators aren't just for functions — they can wrap classes too.** A `singleton` decorator is a great real-world example: it ensures only one instance of a class ever exists, which is useful for things like a shared database connection:
```python
def singleton(cls):
    instances = {}
    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]
    return get_instance

@singleton
class DatabaseConnection:
    def __init__(self):
        print("Database connection created")

db1 = DatabaseConnection()
db2 = DatabaseConnection()
db1 is db2  # True — same object both times
```

**A genuinely practical pattern:** a decorator that manages opening and closing a file, so the function using it doesn't have to think about file handling at all:
```python
def open_file(file_name, mode):
    def decorator(func):
        def wrapper(*args, **kwargs):
            with open(file_name, mode) as file:
                return func(file, *args, **kwargs)
        return wrapper
    return decorator

@open_file('sample.txt', 'w')
def write_to_file(file, text):
    file.write(text)
```
This is the same idea as the context manager class I built in the OOP module — just approached through a decorator instead of `__enter__`/`__exit__`.

---

## What I'm Taking Away From This Module

- Iterators are the actual mechanism behind every `for` loop I've ever written — `__iter__` + `__next__` + `StopIteration` is the whole protocol
- Generators solve the memory problem that regular functions can't — they produce values one at a time instead of building a full list upfront
- `yield` pausing and resuming a function's exact state is genuinely one of the more elegant things I've learned in Python so far
- A decorator is just `func = decorator(func)` in disguise — once that clicked, nested decorators and decorators-with-arguments stopped being confusing
- These three concepts (iterators, generators, decorators) aren't isolated tricks — they're the tools behind patterns I'll keep running into: lazy data pipelines, timing/logging wrappers, and singleton-style resource management

This module is where Python stopped feeling like "a scripting language" and started feeling like a language with real depth to it.

---

*Personal study notes from Module 9 of the [Complete Python Bootcamp by Krish Naik](https://github.com/krishnaik06/Complete-Python-Bootcamp).*
