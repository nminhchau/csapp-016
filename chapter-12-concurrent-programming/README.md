# Chapter 12: Concurrent Programming

## Scope

This chapter explains application-level concurrency: programs with multiple logical control flows whose executions overlap in time. Concurrency is useful for handling slow I/O, interacting with users, serving multiple clients, and exploiting multiple CPU cores.

The chapter compares three main approaches: processes, I/O multiplexing, and threads. It then focuses on shared variables, synchronization with semaphores, using threads for parallelism, and common concurrency bugs such as races and deadlocks.

## Key ideas

### 1. Concurrent flows overlap in time

Two logical control flows are concurrent if their executions overlap.

```text
Flow A:  |---- running/waiting ----|
Flow B:        |---- running/waiting ----|
```

Concurrency does not necessarily mean parallelism. Parallelism means flows are executing at the same instant on different cores. Concurrency only requires overlapping lifetimes.

### 2. Application-level concurrency has practical uses

Common reasons to write concurrent programs:

```text
overlap computation with slow I/O
serve multiple clients at once
keep user interfaces responsive
exploit multiple CPU cores
structure programs as independent tasks
```

Network servers are a central example: while one client is waiting for I/O, the server can make progress on another client.

### 3. Processes provide strong isolation

One way to build a concurrent server is to `fork` a child for each client connection.

Conceptual server loop:

```c
while (1) {
    int connfd = accept(listenfd, NULL, NULL);
    if (fork() == 0) {
        close(listenfd);
        handle_client(connfd);
        close(connfd);
        _exit(0);
    }
    close(connfd);
}
```

Advantages:

```text
separate address spaces
one client bug is less likely to corrupt another
simple mental model
```

Disadvantages:

```text
higher process creation/context-switch overhead
harder to share state
must reap children to avoid zombies
```

### 4. I/O multiplexing uses one process to manage many descriptors

I/O multiplexing lets one process wait for events on multiple file descriptors. The classic interface is `select`.

Conceptual idea:

```text
keep a set of active descriptors
wait until one or more are ready
handle only descriptors that are ready
repeat
```

This enables event-driven servers.

Advantages:

```text
single address space
no process/thread creation per client
explicit control over scheduling
```

Disadvantages:

```text
more complex state-machine programming
one slow handler can block all clients
cannot use multiple cores by itself
```

### 5. Threads share one process address space

Threads are logical flows that run in the context of a single process. Each thread has its own thread ID, stack, stack pointer, program counter, and registers, but shares global variables, heap, open files, and process address space with other threads.

```text
per-thread: stack, registers, PC
shared:     code, globals, heap, file descriptors
```

This makes sharing easy, but also makes races possible.

### 6. POSIX threads use `pthread_create` and `pthread_join`

Create a thread:

```c
pthread_t tid;
pthread_create(&tid, NULL, thread_main, arg);
```

Wait for a thread:

```c
pthread_join(tid, NULL);
```

Thread function shape:

```c
void *thread_main(void *arg) {
    /* work */
    return NULL;
}
```

Detached threads do not need to be joined; their resources are reclaimed automatically when they terminate.

### 7. Threaded servers are simple but require synchronization

A thread-per-client server resembles the process-per-client model, but uses threads instead of child processes.

```c
while (1) {
    int *connfdp = malloc(sizeof(int));
    *connfdp = accept(listenfd, NULL, NULL);
    pthread_create(&tid, NULL, thread, connfdp);
}
```

A common detail: pass each thread its own allocated copy of `connfd`; do not pass the address of a loop variable that will change.

### 8. Shared variables require careful reasoning

A variable is shared if multiple threads can access the same memory location and at least one thread writes it.

Examples:

```text
global variables are shared
heap objects can be shared
stack variables are private unless their addresses are shared
```

Bug-prone example:

```c
int counter = 0;

void *thread(void *vargp) {
    counter++;
    return NULL;
}
```

`counter++` is not atomic; it involves load, add, and store.

### 9. Semaphores synchronize access to shared state

A semaphore is a nonnegative integer with atomic operations:

```text
P(s)  wait/decrement; block if s is 0
V(s)  increment; wake one blocked thread if any
```

Binary semaphore for mutual exclusion:

```c
sem_t mutex;
sem_init(&mutex, 0, 1);

P(&mutex);
/* critical section */
V(&mutex);
```

Only one thread can be in the critical section at a time.

### 10. Mutex protects critical sections

A critical section is code that accesses shared state and must not be interleaved with conflicting operations from other threads.

Example safe counter update:

```c
sem_t mutex;
int counter = 0;

void *thread(void *vargp) {
    P(&mutex);
    counter++;
    V(&mutex);
    return NULL;
}
```

The semaphore prevents lost updates.

### 11. Semaphores can also schedule resources

Counting semaphores can represent available resources or producer-consumer slots.

Classic bounded-buffer idea:

```text
mutex protects buffer data structure
slots counts empty slots
items counts filled slots
```

Producer:

```text
P(slots)
P(mutex)
insert item
V(mutex)
V(items)
```

Consumer:

```text
P(items)
P(mutex)
remove item
V(mutex)
V(slots)
```

### 12. Prethreaded servers use a worker pool

Instead of creating one thread per client forever, a server can create a fixed pool of worker threads. The main thread accepts connections and places them in a shared buffer; workers remove connection descriptors and handle clients.

Benefits:

```text
limits thread creation overhead
bounds number of concurrent workers
uses producer-consumer synchronization
```

### 13. Threads can exploit parallelism

For CPU-bound work, threads can divide a task across cores.

Example idea:

```text
split array into N chunks
one worker thread sums each chunk
main thread combines partial sums
```

Parallel speedup depends on workload balance, synchronization overhead, memory bandwidth, and number of cores.

### 14. Thread safety and reentrancy matter

A function is thread-safe if it works correctly when called concurrently by multiple threads.

A function is reentrant if it does not rely on shared mutable state across invocations. Reentrant functions are usually thread-safe, but not all thread-safe functions are reentrant.

Unsafe pattern:

```c
char *format_msg(void) {
    static char buf[100];
    /* writes into shared static buffer */
    return buf;
}
```

Better pattern: caller provides the buffer or the function allocates independent storage.

### 15. Races occur when correctness depends on timing

A race happens when program correctness depends on a particular interleaving of concurrent flows.

Example bug:

```c
for (int i = 0; i < n; i++)
    pthread_create(&tid[i], NULL, thread, &i);
```

All threads receive the address of the same loop variable `i`, which changes while threads are starting. Allocate or store a separate value per thread instead.

### 16. Deadlocks happen when threads wait forever

Deadlock can occur when threads hold resources while waiting for each other.

Example:

```text
Thread 1 holds A, waits for B
Thread 2 holds B, waits for A
```

Prevention strategy:

```text
always acquire locks in the same global order
release locks on every path
keep critical sections small
```

## Mini examples

### Race on a shared counter

Unsafe:

```c
int cnt = 0;

void *worker(void *arg) {
    for (int i = 0; i < 100000; i++)
        cnt++;
    return NULL;
}
```

Safe with a semaphore:

```c
sem_t mutex;
int cnt = 0;

void *worker(void *arg) {
    for (int i = 0; i < 100000; i++) {
        P(&mutex);
        cnt++;
        V(&mutex);
    }
    return NULL;
}
```

This is correct but may be slow because it locks on every increment. A faster design is often to accumulate locally and combine once.

### Safer thread argument passing

```c
for (long i = 0; i < n; i++) {
    long *arg = malloc(sizeof(long));
    *arg = i;
    pthread_create(&tid[i], NULL, worker, arg);
}
```

Each thread receives a stable value instead of the address of a changing loop variable.

### Lock ordering rule

If code needs two locks, always acquire them in the same order:

```c
P(&lock_a);
P(&lock_b);
/* use resources A and B */
V(&lock_b);
V(&lock_a);
```

If every thread follows this order, circular wait is avoided.

## What to remember

- Concurrency means overlapping logical flows; parallelism means simultaneous execution.
- Processes provide isolation but have higher overhead and harder sharing.
- I/O multiplexing uses one process to handle many descriptors with event-driven control.
- Threads share an address space, making sharing easy and races easy.
- `pthread_create`, `pthread_join`, detach, and thread routines are the core POSIX thread tools.
- Shared variables must be protected when concurrent writes or read-write conflicts are possible.
- Semaphores implement mutual exclusion and resource scheduling.
- Producer-consumer patterns use semaphores for `mutex`, `slots`, and `items`.
- Thread safety, reentrancy, races, and deadlocks are central correctness concerns.
- Prefer simple synchronization rules: small critical sections, consistent lock ordering, and minimal shared mutable state.
