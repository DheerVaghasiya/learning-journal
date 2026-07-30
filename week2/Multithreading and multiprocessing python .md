# Multithreading and Multiprocessing in Python

*Day X of my learning journal — Krish Naik's Complete Python Bootcamp ([Multithreading and Multiprocessing module](https://github.com/krishnaik06/Complete-Python-Bootcamp/tree/main/16-Multithreading%20and%20Multiprocessing))*

This one took me longer to actually "get" than SQLite3 or logging did. The syntax isn't hard, but the *mental model* of why threads and processes behave so differently in Python took a few re-reads. Writing this note slow, on purpose, so future-me doesn't have to relearn this from scratch.

---

## 1. What even is a process, and what is a thread?

**A process** is a running program with its **own private chunk of memory**. When you open Chrome and open VS Code, that's two separate processes — they don't share memory, and if one crashes, the other is completely unaffected.

**A thread** is a smaller unit *inside* a process. A single process can have multiple threads, and all of those threads **share the same memory space** of their parent process.

**The analogy that finally made this click for me:** think of a process as an entire office building. A thread is an employee working inside that building. Multiple employees (threads) share the same building, same filing cabinets, same coffee machine (shared memory) — which makes them fast to spin up and easy to collaborate with, but also means two employees can literally bump into each other reaching for the same file at the same time (this is the race condition problem, more on that later).

Multiple processes, on the other hand, are like multiple *separate office buildings*. Each one has its own filing cabinets — nothing is shared by default, so there's no risk of two buildings' employees fighting over the same file. But now if Building A needs a file from Building B, someone has to physically drive it over — that's the overhead of inter-process communication.

| | Process | Thread |
|---|---|---|
| Memory | own separate memory space | shares memory with sibling threads |
| Creation cost | heavier, slower to spin up | lightweight, fast to spin up |
| Crash impact | one process crashing doesn't kill others | a bad thread can corrupt shared state for the whole process |
| Best for | CPU-heavy work (crunching numbers) | I/O-heavy work (waiting on network/disk) |

### 1.1 The GIL — the thing that trips everyone up

Python has something called the **GIL (Global Interpreter Lock)**. It means: no matter how many threads you spin up, only **one thread can execute Python bytecode at a time** within a single process.

This was the single biggest "wait, what?" moment for me in this whole module. I assumed more threads = more speed, always. Turns out, for pure CPU-bound work (like crunching a huge math loop), Python threads **don't actually run in parallel** — they take turns, controlled by the GIL. So multithreading a CPU-heavy task in Python can end up *no faster*, sometimes even slightly slower, than just running it single-threaded.

This is exactly *why* multiprocessing exists as a separate tool — each process gets its **own Python interpreter and its own GIL**, so processes genuinely run in parallel across CPU cores. Threads don't get that luxury in CPython.

**The one rule that resolves 90% of the "should I use threading or multiprocessing" confusion:**

> **I/O-bound work (waiting on network, disk, APIs, user input) → use threading.**
> **CPU-bound work (heavy computation, number crunching) → use multiprocessing.**

Why threading still helps for I/O-bound work even with the GIL: while one thread is *waiting* on a network response, it releases the GIL, letting another thread run. So threads are great at overlapping "waiting time," just not at overlapping "actual computing time."

---

## 2. Multithreading — practical implementation

Python's built-in `threading` module is what I'll use for this.

### 2.1 Creating and running a basic thread

```python
import threading
import time

def print_numbers():
    for i in range(5):
        time.sleep(1)
        print(f"Number: {i}")

def print_letters():
    for letter in 'abcde':
        time.sleep(1)
        print(f"Letter: {letter}")

# create thread objects
t1 = threading.Thread(target=print_numbers)
t2 = threading.Thread(target=print_letters)

# start them
t1.start()
t2.start()

# wait for both to finish before moving on
t1.join()
t2.join()

print("Done!")
```

Three calls to actually remember:
- `threading.Thread(target=some_function)` — creates the thread, but **does not run it yet**.
- `.start()` — actually kicks it off.
- `.join()` — makes the main program **wait** for that thread to finish before continuing.

If I skip `.join()`, the `print("Done!")` line could run *before* the threads finish — the main program doesn't automatically wait around for background threads. This bit me the first time I ran this without `join()` — "Done!" printed immediately while numbers and letters were still being printed after it.

Because both threads run `time.sleep(1)` five times, and they run **concurrently**, the whole thing finishes in about 5 seconds total — not 10. That's the entire point of using threads here: the waiting overlaps.

### 2.2 Passing arguments to a thread

```python
def greet(name, times):
    for _ in range(times):
        print(f"Hello, {name}")
        time.sleep(0.5)

t = threading.Thread(target=greet, args=("Dheer", 3))
t.start()
t.join()
```

`args` takes a tuple — exactly the same pattern as `executemany()` in SQLite3 from my last note. Python loves this tuple-of-arguments convention.

### 2.3 The race condition problem — and fixing it with `Lock`

This is the scary part of shared memory I mentioned in the analogy earlier. If two threads modify the same variable at the same time, you can get wrong results, unpredictably:

```python
counter = 0

def increment():
    global counter
    for _ in range(100000):
        counter += 1

threads = [threading.Thread(target=increment) for _ in range(2)]
for t in threads:
    t.start()
for t in threads:
    t.join()

print(counter)  # you'd expect 200000... but you might not get it
```

Why this breaks: `counter += 1` isn't actually one atomic step — it's "read counter, add 1, write it back." If thread A reads the value right before thread B also reads it (before either writes back), one of their updates gets silently overwritten. This is a **race condition**.

The fix is a `Lock` — it makes sure only one thread can touch that shared piece of code at a time:

```python
counter = 0
lock = threading.Lock()

def increment():
    global counter
    for _ in range(100000):
        with lock:
            counter += 1

threads = [threading.Thread(target=increment) for _ in range(2)]
for t in threads:
    t.start()
for t in threads:
    t.join()

print(counter)  # now reliably 200000
```

**Analogy:** a `Lock` is like a single bathroom key at a gas station — only one person can be inside at a time, everyone else waits their turn outside. Slower, but nobody walks in on anybody else mid-task.

---

## 3. Multiprocessing — practical implementation

Same overall shape as threading, but using the `multiprocessing` module instead — and because each process gets its own memory and its own interpreter, this is where genuine CPU parallelism happens.

### 3.1 Creating and running a basic process

```python
import multiprocessing
import time

def square_numbers():
    for i in range(5):
        time.sleep(1)
        print(f"Square: {i * i}")

def cube_numbers():
    for i in range(5):
        time.sleep(1)
        print(f"Cube: {i * i * i}")

if __name__ == "__main__":
    p1 = multiprocessing.Process(target=square_numbers)
    p2 = multiprocessing.Process(target=cube_numbers)

    p1.start()
    p2.start()

    p1.join()
    p2.join()

    print("Done!")
```

Syntax-wise this is basically a copy-paste of the threading pattern — `Process` instead of `Thread`, same `.start()` / `.join()`. The API was clearly designed to feel familiar on purpose.

**Important gotcha that cost me a confusing 20 minutes:** the `if __name__ == "__main__":` guard is **required** for multiprocessing on Windows (and good practice everywhere). Without it, when a new process starts, it re-imports your script from scratch — and without the guard, that re-import would try to spawn *another* set of processes infinitely. Threading doesn't have this problem since it doesn't spawn a new interpreter.

### 3.2 Sharing data between processes

Since processes don't share memory by default, you can't just use a global variable like I did with threads. You need special shared objects:

```python
import multiprocessing

def worker(shared_list, lock):
    with lock:
        shared_list.append("done by a process")

if __name__ == "__main__":
    manager = multiprocessing.Manager()
    shared_list = manager.list()
    lock = multiprocessing.Lock()

    processes = [multiprocessing.Process(target=worker, args=(shared_list, lock)) for _ in range(3)]

    for p in processes:
        p.start()
    for p in processes:
        p.join()

    print(shared_list)
```

`multiprocessing.Manager()` creates a special shared object that lives outside any single process, which every process can safely read/write to. This is the multiprocessing equivalent of the `Lock` problem from threading — same idea, different tool, because the underlying memory model is different.

---

## 4. ThreadPoolExecutor and ProcessPoolExecutor — the cleaner way to do this

Manually creating and tracking a list of `Thread`/`Process` objects gets messy once you have more than a handful of tasks. The `concurrent.futures` module gives you a **pool** — you just hand it tasks, and it manages the workers for you.

### 4.1 ThreadPoolExecutor

```python
from concurrent.futures import ThreadPoolExecutor
import time

def task(n):
    time.sleep(1)
    return f"Task {n} done"

with ThreadPoolExecutor(max_workers=3) as executor:
    results = executor.map(task, range(6))
    for result in results:
        print(result)
```

`max_workers=3` means only 3 threads run at once — as soon as one finishes, the pool automatically hands it the next pending task. I don't manage `start()`/`join()` manually anymore; the `with` block handles cleanup for me.

`.submit()` is the version I reach for when I want more control (e.g. I need the actual `Future` object to check status or grab a result later, instead of just mapping over everything at once):

```python
with ThreadPoolExecutor(max_workers=3) as executor:
    futures = [executor.submit(task, i) for i in range(6)]
    for future in futures:
        print(future.result())
```

### 4.2 ProcessPoolExecutor

Exact same interface, but for CPU-bound work across processes:

```python
from concurrent.futures import ProcessPoolExecutor

def cpu_heavy_task(n):
    return sum(i * i for i in range(n))

if __name__ == "__main__":
    with ProcessPoolExecutor(max_workers=4) as executor:
        results = executor.map(cpu_heavy_task, [10_000_000, 10_000_000, 10_000_000, 10_000_000])
        for result in results:
            print(result)
```

**Rule of thumb I'm keeping from this section:** if I ever catch myself manually writing `Thread()`/`Process()` in a loop and tracking a list to `join()` later, that's usually a sign I should just reach for the Pool executor instead — it's less code and handles edge cases (like exceptions inside a worker) more gracefully.

---

## 5. Real use case #1 — web scraping multiple pages with multithreading

This is the textbook "I/O-bound" example: fetching a page over the network means *waiting* most of the time, not computing. Perfect job for threads.

```python
import requests
import time
from concurrent.futures import ThreadPoolExecutor

urls = [
    "https://example.com",
    "https://example.org",
    "https://example.net",
    "https://httpbin.org/delay/1",
    "https://httpbin.org/delay/2",
]

def fetch_page(url):
    response = requests.get(url, timeout=10)
    return f"{url} -> status {response.status_code}, {len(response.content)} bytes"

# --- sequential version, just to see the difference for myself ---
start = time.time()
for url in urls:
    print(fetch_page(url))
print(f"Sequential time: {time.time() - start:.2f}s")

# --- threaded version ---
start = time.time()
with ThreadPoolExecutor(max_workers=5) as executor:
    results = executor.map(fetch_page, urls)
    for result in results:
        print(result)
print(f"Threaded time: {time.time() - start:.2f}s")
```

Running both back to back made the whole "threads are great for I/O" point finally *feel* real instead of just being something I read. The sequential version's total time is roughly the sum of every page's fetch time. The threaded version's total time is roughly just the time of the *slowest single page*, because all the waiting overlaps.

Things I want to remember for real scraping projects:
- Always set a `timeout` on requests — a hung server shouldn't hang my whole pool.
- Don't crank `max_workers` too high when hitting the same website — that's basically a mini DDoS on someone else's server, and a lot of sites will just start blocking/rate-limiting me. A handful of workers per host is usually plenty.
- Wrap `fetch_page` in a `try/except` in real code so one failed URL doesn't blow up the whole batch — the same "don't let one failure ruin everything" instinct from the transactions section of my SQLite3 note applies here too.

---

## 6. Real use case #2 — CPU-heavy batch processing with multiprocessing

The classic CPU-bound example: something genuinely computation-heavy, like checking a batch of numbers for primality, or resizing/processing a batch of images. I'll use a prime-checking example since it's easy to reason about and easy to make artificially "heavy."

```python
import time
from concurrent.futures import ProcessPoolExecutor

def is_prime(n):
    if n < 2:
        return (n, False)
    for i in range(2, int(n ** 0.5) + 1):
        if n % i == 0:
            return (n, False)
    return (n, True)

numbers = list(range(100_000, 100_200))

if __name__ == "__main__":
    # --- sequential ---
    start = time.time()
    results_sequential = [is_prime(n) for n in numbers]
    print(f"Sequential time: {time.time() - start:.2f}s")

    # --- multiprocessing ---
    start = time.time()
    with ProcessPoolExecutor(max_workers=4) as executor:
        results_parallel = list(executor.map(is_prime, numbers))
    print(f"Multiprocessing time: {time.time() - start:.2f}s")

    primes_found = [n for n, prime in results_parallel if prime]
    print(f"Found {len(primes_found)} primes")
```

This is the flip side of the scraping example — here, threading wouldn't have helped much at all because of the GIL (all the work is pure computation, no waiting). Splitting the number range across 4 processes lets 4 CPU cores genuinely work in parallel, which is why multiprocessing shows a real speedup here where threading wouldn't have.

**Where I can see myself actually using this later:** batch-processing a folder of images for a project, running the same ML preprocessing step across thousands of rows of a dataset, or splitting up a big scraping-then-parsing pipeline where the *parsing* step (CPU-heavy) benefits from multiprocessing even though the *fetching* step (I/O-heavy) benefits from threading. Realized while writing this: a serious pipeline could combine both — threads to fetch, processes to crunch what got fetched.

---

## 7. My personal cheat sheet

| Question | Answer |
|---|---|
| Task waits a lot (network, disk, APIs)? | `threading` / `ThreadPoolExecutor` |
| Task is pure computation, CPU-bound? | `multiprocessing` / `ProcessPoolExecutor` |
| Need to run this on Windows or want safe practice everywhere? | wrap process code in `if __name__ == "__main__":` |
| Multiple threads touching the same variable? | protect it with `threading.Lock()` |
| Multiple processes need to share data? | `multiprocessing.Manager()` + a `Lock` |
| Got more than a few tasks to run concurrently? | reach for a Pool executor, not manual `Thread()`/`Process()` loops |

Rules going forward:
1. Always ask "is this I/O-bound or CPU-bound?" *before* picking threading vs multiprocessing — this single question resolves most of the confusion.
2. Never forget `.join()` (or the `with` block for pools) — otherwise the main program won't actually wait for background work to finish.
3. Shared mutable state is where bugs hide — protect it with a `Lock`, or better, avoid sharing state at all if the design allows it.
4. For processes, the `if __name__ == "__main__":` guard isn't optional — treat it as mandatory boilerplate.
5. Default to `ThreadPoolExecutor` / `ProcessPoolExecutor` over raw `Thread`/`Process` unless I have a specific reason not to — cleaner code, safer cleanup.

Next up: want to try combining both patterns in one small project — a scraper that fetches pages with threads, then hands the heavy parsing/text-processing off to a process pool. Feels like the natural next step after writing this note.
