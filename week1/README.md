# 🚀 Python Bootcamp — Learning Log & Reference (Modules 1–4)

This repo tracks my progress through [Complete-Python-Bootcamp](https://github.com/krishnaik06/Complete-Python-Bootcamp) by Krish Naik.

I'm currently through **Module 4 of the course** (`1-Python Basics` → `2-Control Flow` → `3-Data Structures` → `4-Functions`). This file is both a log of what I've covered and a working cheat sheet I refer back to.

**Progress:** ✅ Module 1 · ✅ Module 2 · ✅ Module 3 · ✅ Module 4 · ⬜ Module 5 (Modules/Imports) onward

---

## 📚 Module 1 — Python Basics

**Covered:**
- Syntax, output (`print`), and taking input (`input()`)
- Variables and dynamic typing — no explicit type declarations
- Core types: `int`, `float`, `str`, `bool`
- Arithmetic operators (`+ - * / // %`) and logical operators (`and`, `or`, `not`)

```python
# Dynamic typing — no type declaration needed
course_name = "Engineering Python"   # str
student_count = 120                  # int
pass_rate = 95.5                     # float
is_active = True                     # bool

# f-strings for formatted output
name = input("Enter your name: ")
print(f"Hello, {name}! Welcome to the course.")

# Floor division vs modulo — easy to mix up
x, y = 10, 3
print(x // y)   # 3  → floor division, drops the remainder
print(x % y)    # 1  → modulo, keeps only the remainder
```

---

## 🔀 Module 2 — Control Flow

**Covered:**
- Conditionals: `if` / `elif` / `else`
- `for` loops (iterating ranges/sequences) and `while` loops
- Loop control: `break`, `continue`, `pass`

```python
score = 85
if score >= 90:
    print("Grade: A")
elif score >= 80:
    print("Grade: B")
else:
    print("Grade: C")

# for loop with continue
for i in range(1, 6):
    if i == 3:
        continue        # skips i=3, loop keeps going
    print(f"Iteration: {i}")

# while loop
count = 5
while count > 0:
    print(count)
    count -= 1
```

> **Note to self:** `continue` skips to the next iteration; `break` exits the loop entirely; `pass` does nothing — it's just a placeholder so a block isn't empty (useful when stubbing out code).

---

## 🏗️ Module 3 — Data Structures

**Covered:**
| Structure | Ordered? | Mutable? | Duplicates? | Best for |
|---|---|---|---|---|
| `list` | ✅ | ✅ | ✅ | Sequential, changeable data |
| `tuple` | ✅ | ❌ | ✅ | Fixed data, safe from accidental edits |
| `set` | ❌ | ✅ | ❌ | Uniqueness, fast membership checks, set math |
| `dict` | ✅ (3.7+) | ✅ | Keys: ❌ | Key–value lookups |

```python
# --- LISTS ---
tech_stack = ["Python", "Git", "SQL"]
tech_stack.append("Docker")

squares = [x**2 for x in range(1, 6)]     # [1, 4, 9, 16, 25]

# --- TUPLES ---
coordinates = (19.0760, 72.8777)
lat, lon = coordinates                     # unpacking

# --- SETS ---
raw_data = [1, 2, 2, 3, 3, 4]
unique_data = set(raw_data)                # {1, 2, 3, 4}

set_A = {1, 2, 3}
set_B = {3, 4, 5}
print(set_A & set_B)                       # {3} → intersection
print(set_A | set_B)                       # {1,2,3,4,5} → union

# --- DICTIONARIES ---
student_grades = {
    "physics": 88,
    "calculus": 92,
    "mechanics": 85
}
student_grades["chemistry"] = 90
print(student_grades["calculus"])          # 92
```

---

## ⚙️ Module 4 — Functions

**Covered:**
- Defining reusable code with `def`
- Parameters, default arguments, and return values
- Variable scope (local vs global)
- `*args` for flexible positional arguments
- `lambda` — anonymous one-line functions

```python
def greet_user(name, greeting="Hello"):
    return f"{greeting}, {name}!"

print(greet_user("Alex"))              # Hello, Alex!
print(greet_user("Sam", "Welcome"))    # Welcome, Sam!

# *args — collects extra positional args into a tuple
def calculate_sum(*numbers):
    return sum(numbers)

print(calculate_sum(10, 20, 30, 40))   # 100

# lambda — short, throwaway functions
square = lambda x: x * x
print(square(5))                       # 25
```

> **Note to self:** use `lambda` for quick, single-use logic (e.g. inside `sorted(key=...)` or `map()`). For anything with more than one line of logic, a regular `def` function is clearer.

---

## 🧠 How I'm applying this (not just memorizing syntax)

- **Automating small tasks:** loops + dicts/lists (Modules 2–3) to parse text files or clean up repetitive data.
- **Reusable problem-solving:** functions (Module 4) to wrap a formula once and run it across many inputs instead of recalculating by hand.
- **Prototyping before optimizing:** Python's lack of manual memory management makes it fast to test an algorithm's logic before implementing it in something lower-level.
- **Foundation for later modules:** the list/dict patterns from Module 3 are the same shapes used for JSON, API responses, and database records — so this isn't just "basics," it's the format everything downstream uses.

---

## 📎 Reference

Course: [Complete-Python-Bootcamp](https://github.com/krishnaik06/Complete-Python-Bootcamp) by Krish Naik
