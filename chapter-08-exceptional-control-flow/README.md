# Chapter 8: Exceptional Control Flow

## Scope

This chapter explains how control flow changes for reasons beyond ordinary instruction sequencing, branches, calls, and returns. These abrupt changes are called exceptional control flow (ECF), and they appear at every level of the system: hardware exceptions, operating-system process switches, Unix signals, and user-level nonlocal jumps.

The chapter connects low-level processor events with practical Unix programming: processes, `fork`, `execve`, `waitpid`, signals, signal handlers, and `setjmp`/`longjmp`.

## Key ideas

### 1. Control flow is the sequence of instruction addresses

A processor normally executes a sequence of instructions by moving the program counter from one address to the next.

```text
I0 -> I1 -> I2 -> I3 -> ...
```

Ordinary control transfers include:

```text
jumps
calls
returns
```

Exceptional control flow handles events that are not simply ordinary program branches, such as timer interrupts, page faults, system calls, process termination, or signals.

### 2. Exceptions transfer control to the operating system

An exception is an abrupt change in control flow triggered by an event. The processor transfers control to an exception handler in the operating system kernel.

General pattern:

```text
user code runs
an event occurs
processor jumps to kernel exception handler
handler runs
control may return to current instruction, next instruction, or terminate the program
```

### 3. Exception classes: interrupts, traps, faults, aborts

Main exception categories:

```text
interrupt   asynchronous event from outside the processor
trap        intentional exception, often for system calls
fault       potentially recoverable error
abort       unrecoverable error
```

Examples:

```text
timer interrupt          lets OS regain control and schedule processes
system call trap         user program requests kernel service
page fault               missing virtual-memory page is loaded or program segfaults
machine check abort      serious hardware error
```

### 4. System calls are controlled entries into the kernel

User programs cannot directly perform privileged operations such as reading disk blocks, creating processes, or changing page tables. They request kernel services through system calls.

Example:

```c
ssize_t n = read(fd, buf, sizeof(buf));
```

At the machine level, this triggers a trap into the kernel. The kernel validates arguments, performs the operation, and returns a result to user mode.

### 5. A process is an executing program with its own context

A process gives each program two important illusions:

```text
logical control flow    it seems to have the CPU to itself
private address space   it seems to have its own memory
```

The OS maintains this illusion using context switches and virtual memory.

Process context includes:

```text
register values
program counter
stack pointer
condition codes
virtual address space
open files and kernel data structures
```

### 6. Context switches implement multitasking

The kernel can suspend one process and resume another by saving and restoring context.

```text
process A running
interrupt or system call enters kernel
kernel saves A context
kernel restores B context
process B resumes
```

This is why concurrent processes appear to run at the same time even on limited CPU cores.

### 7. `fork` creates a child process

`fork` creates a new child process that is almost identical to the parent. Both continue executing after the `fork` call, but they receive different return values.

```c
pid_t pid = fork();
if (pid == 0) {
    /* child */
} else {
    /* parent */
}
```

Important properties:

```text
child gets a separate virtual address space
child inherits copies of open file descriptors
parent and child run concurrently
execution order is nondeterministic
```

### 8. `execve` loads and runs a new program

`execve` replaces the current process image with a new program. It does not create a new process by itself.

Common shell-like pattern:

```c
if (fork() == 0) {
    execve("/bin/ls", argv, environ);
    _exit(1);  /* only if execve fails */
}
```

The child keeps the same PID but gets new code, data, heap, stack, and entry point.

### 9. Parent processes must reap children

When a child terminates, it becomes a zombie until the parent reaps it with `wait` or `waitpid`.

```c
int status;
pid_t pid = waitpid(-1, &status, 0);
```

If a parent never reaps children, zombie processes accumulate and consume kernel resources.

### 10. System call errors should be checked

Unix system calls often return `-1` on error and set `errno`.

```c
pid_t pid = fork();
if (pid < 0) {
    perror("fork");
    exit(1);
}
```

Robust systems code checks errors consistently because failures can come from resource limits, invalid arguments, permissions, interrupted calls, and more.

### 11. Signals are small asynchronous notifications

A signal notifies a process that an event occurred.

Examples:

```text
SIGINT   interrupt from keyboard, often Ctrl-C
SIGKILL  terminate, cannot be caught or ignored
SIGSEGV  invalid memory reference
SIGCHLD  child process stopped or terminated
SIGALRM  alarm timer expired
```

A process can:

```text
ignore a signal
terminate because of a signal
catch a signal with a handler
block/unblock a signal temporarily
```

### 12. Signal handlers run concurrently with main code

A signal handler can interrupt normal execution and run between almost any two instructions. This makes signal programming subtle.

Example handler:

```c
volatile sig_atomic_t got_sigint = 0;

void handler(int sig) {
    got_sigint = 1;
}
```

Use only async-signal-safe operations inside handlers. Avoid calling non-reentrant functions such as `printf` from signal handlers.

### 13. Blocking signals prevents race windows

Signals can arrive at inconvenient times. Temporarily blocking signals can protect critical sections.

Typical idea when managing child processes:

```text
block SIGCHLD
fork child
add child PID to job list
unblock SIGCHLD
```

Without blocking, the child might terminate and the handler might run before the parent records the child in its job list.

### 14. `sigsuspend` helps wait for signals correctly

A common bug is checking a flag and then going to sleep while a signal arrives between those two actions. `sigsuspend` atomically changes the signal mask and suspends the process until a signal arrives.

Conceptual pattern:

```text
block signal
while condition not true:
    sigsuspend(temporary mask)
restore old mask
```

This avoids missed wakeups.

### 15. Nonlocal jumps move control across stack frames

C provides `setjmp` and `longjmp` for nonlocal jumps.

```c
jmp_buf env;

if (setjmp(env) == 0) {
    /* initial path */
} else {
    /* returned via longjmp */
}
```

A later call to:

```c
longjmp(env, 1);
```

jumps back to the saved point. This can implement error recovery or escape from deeply nested calls, but it must be used carefully because it bypasses normal function returns.

## Mini examples

### Fork return values

```c
pid_t pid = fork();
if (pid == 0) {
    printf("child
");
} else {
    printf("parent
");
}
```

Both processes run after `fork`. The output order is not guaranteed.

### Reaping all terminated children

```c
void sigchld_handler(int sig) {
    int olderrno = errno;
    while (waitpid(-1, NULL, WNOHANG) > 0)
        ;
    errno = olderrno;
}
```

The loop is important because one `SIGCHLD` may correspond to multiple terminated children.

### Avoiding unsafe handler work

Better signal style:

```c
volatile sig_atomic_t stop = 0;

void handler(int sig) {
    stop = 1;
}

int main(void) {
    while (!stop) {
        /* normal work */
    }
}
```

The handler records the event; normal code does the complex work.

## What to remember

- Exceptional control flow handles abrupt control transfers caused by system events.
- Exceptions include interrupts, traps, faults, and aborts.
- System calls are traps from user code into the kernel.
- Processes provide logical control flow and private address spaces.
- `fork` creates a child; `execve` replaces the current program image.
- Parents must reap terminated children to avoid zombies.
- Signal handlers are concurrent with main code and must be minimal and async-signal-safe.
- Block signals around critical sections to avoid races.
- `sigsuspend` avoids missed-signal races when waiting.
- `setjmp`/`longjmp` implement nonlocal control transfers but should be used carefully.
