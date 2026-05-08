# Lab: Deadlock Detection and Prevention

## Overview

In this lab, you will experience deadlock firsthand, then implement
three different strategies to detect and prevent it. You will build a
concurrent banking system where multiple threads transfer money between
accounts simultaneously. The naive implementation deadlocks; your job is
to fix it three different ways and understand the tradeoffs.

Learning Goals:

- Experience a real deadlock scenario
- Understand all four necessary conditions for deadlock
- Implement timeout-based deadlock detection
- Implement two deadlock prevention strategies (ordered locking and
  try-lock with backoff)
- Compare performance and complexity tradeoffs between approaches

Create a file called `answers.txt` for the answers at the end.

------------------------------------------------------------------------

## Background

### The Four Conditions for Deadlock

Deadlock can only occur when **all four** of the following conditions
hold simultaneously:

1.  **Mutual exclusion**: At least one resource is held in a
    non-shareable mode. Only one thread can hold a mutex at a time.
2.  **Hold-and-wait**: A thread holds at least one resource while
    waiting to acquire additional resources held by other threads.
3.  **No preemption**: Resources cannot be forcibly taken away from a
    thread; a thread must voluntarily release its resources.
4.  **Circular wait**: A cycle exists in the wait-for graph: Thread A
    waits for Thread B, which waits for Thread C, which waits for Thread
    A (or any length cycle).

If you can break *any one* of these conditions, deadlock becomes
impossible.

### Resource Allocation Graphs

A **resource allocation graph** (or wait-for graph) models which threads
hold which resources and which threads are waiting for which resources.
A deadlock corresponds to a **cycle** in this graph. For example:

    Thread A --holds--> Account 1
    Thread A --waits--> Account 2
    Thread B --holds--> Account 2
    Thread B --waits--> Account 1
    (cycle: deadlock!)

### Detection vs Prevention

**Detection** allows deadlock to occur, then notices it (via timeout or
graph analysis) and recovers by releasing locks and retrying. It is
simple to implement but costly at runtime.

**Prevention** structurally eliminates one of the four conditions so
deadlock can never form. Prevention is typically more efficient but
requires careful design of the locking protocol.

------------------------------------------------------------------------

## Part 0: Scenario {#part-0-the-problem}

Imagine you are building the core transaction system for a bank.
Multiple threads process transfers between accounts simultaneously. Each
transfer must be **atomic**: both the deduction from the source and the
credit to the destination must happen together, or not at all. This
requires locking both accounts for the duration of the transfer.

A naive approach:

    To transfer money from Account A to Account B:
    1. Lock Account A
    2. Lock Account B
    3. Deduct money from A
    4. Add money to B
    5. Unlock B
    6. Unlock A

This seems reasonable. But consider what happens when two threads
execute concurrently:

- Thread 1 transfers from Account 1 to Account 2
- Thread 2 transfers from Account 2 to Account 1

Thread 1 locks Account 1 and tries to lock Account 2. Thread 2 locks
Account 2 and tries to lock Account 1. Both threads are now waiting for
a lock the other holds. Neither can proceed. **Deadlock.**

In this lab you will observe this deadlock, then implement three
strategies to handle it.

------------------------------------------------------------------------

## Part 1: Observe Deadlock

### Core Structures

The starter code provides a header file `common.h` with the following
account structure:

``` {.sourceCode .c}
typedef struct {
    int id;
    int balance;
    pthread_mutex_t lock;
} account_t;
```

It also provides utility functions: `init_accounts()`,
`print_balances()`, `verify_total()`, and error handling helpers. A key
invariant: **money is conserved**. The total balance across all accounts
must remain constant throughout all transfers.

### The Naive Transfer

Examine the provided `transfer_naive()` function in `bank.c`:

``` {.sourceCode .c}
static void transfer_naive(account_t *from, account_t *to, int amount) {
    /* Lock source account */
    pthread_mutex_lock(&from->lock);

    /* Simulate some work */
    usleep(1);

    /* Lock destination account (DANGER: may deadlock here!) */
    pthread_mutex_lock(&to->lock);

    /* Perform transfer */
    from->balance -= amount;
    to->balance += amount;

    /* Unlock in reverse order */
    pthread_mutex_unlock(&to->lock);
    pthread_mutex_unlock(&from->lock);

    __sync_fetch_and_add(&stats.successful_transfers, 1);
}
```

The `worker()` function runs in each thread, repeatedly picking random
source and destination accounts and calling the appropriate transfer
function:

``` {.sourceCode .c}
static void *worker(void *arg) {
    struct worker_args *args = (struct worker_args *)arg;

    for (int i = 0; i < args->transfers; i++) {
        /* Pick random source and destination (must be different) */
        int from_id = rand() % args->num_accounts;
        int to_id = rand() % args->num_accounts;
        if (from_id == to_id) {
            i--;  /* Skip this iteration */
            continue;
        }

        account_t *from = &args->accounts[from_id];
        account_t *to = &args->accounts[to_id];
        int amount = 1 + (rand() % 10);  /* Transfer 1-10 dollars */

        /* Call appropriate transfer function based on mode */
        int retry_count = 0;
        int max_retries = 1000;

        if (args->mode == 0) {  /* NAIVE */
            transfer_naive(from, to, amount);
        } else if (args->mode == 1) {  /* TIMEOUT */
            while (transfer_timeout(from, to, amount) != 0) {
                __sync_fetch_and_add(&stats.retries, 1);
                if (++retry_count > max_retries) {
                    __sync_fetch_and_add(&stats.failed_transfers, 1);
                    break;
                }
                usleep(10 * 1000);  /* 10ms delay between retries */
            }
        } else if (args->mode == 2) {  /* ORDERED */
            transfer_ordered(from, to, amount);
        } else if (args->mode == 3) {  /* TRYLOCK */
            while (transfer_trylock(from, to, amount) != 0) {
                __sync_fetch_and_add(&stats.retries, 1);
                if (++retry_count > max_retries) {
                    __sync_fetch_and_add(&stats.failed_transfers, 1);
                    break;
                }
                usleep(rand() % 1000000);  /* Random backoff 0-999999 µs */
            }
        }
    }

    return NULL;
}
```

### Running Naive Mode

Compile and run the naive mode. Use the `timeout` command so it does not
hang forever:

``` {.sourceCode .bash}
make
timeout 5 ./bank -m 0 -t 4 -n 100 || echo "DEADLOCK (as expected)"
```

After 5 seconds, the `timeout` command kills the process. The program
hangs because at least two threads have entered a circular wait.

------------------------------------------------------------------------

## Part 2: Timeout-Based Detection

Instead of blocking forever on a lock, we can set a timeout: \"Try to
acquire this lock for up to 100ms. If you cannot get it, give up,
release any locks you hold, and retry the entire transfer.\" This is
**detection and recovery**: we allow deadlock to form momentarily,
detect it via timeout, and recover by backing off.

### Step 1: Implement mutex_trylock_timed() {#implement-mutex-trylock-timed}

Find the `mutex_trylock_timed()` helper in `bank.c`. It is marked with
TODO. Implement it first: it should poll a mutex with
`pthread_mutex_trylock()` every 10ms until either the lock is acquired
or the timeout expires.

**Requirements:**

- Loop until `timeout_ms` milliseconds have elapsed
- Call `pthread_mutex_trylock(lock)` on each iteration; return 0
  immediately if it succeeds
- Sleep 10ms between attempts with `usleep(10 * 1000)`
- Return `ETIMEDOUT` if the timeout expires without acquiring the lock

### Step 2: Implement transfer_timeout() {#your-task-implement-transfer_timeout}

Find the `transfer_timeout()` function in `bank.c`. It is marked with
TODO. Implement the full function yourself.

The scaffold in `bank.c` provides the overall structure and the
successful-transfer path. Fill in the three marked TODOs:

- When the first lock times out: increment `stats.deadlock_detections`
  with `__sync_fetch_and_add(&stats.deadlock_detections, 1)`
- When the second lock times out: release the `from` lock first
  (`pthread_mutex_unlock(&from->lock)`), then increment the detection
  counter

**The critical mistake to avoid:** if the second lock times out,
forgetting to release the first lock leaves `from` locked permanently.

### Testing Part 2

``` {.sourceCode .bash}
make
./bank -m 1 -t 8 -n 100
```

Expected output (approximate):

    === Deadlock Lab ===
    Mode: TIMEOUT (detect deadlock)
    Threads: 8, Transfers per thread: 100

    === Results ===
    Elapsed time: 2.156 seconds
    Successful transfers: 800
    Failed transfers: 0
    Deadlock detections: 47
    Retries: 47
    Throughput: 211.56 transfers/sec

The program completes (no hang), but throughput is low because each
timeout wastes \~100ms of wall-clock time.

------------------------------------------------------------------------

## Part 3: Prevention via Lock Ordering

If every thread always acquires locks in the **same global order**, a
circular wait can never form. For bank accounts, a natural ordering is
by account ID: always lock the lower-numbered account first.

Example:

- Transfer from Account 5 to Account 2: lock Account 2 first, then
  Account 5
- Transfer from Account 2 to Account 5: lock Account 2 first, then
  Account 5
- Both threads acquire in the same order, so no cycle is possible

### Your Task: Implement transfer_ordered()

Find the `transfer_ordered()` function in `bank.c`. It is marked with
TODO. Implement the full function yourself.

**Requirements:**

- Determine which account has the lower ID
- Always lock the lower-ID account first, then the higher-ID account
- Perform the transfer (deduct from source, credit destination; the
  source/destination are still `from` and `to` regardless of lock order)
- Unlock both mutexes (the scaffold already increments
  `stats.successful_transfers` after your code)

**Hints:**

- The function signature is:
  `static void transfer_ordered(account_t *from, account_t *to, int amount)`
- Use `from->id` and `to->id` to determine lock order
- You can use pointer variables like `first` and `second` to refer to
  the accounts in lock-acquisition order, but the actual balance changes
  still apply to `from` and `to`
- The hint block in the scaffold shows you exactly how to set up `first`
  and `second`

### Testing Part 3

``` {.sourceCode .bash}
./bank -m 2 -t 8 -n 10000
```

Expected output (approximate):

    Mode: ORDERED (prevent deadlock)
    Threads: 8, Transfers per thread: 10000

    === Results ===
    Elapsed time: 0.234 seconds
    Successful transfers: 80000
    Failed transfers: 0
    Deadlock detections: 0
    Retries: 0
    Throughput: 341880.34 transfers/sec

Notice: zero deadlock detections, zero retries, and throughput roughly
100x higher than timeout mode.

------------------------------------------------------------------------

## Part 4: Prevention via Try-Lock with Backoff

Instead of blocking on a lock, use `pthread_mutex_trylock()` which
returns immediately if the lock is unavailable. If you cannot acquire
both locks, release whatever you hold and retry after a random delay.

### Your Task: Implement transfer_trylock()

Find the `transfer_trylock()` function in `bank.c`. It is marked with
TODO. Implement the full function yourself.

**Requirements:**

- Use `pthread_mutex_trylock()` (not `pthread_mutex_lock()`) to attempt
  to acquire the `from` lock. If it returns `EBUSY`, return -1
  immediately
- Use `pthread_mutex_trylock()` to attempt to acquire the `to` lock. If
  it returns `EBUSY`, **release the `from` lock first**, then return -1
- If both locks are acquired, perform the transfer and unlock both
- Increment `stats.successful_transfers` and return 0 on success

**Hints:**

- The function signature is:
  `static int transfer_trylock(account_t *from, account_t *to, int amount)`
- `pthread_mutex_trylock()` returns 0 on success and `EBUSY` if the
  mutex is already locked
- The caller (in `worker()`) handles retries with random backoff:
  `usleep(rand() % 1000000)`. You do not need to implement backoff
  inside this function
- The critical pattern: if the second trylock fails, you must unlock the
  first before returning

### Testing Part 4

``` {.sourceCode .bash}
make
./bank -m 3 -t 8 -n 10000
```

Expected output (approximate):

    Mode: TRYLOCK (prevent deadlock)
    Threads: 8, Transfers per thread: 10000

    === Results ===
    Elapsed time: 0.267 seconds
    Successful transfers: 80000
    Failed transfers: 0
    Deadlock detections: 0
    Retries: 2847
    Throughput: 299625.47 transfers/sec

Notice: zero deadlock detections but thousands of retries. Throughput is
high (close to ordered locking) because retries are fast: each failed
trylock takes microseconds, not the 100ms wasted by timeouts.

------------------------------------------------------------------------

## Part 5: Performance Comparison

Run all four modes and compare their behavior:

``` {.sourceCode .bash}
echo "=== NAIVE MODE (will deadlock) ==="
timeout 5 ./bank -m 0 -t 8 -n 100 || echo "(deadlock after 5 seconds)"

echo ""
echo "=== TIMEOUT MODE (detect deadlock) ==="
./bank -m 1 -t 8 -n 1000

echo ""
echo "=== ORDERED MODE (prevent deadlock) ==="
./bank -m 2 -t 8 -n 50000

echo ""
echo "=== TRYLOCK MODE (prevent deadlock) ==="
./bank -m 3 -t 8 -n 50000
```

Record the throughput, deadlock detections, and retry counts for each
mode in your `answers.txt`. You should observe something like (it will
vary based on your setup):

    Mode          Throughput       Deadlock Detections    Retries
    ----          ----------       -------------------    -------
    Naive         (hangs)          N/A                    N/A
    Timeout       ~500 tx/s       ~2500                   ~2500
    Ordered       ~750,000 tx/s    0                      0
    Trylock       ~300,000 tx/s    0                      ~20

Verify that `verify_total()` confirms money conservation in all modes
that complete.

------------------------------------------------------------------------

## Questions

Answer the following five questions in your `answers.txt` file. Each
answer should be a thoughtful paragraph (3-5 sentences).

**Question 1**: The four necessary conditions for deadlock are mutual
exclusion, hold-and-wait, no preemption, and circular wait. For each of
the three strategies you implemented (timeout, ordered locking,
try-lock), identify which condition it breaks and explain the mechanism.

**Question 2**: Ordered locking achieves much higher throughput than
timeout-based detection. Explain why timeout detection is so much
slower, considering what happens during the timeout period (wasted CPU
time, delayed progress, retry overhead).

**Question 3**: In a distributed system (e.g., microservices making RPC
calls that acquire remote locks), ordered locking may be impossible
because the set of resources is not known in advance. Explain why, and
describe how timeout-based detection becomes the practical choice
despite its performance cost.

**Question 4**: The try-lock approach uses random backoff to avoid
\"thundering herd\" collisions. Explain what would happen with a fixed
delay instead, and why exponential backoff (doubling the delay each
retry) might be even better than uniform random backoff.

**Question 5**: Real databases use a \"wait-for graph\" to detect
deadlock cycles, then abort the youngest transaction to break the cycle.
Compare this to the timeout approach: what are the advantages of
graph-based detection in terms of precision, and what are the costs in
terms of bookkeeping overhead?

------------------------------------------------------------------------

## Submission

Submit the following files:

- `bank.c`: your completed implementation with all three transfer
  strategies
- `answers.txt`: your answers to the five questions above
