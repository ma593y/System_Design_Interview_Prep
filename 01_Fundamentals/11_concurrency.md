# Concurrency

## Overview

Concurrency is executing multiple tasks simultaneously or in overlapping time periods. Essential for building responsive, scalable systems.

## Concurrency vs Parallelism

```
Concurrency: Dealing with multiple things at once
Parallelism: Doing multiple things at once

Concurrency (single core):
Task A: ████░░░░████░░░░████
Task B: ░░░░████░░░░████░░░░
        ─────────────────────▶ Time
        (Interleaved execution)

Parallelism (multi-core):
Core 1: ████████████████████
Core 2: ████████████████████
        ─────────────────────▶ Time
        (Simultaneous execution)
```

## Threads vs Processes

### Processes

Independent execution units with own memory space.

```
┌─────────────────┐    ┌─────────────────┐
│    Process A    │    │    Process B    │
├─────────────────┤    ├─────────────────┤
│  Memory Space   │    │  Memory Space   │
│  ┌───────────┐  │    │  ┌───────────┐  │
│  │   Stack   │  │    │  │   Stack   │  │
│  │   Heap    │  │    │  │   Heap    │  │
│  │   Data    │  │    │  │   Data    │  │
│  └───────────┘  │    │  └───────────┘  │
└─────────────────┘    └─────────────────┘
    Isolated              Isolated
```

**Pros:** Isolation, stability, security
**Cons:** Higher overhead, slower IPC

### Threads

Lightweight units within a process, shared memory.

```
┌─────────────────────────────────────────┐
│               Process                    │
├─────────────────────────────────────────┤
│              Shared Memory               │
│  ┌─────────────────────────────────┐    │
│  │     Heap (shared)                │    │
│  │     Global data (shared)         │    │
│  └─────────────────────────────────┘    │
│                                          │
│  ┌────────┐  ┌────────┐  ┌────────┐     │
│  │Thread 1│  │Thread 2│  │Thread 3│     │
│  │ Stack  │  │ Stack  │  │ Stack  │     │
│  └────────┘  └────────┘  └────────┘     │
└─────────────────────────────────────────┘
```

**Pros:** Lightweight, fast communication
**Cons:** Shared state complexity, synchronization needed

## Synchronization Primitives

### Mutex (Mutual Exclusion)

Only one thread can hold the lock.

```python
mutex = Lock()

def critical_section():
    mutex.acquire()
    try:
        # Only one thread here at a time
        shared_resource.update()
    finally:
        mutex.release()

# Or using context manager
with mutex:
    shared_resource.update()
```

### Semaphore

Controls access to a resource pool.

```python
# Allow 3 concurrent database connections
db_semaphore = Semaphore(3)

def query_database():
    with db_semaphore:
        # At most 3 threads here
        connection = get_connection()
        result = connection.query()
        return result
```

### Read-Write Lock

Multiple readers OR one writer.

```python
rwlock = ReadWriteLock()

def read_data():
    with rwlock.read():
        # Multiple readers allowed
        return shared_data

def write_data(value):
    with rwlock.write():
        # Exclusive access
        shared_data = value
```

### Condition Variables

Wait for a condition to be true.

```python
condition = Condition()
queue = []

def producer():
    with condition:
        queue.append(item)
        condition.notify()  # Wake up consumer

def consumer():
    with condition:
        while not queue:
            condition.wait()  # Wait for items
        return queue.pop()
```

## Common Concurrency Problems

### Race Condition

Multiple threads access shared data, result depends on timing.

```python
# Race condition example
counter = 0

def increment():
    global counter
    temp = counter    # Thread 1: temp = 0
    temp = temp + 1   # Thread 2: temp = 0 (same time!)
    counter = temp    # Both set counter = 1, expected 2

# Fix: Use atomic operations or locks
counter_lock = Lock()

def safe_increment():
    global counter
    with counter_lock:
        counter += 1
```

### Deadlock

Two or more threads waiting for each other.

```
Thread 1: Holds Lock A, waiting for Lock B
Thread 2: Holds Lock B, waiting for Lock A

Thread 1          Thread 2
    │                 │
    ▼                 ▼
Lock A ───────────▶ Lock B
   ▲                  │
   │                  │
   └──────────────────┘
        Deadlock!
```

**Prevention Strategies:**
1. **Lock ordering:** Always acquire locks in same order
2. **Lock timeout:** Give up after waiting too long
3. **Deadlock detection:** Detect and break cycles
4. **Avoid nested locks:** Minimize lock scope

### Livelock

Threads keep changing state in response to each other, no progress.

```
Thread 1: "I'll back off for Thread 2"
Thread 2: "I'll back off for Thread 1"
Both: Keep backing off, never proceed
```

### Starvation

A thread never gets resources it needs.

```
High priority threads keep running
Low priority thread never scheduled

Fix: Fair scheduling, priority aging
```

## Async/Await Pattern

Non-blocking I/O without explicit threads.

```python
# Synchronous (blocking)
def fetch_data():
    response1 = http_get(url1)  # Wait...
    response2 = http_get(url2)  # Wait...
    return response1, response2

# Asynchronous (non-blocking)
async def fetch_data():
    task1 = asyncio.create_task(http_get(url1))
    task2 = asyncio.create_task(http_get(url2))
    response1, response2 = await asyncio.gather(task1, task2)
    return response1, response2
```

### Event Loop

```
┌─────────────────────────────────────────┐
│              Event Loop                  │
├─────────────────────────────────────────┤
│  ┌─────────────────────────────────┐    │
│  │         Event Queue             │    │
│  │  [IO Ready] [Timer] [Callback]  │    │
│  └─────────────────────────────────┘    │
│                  │                       │
│                  ▼                       │
│         Process next event              │
│                  │                       │
│         Run associated callback         │
│                  │                       │
│         Back to queue                   │
└─────────────────────────────────────────┘
```

## Thread Pool

Reuse threads instead of creating new ones.

```python
from concurrent.futures import ThreadPoolExecutor

# Create pool with 10 workers
with ThreadPoolExecutor(max_workers=10) as executor:
    # Submit tasks
    futures = [executor.submit(process, item) for item in items]

    # Get results
    results = [f.result() for f in futures]
```

**Benefits:**
- Reduced overhead of thread creation
- Controlled resource usage
- Better memory management

## Lock-Free Data Structures

Using atomic operations instead of locks.

### Compare-and-Swap (CAS)

```python
# Pseudocode for CAS
def compare_and_swap(location, expected, new_value):
    atomic:
        if location == expected:
            location = new_value
            return True
        return False

# Lock-free increment
def atomic_increment(counter):
    while True:
        current = counter.get()
        if counter.compare_and_swap(current, current + 1):
            break
```

### Lock-Free Queue

```
Enqueue:                    Dequeue:
   ┌───┐  ┌───┐  ┌───┐        ┌───┐  ┌───┐  ┌───┐
   │ A │──│ B │──│ C │        │ A │──│ B │──│ C │
   └───┘  └───┘  └───┘        └───┘  └───┘  └───┘
                    ▲          ▲
               Tail (CAS)     Head (CAS)
```

## Concurrency in Distributed Systems

### Distributed Locks

```
┌────────────────────────────────────────┐
│          Lock Service (Redis/ZK)        │
└─────────────────┬──────────────────────┘
                  │
    ┌─────────────┼─────────────┐
    ▼             ▼             ▼
┌────────┐   ┌────────┐   ┌────────┐
│Server 1│   │Server 2│   │Server 3│
│ACQUIRED│   │WAITING │   │WAITING │
└────────┘   └────────┘   └────────┘
```

**Redis Lock (Redlock):**
```
SET resource_name my_unique_id NX PX 30000

NX = Only if not exists
PX = Expire in 30 seconds (prevent deadlock)
```

### Optimistic Locking

Assume no conflicts, detect at commit.

```sql
-- Read with version
SELECT id, data, version FROM items WHERE id = 1
-- version = 5

-- Update only if version matches
UPDATE items SET data = 'new', version = 6
WHERE id = 1 AND version = 5

-- If no rows updated, retry
```

### Pessimistic Locking

Lock before modifying.

```sql
-- Lock the row
SELECT * FROM items WHERE id = 1 FOR UPDATE

-- Modify
UPDATE items SET data = 'new' WHERE id = 1

-- Commit releases lock
COMMIT
```

## Concurrency Patterns

### Producer-Consumer

```
┌──────────┐    ┌─────────────┐    ┌──────────┐
│ Producer │───▶│   Buffer    │───▶│ Consumer │
│ Producer │───▶│   (Queue)   │───▶│ Consumer │
└──────────┘    └─────────────┘    └──────────┘

Decouples production rate from consumption rate
```

### Actor Model

Isolated actors communicate via messages.

```
┌─────────┐      ┌─────────┐      ┌─────────┐
│ Actor A │─msg─▶│ Actor B │─msg─▶│ Actor C │
│ Mailbox │      │ Mailbox │      │ Mailbox │
│  State  │      │  State  │      │  State  │
└─────────┘      └─────────┘      └─────────┘

No shared state, message passing only
(Erlang, Akka)
```

### Fork-Join

Split work, process in parallel, combine results.

```
        ┌─── Fork ───┐
        ▼            ▼
    ┌───────┐   ┌───────┐
    │Task 1 │   │Task 2 │
    └───┬───┘   └───┬───┘
        │           │
        └─── Join ──┘
              │
           Result
```

## Interview Talking Points

1. "Use locks for shared mutable state, prefer immutable data when possible"
2. "Async I/O for I/O-bound tasks, threads for CPU-bound tasks"
3. "Always acquire locks in consistent order to prevent deadlocks"
4. "Thread pools reuse threads and control resource usage"
5. "Optimistic locking for low-contention scenarios"

## Common Interview Questions

1. **Q: What is a race condition and how do you prevent it?**
   A: When multiple threads access shared data and result depends on timing. Prevent with locks, atomic operations, or immutable data.

2. **Q: How do you prevent deadlocks?**
   A: Lock ordering, timeouts, minimize lock scope, or use lock-free data structures.

3. **Q: When would you use async vs threads?**
   A: Async for I/O-bound work (network, disk), threads for CPU-bound work. Async has less overhead for many concurrent I/O operations.

4. **Q: How do distributed locks work?**
   A: Use external coordination service (Redis, ZooKeeper). Set lock with expiration to prevent deadlock if holder dies.

## Key Takeaways

- Understand the difference between concurrency and parallelism
- Use appropriate synchronization primitives
- Avoid shared mutable state when possible
- Be aware of deadlock, race conditions, and starvation
- Choose between threads and async based on workload type

## Further Reading

- "Java Concurrency in Practice" by Brian Goetz
- "Designing Data-Intensive Applications" Chapter 7
- Go concurrency patterns
- Erlang/Elixir actor model
