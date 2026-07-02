# Module 6: File Handling — My Learning Notes

*Notes from working through Module 6 of [Krish Naik's Complete Python Bootcamp](https://github.com/krishnaik06/Complete-Python-Bootcamp).*

---

## The `with` Statement Is the Whole Game

The single most important habit I built in this module: **always use `with open(...)` instead of manually calling `open()` and `close()`**. It automatically closes the file for me, even if something goes wrong inside the block — no risk of leaving a file handle open by accident.

```python
def read_file(filename):
    with open(filename, 'r') as file:
        for line in file:
            print(line.strip())
```

Everything else in this module builds on that one pattern.

---

## Reading, Writing, and Appending

The three file modes I now think of as fundamentally different operations, not just a letter I pass in:

| Mode | What it does |
|---|---|
| `'r'` | Read only — file must already exist |
| `'w'` | Write — **overwrites** the entire file if it exists |
| `'a'` | Append — adds to the end without touching existing content |

**Writing a list of lines to a file:**
```python
def write_file(lines, filename):
    with open(filename, 'w') as file:
        for line in lines:
            file.write(line + '\n')
```

**Appending without wiping existing content** — this is the one I'll actually use for logging:
```python
def append_to_file(text, filename):
    with open(filename, 'a') as file:
        file.write(text + '\n')
```

**Copying a file** by opening both the source and destination in the same `with` block:
```python
def copy_file(source, destination):
    with open(source, 'r') as src:
        with open(destination, 'w') as dest:
            dest.write(src.read())
```

---

## Actually Working With File Content

Reading a file isn't just "get the text" — depending on what I need, I reach for a different read method:

- `file.read()` → whole file as one string
- `file.readlines()` → list of lines (useful when I need to reverse, slice, or count them)
- looping `for line in file` → memory-efficient, line by line

**Counting words in a file:**
```python
def count_words(filename):
    with open(filename, 'r') as file:
        text = file.read()
        return len(text.split())
```

**Counting lines, words, and characters together** — this is where `readlines()` earns its keep, since I need the line list for multiple calculations:
```python
def count_lwc(filename):
    with open(filename, 'r') as file:
        lines = file.readlines()
        words = sum(len(line.split()) for line in lines)
        characters = sum(len(line) for line in lines)
    return len(lines), words, characters
```

**Reading a file in reverse order:**
```python
def read_reverse(filename):
    with open(filename, 'r') as file:
        lines = file.readlines()
    for line in reversed(lines):
        print(line.strip())
```

**Find and replace inside a file** — read everything, transform the string in memory, then overwrite:
```python
def find_and_replace(filename, old_word, new_word):
    with open(filename, 'r') as file:
        text = file.read()
    new_text = text.replace(old_word, new_word)
    with open(filename, 'w') as file:
        file.write(new_text)
```

---

## Working With Multiple Files at Once

**Merging several files into one:**
```python
def merge_files(file_list, output_file):
    with open(output_file, 'w') as outfile:
        for fname in file_list:
            with open(fname, 'r') as infile:
                outfile.write(infile.read() + '\n')
```

**Splitting a large file into chunks** — this pattern (slicing a list in steps of N) is one I now recognize as generally useful, not just for files:
```python
def split_file(filename, lines_per_file):
    with open(filename, 'r') as file:
        lines = file.readlines()
    for i in range(0, len(lines), lines_per_file):
        with open(f'{filename}_part{i//lines_per_file + 1}.txt', 'w') as part_file:
            part_file.writelines(lines[i:i + lines_per_file])
```

---

## Beyond Plain Text: Binary, CSV, JSON

**Binary files** need `'rb'` / `'wb'` modes — this matters for anything that isn't plain text (images, etc.):
```python
def copy_binary_file(source, destination):
    with open(source, 'rb') as src:
        with open(destination, 'wb') as dest:
            dest.write(src.read())
```

**CSV files** — using the `csv` module instead of manually splitting on commas (which breaks the moment a value contains a comma):
```python
import csv

def read_csv_as_dicts(filename):
    with open(filename, 'r') as file:
        reader = csv.DictReader(file)
        return list(reader)
```

**JSON files** — this is one I already know I'll be using constantly once I get into LLM/API work, since most API responses are JSON:
```python
import json

def read_json(filename):
    with open(filename, 'r') as file:
        return json.load(file)
```

---

## Handling Things Going Wrong

Real file operations fail — the file doesn't exist, or I don't have permission to read it. Wrapping file operations in `try/except` is what makes this production-ready rather than tutorial-ready:

```python
def read_protected_file(filename):
    try:
        with open(filename, 'r') as file:
            print(file.read())
    except PermissionError as e:
        print(f"Permission error: {e}")
```

**Logging with timestamps** — a small but genuinely practical pattern, combining `datetime` with append mode:
```python
import datetime

def log_message(message, filename='activity.log'):
    timestamp = datetime.datetime.now().isoformat()
    with open(filename, 'a') as file:
        file.write(f'[{timestamp}] {message}\n')
```

---

## What I'm Taking Away From This Module

- `with open(...)` is non-negotiable — it should be my default, not an optimization I add later
- Read mode matters: `read()` vs `readlines()` vs line-by-line iteration each solve a different problem
- `'w'` silently destroys existing content — I need to be deliberate about when I use write vs append
- CSV and JSON aren't "special" file types, they're just plain text with a module that understands their structure — `csv` and `json` save me from writing fragile parsing code myself
- File I/O is where exception handling starts to feel necessary rather than optional, since the outside world (missing files, permissions) is unpredictable — which sets up the next module perfectly

---

*Personal study notes from Module 6 of the [Complete Python Bootcamp by Krish Naik](https://github.com/krishnaik06/Complete-Python-Bootcamp).*
