# Chapter 5: Optimizing Program Performance

## Scope

This chapter explains how to make programs faster without changing their meaning. The emphasis is not on clever micro-tricks first, but on a disciplined process: write correct code, choose good algorithms and data structures, understand what compilers can optimize, measure performance, then remove bottlenecks.

The running examples focus on C programs, loops, compiler optimization, modern processor pipelines, instruction-level parallelism, memory behavior, and profiling.

## Key ideas

### 1. Correctness comes before speed

A fast program that returns the wrong result is useless. Optimization should preserve program behavior and keep the code understandable enough to review and maintain.

Good order of work:

```text
1. Make it correct.
2. Make it clear.
3. Measure where time is spent.
4. Optimize the important paths.
5. Re-measure.
```

### 2. Compilers are powerful but conservative

Optimizing compilers can remove redundant work, simplify expressions, use registers, inline functions, and transform loops. However, C features such as pointers, aliasing, casts, and side effects limit what the compiler can safely assume.

Example aliasing issue:

```c
void twiddle1(long *xp, long *yp) {
    *xp += *yp;
    *xp += *yp;
}

void twiddle2(long *xp, long *yp) {
    *xp += 2 * *yp;
}
```

These look equivalent, but if `xp == yp`, they produce different results. The compiler must be careful unless it can prove the pointers do not alias.

### 3. Express performance with cycles per element (CPE)

For loops that process arrays, the book often measures performance as cycles per element. This makes it easier to compare versions independent of input size.

```text
CPE = CPU cycles / number of processed elements
```

Example: if processing 1,000,000 elements takes 4,000,000 cycles, performance is about `4.0 CPE`.

### 4. Remove loop inefficiencies

A common performance bug is doing the same work repeatedly inside a loop.

Less efficient:

```c
for (long i = 0; i < strlen(s); i++) {
    /* use s[i] */
}
```

Better:

```c
long n = strlen(s);
for (long i = 0; i < n; i++) {
    /* use s[i] */
}
```

If `strlen` scans the whole string each time, the first version can turn a linear loop into quadratic work.

### 5. Reduce expensive procedure calls in hot loops

Function calls are not always expensive, and compilers may inline them, but calls inside critical loops can block optimization or add overhead if the compiler cannot see through them.

Example idea: if a loop repeatedly calls `get_vec_element(v, i, &val)`, it may be faster to expose the array pointer and length once, then operate directly in the loop when doing so is safe.

### 6. Eliminate unnecessary memory references

Memory operations are usually more expensive than register operations, and repeated loads/stores can block optimization.

Less efficient pattern:

```c
for (long i = 0; i < n; i++) {
    *dest = *dest + a[i];
}
```

Better pattern:

```c
long acc = *dest;
for (long i = 0; i < n; i++) {
    acc += a[i];
}
*dest = acc;
```

The second version keeps the accumulation in a register and writes memory once.

### 7. Modern processors exploit instruction-level parallelism

Modern CPUs do not simply execute one instruction at a time from start to finish. They use pipelining, multiple functional units, branch prediction, out-of-order execution, and register renaming to overlap independent operations.

Important performance concepts:

```text
latency      minimum time for one operation's result to be available
throughput   rate at which operations can be started/completed
critical path longest dependency chain limiting total speed
```

Example: if every iteration depends on the previous accumulator value, the loop may be limited by operation latency even if the CPU has multiple execution units.

### 8. Loop unrolling reduces overhead and exposes parallelism

Loop unrolling processes multiple elements per iteration. It reduces loop-control overhead and can reveal independent operations to the processor.

Basic loop:

```c
for (long i = 0; i < n; i++) {
    acc += a[i];
}
```

Unrolled by 2:

```c
long i;
for (i = 0; i + 1 < n; i += 2) {
    acc += a[i] + a[i+1];
}
for (; i < n; i++) {
    acc += a[i];
}
```

Unrolling helps most when it reduces overhead or allows multiple independent operations. It can hurt if it increases code size or register pressure too much.

### 9. Multiple accumulators can break dependency chains

A single accumulator creates a serial dependency:

```text
acc0 depends on previous acc0 every iteration
```

Using multiple accumulators can expose parallelism:

```c
long acc0 = 0;
long acc1 = 0;
for (long i = 0; i + 1 < n; i += 2) {
    acc0 += a[i];
    acc1 += a[i+1];
}
long result = acc0 + acc1;
```

Now the CPU can update `acc0` and `acc1` more independently.

### 10. Reassociation can improve performance but may change results

For integer addition, changing grouping often preserves the mathematical result except for overflow behavior in C details. For floating point, reassociation can change rounding results.

Example:

```c
/* left-associated */
acc = ((acc + a[i]) + a[i+1]);

/* reassociated */
acc = acc + (a[i] + a[i+1]);
```

The second form may expose more parallelism, but floating-point arithmetic is not truly associative.

### 11. Branch prediction matters

Modern processors predict branch outcomes to keep pipelines full. Predictable branches are cheap; unpredictable branches can be expensive because mispredictions flush work from the pipeline.

Example: processing sorted data may run faster than random data if a branch becomes easier to predict.

```c
if (a[i] > threshold) {
    count++;
}
```

If the condition alternates randomly, prediction is harder.

### 12. Memory performance can dominate

Even if arithmetic is optimized, loads and stores can become the bottleneck. Cache locality, memory bandwidth, and load/store dependencies strongly affect performance.

Example of better spatial locality:

```c
for (long i = 0; i < n; i++) {
    sum += a[i];
}
```

This scans contiguous memory, which is cache-friendly. Random pointer chasing is often much slower because each access may miss cache.

### 13. Profiling guides optimization

Do not guess where the program is slow. Use profiling tools to identify hot functions and loops, optimize those, and measure again.

Typical workflow:

```text
compile with profiling support or debug symbols
run representative workload
inspect time-consuming functions
optimize one bottleneck
benchmark again
```

## Mini examples

### Avoid accidental quadratic behavior

```c
/* Bad if length is recomputed by scanning every time */
for (size_t i = 0; i < strlen(s); i++) {
    putchar(s[i]);
}

/* Better */
size_t n = strlen(s);
for (size_t i = 0; i < n; i++) {
    putchar(s[i]);
}
```

### Keep hot values in registers

```c
/* Repeated memory update */
void sum_into(long *dest, long *a, long n) {
    for (long i = 0; i < n; i++)
        *dest += a[i];
}

/* Register accumulator */
void sum_into_fast(long *dest, long *a, long n) {
    long acc = *dest;
    for (long i = 0; i < n; i++)
        acc += a[i];
    *dest = acc;
}
```

### Multiple accumulators

```c
long sum2(long *a, long n) {
    long acc0 = 0, acc1 = 0;
    long i;
    for (i = 0; i + 1 < n; i += 2) {
        acc0 += a[i];
        acc1 += a[i+1];
    }
    for (; i < n; i++)
        acc0 += a[i];
    return acc0 + acc1;
}
```

## What to remember

- Optimize only after correctness and measurement.
- Compilers optimize aggressively, but pointer aliasing and side effects limit them.
- CPE is a useful metric for loops over arrays.
- Move invariant work out of loops.
- Avoid unnecessary memory references in hot paths; use local accumulators when safe.
- Loop unrolling and multiple accumulators can expose instruction-level parallelism.
- Branch misprediction and memory stalls often dominate runtime.
- Profiling is the safest way to find real bottlenecks.
