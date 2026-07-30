# Python Memory Management — Allocation, Deallocation, and Garbage Collection

*Notes from Krish Naik's Complete Python Bootcamp — [Memory Management module](https://github.com/krishnaik06/Complete-Python-Bootcamp/tree/main/15-Memory%20Management)*

Every Python program creates objects — lists, strings, class instances, numbers, all of it. Someone has to keep track of where all that stuff lives in the computer's memory, and someone has to clean it up once it's no longer needed. In most lower-level languages (C, for example) that "someone" is the programmer. In Python, it's the language itself. This note is about understanding exactly how Python does that job, so I'm not just trusting a black box — I actually know what's happening underneath.

---

## 1. What is memory management, and why does it matter?

**Definition:** Memory management is the process of allocating memory to objects when they're created, and freeing (deallocating) that memory once those objects are no longer needed, so the same memory can be reused for something else.

Every object in a running program takes up space in RAM. If a program keeps creating objects and never frees the memory they used, the program's memory usage keeps climbing until, eventually, it can crash or slow the whole system down. This is called a **memory leak**.

Python handles this automatically through a combination of three mechanisms working together:
1. **Reference counting** — the primary, everyday method
2. **Garbage collection** — a backup mechanism for a specific edge case reference counting can't solve on its own
3. **Internal memory optimizations** — Python reuses and pools memory internally rather than constantly asking the operating system for fresh memory

Understanding these isn't just trivia — it directly affects how efficient and bug-free the code I write actually is, especially once a program starts handling large amounts of data.

---

## 2. Reference counting — Python's primary method

**Definition:** Reference counting means every object in Python keeps an internal count of how many "references" (variables, list entries, other objects) are currently pointing at it. The moment that count reaches zero — meaning nothing in the program can reach that object anymore — Python immediately frees the memory it was using.

**How it works, step by step:**
- When an object is created and assigned to a variable, its reference count starts at 1 (or higher, since the act of checking it also temporarily adds a reference).
- Every time another variable is set to point at the same object, the count goes up by 1.
- Every time a reference is deleted, goes out of scope, or is reassigned to something else, the count goes down by 1.
- The instant the count hits 0, the object is deallocated immediately — there's no waiting around.

This is why Python's memory cleanup feels mostly invisible and instant — it's not running on a timer or a schedule, it's reacting the moment an object becomes unreachable.

Here's the exact experiment I ran to see this in action:

```python
import sys

a = []
## 2 (one reference from 'a' and one from getrefcount() itself)
print(sys.getrefcount(a))
```

`sys.getrefcount()` tells you the current reference count of an object. The result here is `2`, not `1` — and the reason is worth understanding properly: one reference is the variable `a` itself, and the second is a *temporary* reference created just by passing `a` into `getrefcount()` as an argument. So the function's own act of checking briefly bumps the count.

Now watch what happens when I create a second variable pointing at the same list:

```python
b = a
print(sys.getrefcount(b))
```

The count goes up, because now both `a` and `b` point at the exact same list object in memory — not two separate lists, the *same* one.

```python
del b
print(sys.getrefcount(a))
```

Once `b` is deleted, the count drops back down. The object itself is still alive (because `a` still points to it), but one fewer thing is holding a reference to it now.

**The analogy that makes this click:** think of an object's reference count like a light switch with multiple physical switches wired to the same bulb. As long as at least one switch is flipped "on" (as long as at least one variable/reference still points at the object), the bulb stays lit — the object stays alive. The instant the *last* switch is flipped off — the last reference is removed — the bulb goes out immediately, and Python reclaims that memory right then, not later.

---

## 3. The one problem reference counting can't solve: circular references

Reference counting is fast and simple, but it has exactly one blind spot: **objects that reference each other in a cycle, with nothing outside the cycle pointing at either of them.**

**Definition:** A circular reference (also called a reference cycle) happens when two or more objects hold references to each other, forming a loop, such that even if nothing else in the program can reach any of them, their reference counts never actually drop to zero — because they're still "holding hands" with each other.

Here's the exact scenario, recreated in code:

```python
import gc

class MyObject:
    def __init__(self, name):
        self.name = name
        print(f"Object {self.name} created")

    def __del__(self):
        print(f"Object {self.name} deleted")

# Create a circular reference
obj1 = MyObject("obj1")
obj2 = MyObject("obj2")
obj1.ref = obj2
obj2.ref = obj1

del obj1
del obj2

## Manually trigger garbage collection
gc.collect()
```

Walking through what actually happens here: `obj1` and `obj2` are created, and then each one is given a `.ref` attribute pointing at the *other* one. Now `obj1` is referenced by `obj2.ref`, and `obj2` is referenced by `obj1.ref`. They're pointing at each other in a closed loop.

When `del obj1` and `del obj2` run, the *variable names* `obj1` and `obj2` are removed — but each object still has one reference left, coming from the other object's `.ref` attribute! Neither reference count actually reaches zero, even though, from the program's point of view, there's no longer any way to reach either object. They're floating in memory, unreachable, but not yet cleaned up. This is exactly what a **memory leak** looks like in a reference-counted system.

This is precisely why Python needs a second mechanism on top of reference counting.

---

## 4. Garbage collection — the backup mechanism for cycles

**Definition:** Garbage collection, in Python, specifically refers to the **cyclic garbage collector** — a separate system that periodically scans for groups of objects that reference each other in a cycle but are unreachable from anywhere else in the program, and cleans them up even though their individual reference counts never hit zero.

This is a genuinely important distinction to keep straight: reference counting handles the vast majority of memory cleanup automatically and instantly. Garbage collection is a *supplementary* system that only needs to step in for the specific circular-reference case reference counting structurally cannot solve on its own.

Python's garbage collector is accessible through the built-in `gc` module:

```python
import gc

## enable garbage collection (it's on by default, but this is how you'd explicitly turn it on)
gc.enable()

## disable it
gc.disable()

## manually force a collection cycle right now
gc.collect()

## get statistics about the garbage collector's activity
print(gc.get_stats())

## get objects that were found to be unreachable but couldn't be freed (rare, but possible)
print(gc.garbage)
```

**How the cyclic collector actually works, conceptually:** Python's objects are organized by the garbage collector into "generations" (generation 0, 1, and 2), based on how long they've survived. New objects start in generation 0. Every time a garbage collection pass runs and an object survives it, it gets promoted to the next generation. The idea behind this: most objects that become garbage do so very young (short-lived temporary variables, loop iterators, etc.), so generation 0 gets scanned most frequently, while older objects (which have already proven they tend to stick around) get scanned less often. This makes the whole process more efficient than treating every single object equally.

Calling `gc.collect()` manually forces an immediate full scan across all generations right now, rather than waiting for Python's automatic schedule to get around to it. In the circular reference example above, calling `gc.collect()` is what finally triggers `obj1` and `obj2`'s `__del__` methods to run, printing `"Object obj1 deleted"` and `"Object obj2 deleted"` — proof that the cycle was found and cleaned up.

---

## 5. Memory management best practices

These are the five practices called out directly in the module, and each one maps back to something specific about how reference counting and garbage collection actually work underneath.

### 5.1 Prefer local variables over global ones

**Why this matters:** a local variable (one defined inside a function) only exists for the duration of that function call. The moment the function returns, Python automatically removes that reference, and if nothing else refers to that object, it's freed immediately via reference counting. A global variable, by contrast, sticks around for the entire lifetime of the program (or module) — so anything it points to also sticks around, whether it's still actually needed or not.

**In plain terms:** local variables get cleaned up automatically and promptly, just by virtue of the function ending. Globals require conscious effort to clean up, and are easy to forget about.

### 5.2 Avoid circular references where possible

Section 3 and 4 above cover exactly why: cycles bypass the fast, instant reference-counting cleanup and instead have to wait around for the (slower, periodic) garbage collector to notice them. In a program creating and discarding a lot of interlinked objects — think a tree or graph structure where nodes point back to their parents — this can genuinely add up to real memory and performance overhead if it happens at scale.

### 5.3 Use generators for memory efficiency

**Definition:** A generator is a special kind of function (using `yield` instead of `return`) that produces values **one at a time, on demand**, instead of computing and holding an entire collection in memory all at once.

```python
## Using a generator instead of building a full list in memory
def generate_numbers(n):
    for i in range(n):
        yield i

## using the generator
for num in generate_numbers(100000):
    print(num)
    if num > 10:
        break
```

Compare this to the "obvious" alternative:

```python
def generate_numbers_list(n):
    return [i for i in range(n)]
```

The list version has to build and hold *all* 100,000 numbers in memory at once, even if the code using it only ever looks at the first 11 of them (like the loop above does with its `break`). The generator version, on the other hand, produces exactly one number at a time, only when asked (via each iteration of the `for` loop) — so at any given moment, it's only holding a single number in memory, not the whole sequence.

**The analogy that makes this obvious:** a list is like printing an entire 100,000-page book in one go, whether or not anyone plans to read past page 11. A generator is like a printer that prints one page at a time, exactly when the reader turns to it — dramatically less paper wasted if the reader stops early, and dramatically less memory used no matter how far they go.

### 5.4 Explicitly delete objects you're done with

Using `del` on a variable removes that reference immediately, rather than waiting for it to naturally go out of scope (say, at the end of a long function). For large objects that are no longer needed partway through a long-running block of code, this can free up memory sooner rather than later.

```python
big_data = [i for i in range(10_000_000)]
## ... use big_data for something ...
del big_data   # explicitly release it now, don't wait for the function to end
```

### 5.5 Profile memory usage — don't guess, measure

**Definition:** Memory profiling means using a tool to actually observe how much memory your code is using, and exactly which lines are responsible for allocating it — instead of guessing where a memory problem might be coming from.

Python's built-in `tracemalloc` module does exactly this:

```python
import tracemalloc

def create_list():
    return [i for i in range(10000)]

def main():
    tracemalloc.start()

    create_list()

    snapshot = tracemalloc.take_snapshot()
    top_stats = snapshot.statistics('lineno')

    print("[ Top 10 ]")
    for stat in top_stats[::]:
        print(stat)

main()
```

Walking through what this does: `tracemalloc.start()` begins tracking every memory allocation from that point forward. `take_snapshot()` captures a point-in-time picture of everything currently allocated. `snapshot.statistics('lineno')` groups all of that memory usage by the exact line of code responsible for it — so instead of just knowing "my program uses X megabytes," I know precisely *which line* is responsible for how much of it. This is the real-world tool for hunting down memory leaks or unexpectedly heavy allocations in a larger project, rather than relying on guesswork.

(There's also a popular third-party library, `memory_profiler`, which does something similar but at the level of decorating individual functions — worth knowing it exists, even though `tracemalloc` alone is enough to get started.)

---

## 6. Bringing it all together — how these pieces fit

Here's the full picture, laid out as it would happen for a typical object during a program's life:

1. **Allocation:** an object is created (e.g. `a = []`), and Python allocates memory for it, setting its reference count to reflect however many variables/structures currently point to it.
2. **Reference counting, continuously:** as the program runs, that count goes up and down as references are created and removed (new variables pointing at it, `del` statements, variables going out of scope, being reassigned, etc.).
3. **Immediate deallocation (the common case):** the moment the count hits zero, Python frees that memory right away. This handles the overwhelming majority of objects in a typical program.
4. **Garbage collection (the exception case):** for the specific case of circular references — where two or more objects keep each other's reference count above zero forever, despite being unreachable from the rest of the program — the cyclic garbage collector periodically (or manually, via `gc.collect()`) scans for these cycles and cleans them up.
5. **Best practices layer on top:** using local variables, avoiding unnecessary cycles, using generators instead of building full collections in memory, deleting large objects explicitly when done, and profiling with `tracemalloc` are all practical habits that either help reference counting do its job sooner, reduce how often garbage collection has to step in, or simply reduce how much memory the program needs in the first place.

---

## 7. Cheat sheet

| Concept | What it means |
|---|---|
| Reference counting | every object tracks how many references point to it; hits 0 → freed instantly |
| `sys.getrefcount(obj)` | check an object's current reference count (adds 1 temporarily just by being called) |
| Circular reference | two+ objects reference each other, keeping counts above 0 even when unreachable |
| Garbage collector (`gc` module) | scans for and cleans up circular references reference counting can't catch on its own |
| `gc.collect()` | manually force an immediate collection pass right now |
| Generator (`yield`) | produces values one at a time, keeping only one in memory instead of a whole collection |
| `del` | explicitly remove a reference immediately, instead of waiting for scope to end |
| `tracemalloc` | measure exactly how much memory is used, and which line of code is responsible |

**The one-sentence summary worth remembering:** Python handles memory automatically through reference counting for almost everything, backed up by a cyclic garbage collector for the one case (circular references) reference counting can't solve alone — and the best practices above are really just ways of working *with* that system instead of accidentally fighting it.
