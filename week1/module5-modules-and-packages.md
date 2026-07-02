# Module 5: Modules & Packages — My Learning Notes

*Notes from working through Module 5 of [Krish Naik's Complete Python Bootcamp](https://github.com/krishnaik06/Complete-Python-Bootcamp), written in my own words as part of my Python foundations before moving into OOP and advanced Python.*

---

## Importing Modules — The Basics

The first thing I really internalized here: Python's standard library is huge, and I barely need to write anything myself for common tasks — I just need to know *which module* to reach for.

```python
import math
print(math.sqrt(25))          # 5.0
print(math.sin(math.radians(90)))  # 1.0
```

**Aliasing** — importing a module under a shorter name — is something I'll use constantly (this is exactly the `import pandas as pd` pattern I've seen everywhere):

```python
import datetime as dt
print(dt.datetime.now())
```

**Importing specific functions** instead of the whole module keeps my code cleaner when I only need one or two things from a big module:

```python
from math import sqrt, pow
print(sqrt(16))   # 4.0
print(pow(2, 3))  # 8.0
```

**Handling missing modules gracefully** — instead of letting the program crash if a module isn't installed:

```python
try:
    import non_existent_module
except ImportError as e:
    print(f"Error importing module: {e}")
```

---

## Standard Library Modules Worth Knowing Cold

A few modules came up again and again, and I now think of them as my "default toolkit":

| Module | What I use it for |
|---|---|
| `os` | Creating/removing directories, listing files, interacting with the OS |
| `sys` | Checking Python version, reading command-line arguments |
| `math` | GCD, factorial, trig, sqrt, and other numeric operations |
| `datetime` | Current date/time, date arithmetic, day-of-week lookups |
| `random` | Random numbers, shuffling lists |

**`os` module** — file system operations:
```python
import os
os.mkdir('new_directory')
print(os.listdir('.'))
os.rmdir('new_directory')
```

**`datetime` module** — this one is genuinely useful beyond just "printing today's date". Date arithmetic with `timedelta` is something I know I'll use for logging and scheduling logic later:
```python
import datetime
today = datetime.date.today()
future_date = today + datetime.timedelta(days=100)
given_date = datetime.date(2022, 1, 1)
print(given_date.strftime('%A'))  # tells you the day of the week
```

**`random` module** — for generating and shuffling data, useful for testing:
```python
import random
random_numbers = [random.randint(1, 50) for _ in range(5)]
lst = [1, 2, 3, 4, 5]
random.shuffle(lst)
```

---

## Building My Own Package

This is the part that made "modules" click as a *structural* concept, not just an import statement.

**Basic package structure:**
```
mypackage/
    __init__.py
    module1.py     # def add(a, b): return a + b
    module2.py     # def multiply(a, b): return a * b
```

Without an `__init__.py` doing anything special, I have to reach into each module explicitly:
```python
from mypackage import module1, module2
print(module1.add(2, 3))
print(module2.multiply(2, 3))
```

**Using `__init__.py` to flatten imports** — this is the trick that makes a package feel clean to use from the outside:
```python
# inside __init__.py
from .module1 import add
from .module2 import multiply
```
Now anyone using my package can just do:
```python
from mypackage import add, multiply
```
No need to know the internal file structure at all. This is the same pattern I've seen in every well-designed library — the public interface is decided by `__init__.py`, not by how the files happen to be organized internally.

**Subpackages and relative imports** — once a package grows, I can nest packages inside packages:
```
mypackage/
    __init__.py
    module1.py
    subpackage/
        __init__.py
        module2.py
```
```python
# inside mypackage/__init__.py
from .module1 import add
from .subpackage.module2 import multiply
```
The `.` before the module name is a **relative import** — it means "look inside my own package," not the global namespace. This matters once a project has more than a couple of files.

**Handling errors when importing from my own package:**
```python
try:
    from mypackage import non_existent_function
except ImportError as e:
    print(f"Error importing function: {e}")
```

---

## What I'm Taking Away From This Module

- Importing isn't just "grab a library" — knowing *how* to import (whole module, aliased, specific functions) directly affects code readability
- The standard library already solves most of the small utility problems I'll run into — `os`, `sys`, `math`, `datetime`, `random` cover a lot of ground
- `__init__.py` is the real design decision point in a package — it's where I control what the outside world sees
- Wrapping imports in `try/except ImportError` is a small habit that makes code far more robust when dependencies are missing

This module felt like laying the groundwork for writing code that's organized like an actual project, not just a single script — which is exactly what I need heading into OOP.

---

*Personal study notes from Module 5 of the [Complete Python Bootcamp by Krish Naik](https://github.com/krishnaik06/Complete-Python-Bootcamp).*
