

# Python Multithreading Fundamentals — updated Oct 5, 2025

Beginner‑friendly guide to Python threads: when to use them, how to write correct code with the `threading` and `concurrent.futures` modules, how to coordinate work, and how threads interact with the GIL. Use with `002_SETUP.md` for environment setup.

---

**Assumptions and Conventions**
- macOS + zsh, Python 3.12+ or 3.13.  
- Run examples in a venv.  
- Copy‑paste code blocks into files and run with `python file.py`.

---

## Table of Contents
- [0) Threads vs Processes vs Async](#0-threads-vs-processes-vs-async)
- [1) The GIL in one paragraph](#1-the-gil-in-one-paragraph)
- [2) Minimal thread: start and join](#2-minimal-thread-start-and-join)
- [3) Daemon vs non‑daemon threads](#3-daemon-vs-non-daemon-threads)
- [4) Locks, RLocks, and critical sections](#4-locks-rlocks-and-critical-sections)
- [5) Events, Conditions, and Semaphores](#5-events-conditions-and-semaphores)
- [6) Thread‑safe queues (producer/consumer)](#6-thread-safe-queues-producerconsumer)
- [7) concurrent.futures ThreadPoolExecutor](#7-concurrentfutures-threadpoolexecutor)
- [8) Cancellation, timeouts, and exceptions](#8-cancellation-timeouts-and-exceptions)
- [9) Patterns for I/O bound tasks](#9-patterns-for-io-bound-tasks)
- [10) Patterns for CPU bound tasks](#10-patterns-for-cpu-bound-tasks)
- [11) Thread‑safety tips and gotchas](#11-thread-safety-tips-and-gotchas)
- [12) Testing and debugging](#12-testing-and-debugging)
- [13) Troubleshooting](#13-troubleshooting)
- [14) Recap](#14-recap)

---

## 0) Threads vs Processes vs Async
**Use threads** when doing many I/O‑bound operations that block (HTTP calls, disk, database). Each thread can wait while others run.  
**Use processes** for CPU‑bound work. The GIL prevents true CPU parallelism in pure‑Python threads; processes bypass this. See `multiprocessing` or `ProcessPoolExecutor`.  
**Use asyncio** for very high concurrency I/O where the libraries support async; single thread, cooperative scheduling.

Rule of thumb: I/O → threads or asyncio; CPU → processes or native extensions (NumPy, Cython).

---

## 1) The GIL in one paragraph
CPython has a **Global Interpreter Lock**. Only one thread executes Python bytecode at a time. I/O releases the GIL, so threads help for network/disk waits. CPU‑heavy pure‑Python code does not speed up with more threads. Many C‑backed libraries (NumPy, OpenSSL) release the GIL around native code; those can scale better with threads.

---

## 2) Minimal thread: start and join
`threading.Thread` runs a target function concurrently.
```python
# file: t_minimal.py
import threading, time

def worker(name: str):
    print(f"start {name}")
    time.sleep(1)
    print(f"done  {name}")

th = threading.Thread(target=worker, args=("A",))
th.start()
th.join()  # wait until thread finishes
print("all done")
```

---

## 3) Daemon vs non‑daemon threads
- **Non‑daemon** (default): program waits for them to finish on exit.  
- **Daemon**: program exits even if they’re running. Use only for background, non‑critical tasks.
```python
import threading, time

def bg():
    while True:
        time.sleep(0.5)
        print("tick")

th = threading.Thread(target=bg, daemon=True)
th.start()
print("main exits; daemon thread will be killed")
```

---

## 4) Locks, RLocks, and critical sections
Multiple threads updating shared state need mutual exclusion.
```python
# file: t_lock.py
import threading

counter = 0
lock = threading.Lock()

def incr(n: int):
    global counter
    for _ in range(n):
        with lock:         # critical section
            counter += 1

threads = [threading.Thread(target=incr, args=(100_000,)) for _ in range(4)]
[t.start() for t in threads]
[t.join() for t in threads]
print(counter)  # correct: 400000
```
`RLock` is a re‑entrant lock when the same thread must acquire the lock multiple times (nested calls). Prefer `Lock` unless re‑entrancy is required.

---

## 5) Events, Conditions, and Semaphores
- **Event**: one‑bit flag with `set()/is_set()/wait()`. Good for shutdown signals.
```python
import threading, time
stop = threading.Event()

def loop():
    while not stop.is_set():
        time.sleep(0.1)

th = threading.Thread(target=loop); th.start()
time.sleep(1); stop.set(); th.join()
```
- **Condition**: wait/notify pattern for state changes.  
- **Semaphore**: limit concurrent access to a resource (e.g., at most N open sockets).

---

## 6) Thread‑safe queues (producer/consumer)
Use `queue.Queue` for communication; it handles locking for you.
```python
# file: t_queue.py
import threading, queue, time

q: queue.Queue[int] = queue.Queue()
results: list[int] = []

def producer():
    for i in range(10):
        q.put(i)
    q.put(None)  # sentinel to stop consumer

def consumer():
    while True:
        item = q.get()
        if item is None:
            q.task_done()
            break
        time.sleep(0.05)
        results.append(item * item)
        q.task_done()

pt = threading.Thread(target=producer)
ct = threading.Thread(target=consumer)
pt.start(); ct.start()
q.join()  # wait until all items processed
pt.join(); ct.join()
print(results)
```

---

## 7) concurrent.futures ThreadPoolExecutor
Higher‑level pool API for mapping functions across inputs.
```python
# file: t_pool.py
from concurrent.futures import ThreadPoolExecutor, as_completed
import time

def work(x):
    time.sleep(0.2)
    return x * 2

items = list(range(10))
with ThreadPoolExecutor(max_workers=4) as ex:
    futures = [ex.submit(work, x) for x in items]
    for fut in as_completed(futures):
        print(fut.result())
```
`map()` preserves input order; `as_completed()` yields results as soon as each finishes.

---

## 8) Cancellation, timeouts, and exceptions
- A Python thread cannot be force‑killed; design cooperative cancellation with `Event` or queue sentinels.  
- `future.cancel()` prevents running tasks from starting, but cannot stop a running thread.  
- Exceptions in a worker bubble up when you call `future.result()`.
```python
from concurrent.futures import ThreadPoolExecutor, wait, FIRST_EXCEPTION

def bad():
    raise RuntimeError("boom")

with ThreadPoolExecutor() as ex:
    futs = [ex.submit(bad) for _ in range(3)]
    done, pending = wait(futs, return_when=FIRST_EXCEPTION)
    for f in done:
        try:
            f.result()
        except Exception as e:
            print("caught:", e)
    for p in pending:
        p.cancel()
```

---

## 9) Patterns for I/O bound tasks
Example: parallel HTTP downloads with a pool and session reuse.
```python
# file: t_http.py
from concurrent.futures import ThreadPoolExecutor, as_completed
import requests

URLS = ["https://httpbin.org/bytes/1024" for _ in range(8)]

def fetch(url: str, session: requests.Session) -> int:
    r = session.get(url, timeout=10)
    r.raise_for_status()
    return len(r.content)

with requests.Session() as s, ThreadPoolExecutor(max_workers=8) as ex:
    futures = [ex.submit(fetch, u, s) for u in URLS]
    total = 0
    for f in as_completed(futures):
        total += f.result()
    print("downloaded bytes:", total)
```
Tips: reuse connections with `Session`, set timeouts, limit `max_workers` to avoid server overload.

---

## 10) Patterns for CPU bound tasks
Threads will not speed up pure‑Python CPU work due to the GIL. Use processes instead.
```python
# CPU-bound demo: use processes, not threads
from concurrent.futures import ProcessPoolExecutor

def fib(n: int) -> int:
    return n if n < 2 else fib(n-1) + fib(n-2)

with ProcessPoolExecutor() as ex:
    print(list(ex.map(fib, [28, 30, 32])))
```
If you must stay in one process, use C‑accelerated libs (NumPy, Pillow) that release the GIL, or try `multiprocessing.shared_memory` for data exchange.

---

## 11) Thread‑safety tips and gotchas
- Prefer immutable data or confine mutation to one thread.  
- Never access shared structures without a lock unless they are documented thread‑safe.  
- Use `queue.Queue` for communication instead of shared lists.  
- Beware of **deadlocks**: always acquire multiple locks in a consistent order.  
- Avoid blocking calls in GUI threads (Tkinter) — hand off work to a worker thread and post results to the GUI thread (see `007_TKINTER.md`).  
- Logging is thread‑safe; prefer it over `print` for concurrent apps.

---

## 12) Testing and debugging
- Use `pytest` and design deterministic tests.  
- Seed randomness and cap thread counts.  
- Add timeouts to tests to catch hangs.  
- For race conditions, run tests in loops (`pytest -q -k test_name -x --count=50` with `pytest‑repeat`).  
- Capture thread dumps with `faulthandler`:
```python
import faulthandler, signal
faulthandler.register(signal.SIGUSR1)  # kill -USR1 <pid> to dump traces
```

---

## 13) Troubleshooting
- **High CPU, low throughput**: I/O missing timeouts or too many workers. Add `timeout=` and tune `max_workers`.  
- **Program hangs on exit**: you started non‑daemon threads and forgot to `join()`.  
- **Data corruption**: missing locks; wrap mutating sections in `with lock:`.  
- **No speed‑up** with threads on CPU work: GIL; use processes.  
- **Requests fail at scale**: add backoff and retry, respect rate limits.

---

## 14) Recap
```plaintext
Decide I/O vs CPU → Choose threads/processes/async → Start threads → Coordinate with Lock/Event/Queue → Use ThreadPoolExecutor for pools → Handle cancellation & exceptions → Test with timeouts → Profile and tune workers
```

**Next**: Explore `asyncio` for high‑concurrency I/O, `multiprocessing` for CPU‑bound tasks, and structured logging/metrics for observability.