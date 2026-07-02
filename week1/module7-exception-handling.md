# Module 7: Exception Handling — My Learning Notes

*Notes from working through Module 7 of [Krish Naik's Complete Python Bootcamp](https://github.com/krishnaik06/Complete-Python-Bootcamp).*

---

## The Core Pattern: try / except / finally

The mental model that stuck with me: **`try` is "attempt this," `except` is "here's what to do if it breaks," and `finally` is "do this no matter what happened."**

```python
def divide(a, b):
    try:
        result = a / b
    except ZeroDivisionError as e:
        print(f"Error: {e}")
        result = None
    finally:
        print("Execution complete.")
    return result
```

The `finally` block runs whether the division succeeded or failed — that's what makes it different from just putting cleanup code after the `try/except`. It's guaranteed to run, which matters a lot for things like closing files or releasing resources.

---

## Catching the *Right* Exception

The biggest shift in how I think about error handling: I stopped writing a generic `except:` and started catching the **specific exception type** I actually expect. Each built-in exception tells a different story about what went wrong:

| Exception | When it happens |
|---|---|
| `ZeroDivisionError` | Dividing by zero |
| `ValueError` | Right type, wrong value (e.g. `int("abc")`) |
| `TypeError` | Wrong type entirely (e.g. adding a string to an int) |
| `KeyError` | Dictionary key doesn't exist |
| `IndexError` | List index out of range |
| `FileNotFoundError` | File doesn't exist |
| `PermissionError` | No permission to access a file |
| `IOError` | General input/output failure |
| `json.JSONDecodeError` | Malformed JSON string |

**Handling bad user input:**
```python
def get_integer():
    try:
        value = int(input("Enter an integer: "))
    except ValueError as e:
        print(f"Error: {e}")
        value = None
    finally:
        print("Execution complete.")
    return value
```

**Handling a missing dictionary key:**
```python
def get_dict_value(d, key):
    try:
        return d[key]
    except KeyError as e:
        print(f"Error: {e}")
        return None
```

**Handling an out-of-range list index:**
```python
def get_list_element(lst, index):
    try:
        return lst[index]
    except IndexError as e:
        print(f"Error: {e}")
        return None
```

Being specific like this means my error messages actually mean something — "KeyError" tells me instantly it's a dictionary problem, instead of a vague generic failure.

---

## Real-World Exception Sources

A few examples that made this feel less like a toy exercise and more like something I'll actually use:

**Network requests failing:**
```python
import requests

def read_url(url):
    try:
        response = requests.get(url)
        response.raise_for_status()
        return response.text
    except requests.RequestException as e:
        print(f"Network error: {e}")
        return None
```

**Parsing JSON that might be malformed** — this is directly relevant to API work, since I can never fully trust that a response is valid JSON:
```python
import json

def parse_json(json_string):
    try:
        return json.loads(json_string)
    except json.JSONDecodeError as e:
        print(f"JSON error: {e}")
        return None
```

**Type errors when summing a mixed list:**
```python
def sum_list(lst):
    total = 0
    try:
        for item in lst:
            total += item
    except TypeError as e:
        print(f"Error: {e}")
        return None
    return total
```

---

## Nesting try/except When One Failure Can Cause Another

Sometimes one operation depends on another succeeding first (e.g. converting a string to a number, *then* dividing by it). Nesting keeps each failure point isolated with its own specific handling:

```python
def nested_exception_handling(s):
    try:
        try:
            num = int(s)
        except ValueError as e:
            print(f"Conversion error: {e}")
            num = None
        finally:
            print("Conversion attempt complete.")

        if num is not None:
            try:
                result = 10 / num
            except ZeroDivisionError as e:
                print(f"Division error: {e}")
                result = None
            finally:
                print("Division attempt complete.")
            return result
    finally:
        print("Overall execution complete.")
```

This showed me that exception handling isn't just a flat try/except wrapper — it can (and sometimes should) mirror the actual structure of dependent operations in my code.

---

## Custom Exceptions — Making My Own Error Types

Once I understood built-in exceptions, defining my own felt natural — it's just a class that inherits from `Exception`:

```python
class NegativeNumberError(Exception):
    pass

def check_for_negatives(lst):
    try:
        for num in lst:
            if num < 0:
                raise NegativeNumberError(f"Negative number found: {num}")
    except NegativeNumberError as e:
        print(f"Error: {e}")
```

**Why this matters:** custom exceptions let me encode *business logic* errors, not just Python-level errors. `NegativeNumberError` communicates something specific to my program's rules — a generic `ValueError` wouldn't say the same thing.

I saw this pattern again with a `BankAccount` class raising `InsufficientBalanceError` — the exception name itself documents exactly what went wrong, which makes debugging (and reading someone else's code) much faster.

---

## Exceptions Inside Functions, Classes, and Comprehensions

Exception handling isn't confined to simple scripts — it shows up wherever failure is possible:

**Inside a class method:**
```python
class Calculator:
    def divide(self, a, b):
        try:
            return a / b
        except ZeroDivisionError as e:
            print(f"Error: {e}")
            return None
```

**When one function calls another that might raise:**
```python
def risky_function():
    raise ValueError("An error occurred in risky_function.")

def safe_function():
    try:
        risky_function()
    except ValueError as e:
        print(f"Error: {e}")
```

**Inside a list comprehension** — this one surprised me a little, since I originally assumed comprehensions couldn't have try/except inside them directly. The trick is wrapping the whole comprehension, not each element:
```python
def convert_with_comprehension(lst):
    try:
        return [int(item) for item in lst]
    except ValueError as e:
        print(f"Error: {e}")
        return None
```

---

## What I'm Taking Away From This Module

- Catch specific exceptions, not everything — the exception type *is* the diagnostic information
- `finally` guarantees cleanup regardless of success or failure — critical for files, connections, and resources
- Custom exceptions are cheap to create and make my code's failure modes self-documenting
- Nesting try/except should mirror the actual dependency structure of the operations, not just be flat and generic
- This module is really what makes file handling and API calls (from earlier and later modules) *safe* to use in real programs — nothing external can be trusted to always succeed

---

*Personal study notes from Module 7 of the [Complete Python Bootcamp by Krish Naik](https://github.com/krishnaik06/Complete-Python-Bootcamp).*
