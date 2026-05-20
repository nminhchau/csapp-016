# Chapter 6: The Memory Hierarchy

## Scope

This chapter replaces the simple “memory is a flat array with uniform access time” model with the real model used by modern systems: a hierarchy of storage devices with different speeds, capacities, and costs.

The central idea is that fast storage is small and expensive, slow storage is large and cheap, and systems use caching plus program locality to make the whole hierarchy appear both large and fast most of the time.

## Key ideas

### 1. Memory is a hierarchy, not one uniform device

A modern system stores data across multiple levels:

```text
CPU registers
L1 cache
L2 cache
L3 cache
Main memory (DRAM)
Local disk / SSD
Remote storage over network
```

Upper levels are faster, smaller, and more expensive per byte. Lower levels are slower, larger, and cheaper.

The hierarchy works because programs usually reuse nearby or recently used data, so most accesses can be served by upper levels.

### 2. Storage technologies have different tradeoffs

Different devices have very different latency, bandwidth, capacity, and cost.

Common categories:

```text
SRAM   very fast, used for caches, expensive, low density
DRAM   slower than SRAM, used for main memory, cheaper, higher density
SSD    persistent flash storage, faster than disks for random access
Disk   magnetic storage, large and cheap, slow mechanical seeks
```

A performance-sensitive program should care not only about number of operations, but also where its data lives.

### 3. Locality makes caching effective

Programs with good locality reuse data and instructions in predictable ways.

#### Temporal locality

If a program uses a memory location now, it is likely to use the same location again soon.

Example:

```c
for (int i = 0; i < n; i++) {
    sum += a[i];
}
```

The variable `sum` has strong temporal locality because it is reused every iteration.

#### Spatial locality

If a program uses one memory location, it is likely to use nearby locations soon.

Example:

```c
for (int i = 0; i < n; i++) {
    sum += a[i];
}
```

The array scan has strong spatial locality because `a[i]`, `a[i+1]`, `a[i+2]`, ... are contiguous in memory.

### 4. Cache hit and miss behavior dominates performance

A cache stores copies of blocks from a lower level. When the CPU requests data:

```text
cache hit   requested data is found in the cache
cache miss  data must be fetched from a lower, slower level
```

Important terms:

```text
block/cache line   fixed-size chunk moved between cache and memory
hit rate           fraction of accesses served by cache
miss rate          fraction of accesses not found in cache
miss penalty       extra time to fetch from lower level
```

Even a small miss rate can be costly if the miss penalty is large.

### 5. Generic cache organization: sets, lines, and tags

A cache is typically divided into sets. Each set contains one or more cache lines. Each line stores a valid bit, tag, and block data.

Address bits are split conceptually:

```text
[ tag bits | set index bits | block offset bits ]
```

- block offset selects a byte within a cache block
- set index selects which set to look in
- tag identifies which memory block is currently stored there

### 6. Direct-mapped caches are simple but can conflict

In a direct-mapped cache, each memory block maps to exactly one cache line.

```text
set = block_number mod number_of_sets
```

Benefit: simple and fast.

Cost: two frequently used blocks that map to the same set repeatedly evict each other, causing conflict misses.

### 7. Set-associative caches reduce conflicts

In an E-way set-associative cache, each set has E possible lines. A memory block maps to one set, but can occupy any line in that set.

```text
E = 1       direct mapped
E > 1       set associative
one set     fully associative
```

More associativity reduces conflict misses but increases hardware complexity and lookup cost.

### 8. Write behavior requires policy choices

For stores, caches need policies:

```text
write-through   update cache and lower memory immediately
write-back      update cache first, write lower memory later when evicted
write-allocate  on store miss, load block into cache then write
no-write-allocate write directly to lower level on store miss
```

These policies trade simplicity, bandwidth, and performance.

### 9. Cache-friendly code uses locality deliberately

Two practical rules:

```text
1. Reuse data that is already in cache.
2. Access memory with stride 1 when possible.
```

Good spatial locality:

```c
for (int i = 0; i < n; i++) {
    sum += a[i];
}
```

Poorer locality with large stride:

```c
for (int i = 0; i < n; i += 16) {
    sum += a[i];
}
```

The second loop may use only a small part of each loaded cache line.

### 10. Row-major layout matters for C arrays

C stores multidimensional arrays in row-major order. Consecutive elements in the last dimension are adjacent.

Cache-friendly traversal:

```c
int sum = 0;
for (int i = 0; i < N; i++) {
    for (int j = 0; j < M; j++) {
        sum += a[i][j];
    }
}
```

Less cache-friendly traversal:

```c
int sum = 0;
for (int j = 0; j < M; j++) {
    for (int i = 0; i < N; i++) {
        sum += a[i][j];
    }
}
```

The first version walks contiguous memory; the second jumps by row size each inner-loop iteration.

### 11. The memory mountain shows locality effects

The book uses the “memory mountain” to show how read throughput depends on working-set size and stride.

General pattern:

```text
small working set + small stride  -> high throughput
large working set + large stride  -> low throughput
```

This visualizes why cache-conscious layout and traversal can produce large performance differences without changing the algorithmic complexity.

## Mini examples

### Estimating cache address fields

Suppose a cache has:

```text
S = 64 sets
B = 64 bytes per block
```

Then:

```text
set index bits = log2(64) = 6
block offset bits = log2(64) = 6
```

The remaining high-order address bits form the tag.

### Matrix traversal

For a C matrix `int a[1024][1024]`, this order is usually faster:

```c
for (int i = 0; i < 1024; i++)
    for (int j = 0; j < 1024; j++)
        total += a[i][j];
```

because the inner loop reads adjacent `int` values.

### Avoid wasting cache lines

If a cache line is 64 bytes and `int` is 4 bytes, one line holds 16 integers. A stride-1 scan can use all 16 values from a loaded line. A stride-16 scan may use only one integer from each line.

## What to remember

- Real memory has levels; access time is not uniform.
- Caches work because programs usually have temporal and spatial locality.
- Cache misses are expensive; good locality reduces misses.
- Caches are organized using blocks, sets, lines, tags, valid bits, and offsets.
- Direct-mapped caches are simple but prone to conflict misses.
- Set associativity reduces conflicts at some hardware cost.
- Write policies affect correctness, bandwidth, and performance.
- For C arrays, row-major traversal is usually cache-friendly.
- Stride, working-set size, and access order can matter as much as instruction count.
