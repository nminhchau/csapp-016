# Chapter 9: Virtual Memory

## Scope

This chapter explains virtual memory (VM), the abstraction that gives each process a large, uniform, private address space even though physical memory is shared among many processes.

Virtual memory combines hardware address translation, page tables, caches, disk files, exceptions, and kernel software. It is used for caching, memory management, protection, memory mapping, dynamic allocation, and detecting or preventing many classes of memory errors.

## Key ideas

### 1. Virtual memory gives each process a private address space

Without virtual memory, processes would directly share physical memory, making memory management and protection difficult. With VM, each process uses virtual addresses, and hardware plus the OS translate them to physical addresses.

```text
process uses virtual address -> MMU translates -> physical address in DRAM
```

Benefits:

```text
- each process sees a clean, uniform address space
- processes are isolated from each other
- physical memory can be used as a cache for data stored on disk
- programs can use more virtual address space than available DRAM
```

### 2. Physical addressing vs virtual addressing

Physical addressing:

```text
CPU address directly names a byte in physical memory
```

Virtual addressing:

```text
CPU generates a virtual address
MMU translates it to a physical address
memory is accessed using the physical address
```

Modern general-purpose systems use virtual addressing for ordinary process execution.

### 3. Address spaces are sets of possible addresses

An address space is an ordered set of nonnegative integer addresses.

```text
{0, 1, 2, ..., N-1}
```

A virtual address space is the set of addresses visible to a process. A physical address space is the set of addresses in actual main memory.

### 4. VM uses DRAM as a cache for disk

Virtual memory treats physical memory as a cache for pages from a larger virtual address space stored on disk.

```text
virtual page (VP)    fixed-size block in virtual memory
physical page (PP)   fixed-size block in physical memory
page table           maps virtual pages to physical pages or disk locations
```

If a virtual page is currently in physical memory, accessing it is a page hit. If not, the access triggers a page fault.

### 5. Page faults are recoverable exceptions

When a process accesses a virtual page that is not resident in DRAM, the hardware triggers a page fault exception. The kernel page fault handler can load the page from disk into memory, update the page table, and restart the faulting instruction.

Conceptual flow:

```text
load virtual address
page table says page not present
page fault exception
kernel selects victim page if needed
kernel loads requested page from disk
kernel updates page table
instruction restarts
```

This works well because programs tend to have locality.

### 6. Page tables map virtual pages to physical pages

Each process has its own page table. A page table entry (PTE) contains metadata such as:

```text
valid/present bit
physical page number or disk location
permission bits: read/write/execute/user
dirty bit
reference bit
```

The virtual address is split into:

```text
[ virtual page number | virtual page offset ]
```

The offset is copied unchanged into the physical address; the page number is translated.

### 7. VM simplifies memory management

Because each process has a similar virtual layout, the OS can manage memory more cleanly.

Typical process virtual address layout:

```text
low addresses
  code/text
  read-only data
  initialized data
  bss
  heap grows upward
  memory-mapped region
  stack grows downward
high addresses
```

Different virtual pages can map to different physical pages, shared pages, files, or no current physical page.

### 8. VM provides memory protection

Permission bits in page table entries let the OS enforce protection.

Examples:

```text
user code cannot write read-only text pages
one process cannot access another process's private pages
user process cannot access kernel-only pages
execute permission can prevent running data as code
```

Invalid access triggers a fault, often reported to the process as `SIGSEGV`.

### 9. Address translation is hardware-assisted

The Memory Management Unit (MMU) translates virtual addresses using page tables. A basic translation:

```text
VPN = virtual address page number
VPO = virtual address page offset
PTE = page_table[VPN]
PPN = PTE physical page number
physical address = PPN || VPO
```

If the PTE is invalid or permissions fail, the MMU triggers an exception.

### 10. TLB speeds up address translation

Page-table lookup can require memory accesses, so processors use a Translation Lookaside Buffer (TLB): a small cache of recent virtual-page to physical-page translations.

```text
TLB hit   translation found quickly
TLB miss  hardware or OS walks page table
```

Good locality helps the TLB just as it helps ordinary caches.

### 11. Multi-level page tables save space

A flat page table for a large virtual address space would be huge. Multi-level page tables allocate lower-level tables only for regions that are actually used.

Example idea:

```text
virtual address -> level 1 index -> level 2 index -> ... -> page offset
```

This is space-efficient because many virtual address ranges are unused.

### 12. Memory mapping connects virtual pages to files or anonymous objects

Memory mapping maps a region of virtual memory to an object, often a file.

```c
void *p = mmap(NULL, length, PROT_READ, MAP_PRIVATE, fd, 0);
```

Uses:

```text
load executable segments
load shared libraries
map files into memory
implement shared memory
allocate anonymous memory
```

Shared libraries and `fork` benefit from VM mechanisms such as shared pages and copy-on-write.

### 13. `fork` and `execve` rely on VM

`fork` can create a child efficiently by initially sharing pages between parent and child as copy-on-write. Only when one process writes to a shared page does the kernel copy it.

`execve` discards the old virtual address space and maps in the new program's code, data, heap, stack, and shared libraries.

### 14. Dynamic memory allocation manages the heap

C programs use dynamic memory allocation to request variable-size blocks at run time.

```c
int *a = malloc(n * sizeof(int));
if (a == NULL) {
    /* handle allocation failure */
}
free(a);
```

The allocator manages blocks within the heap and obtains more heap memory from the kernel when needed.

### 15. Allocators balance throughput and memory utilization

Allocator goals:

```text
throughput      allocations/frees per unit time
utilization     fraction of heap used for payload rather than overhead/waste
```

Key implementation issues:

```text
free block organization
placement policy: first fit, next fit, best fit
splitting large free blocks
coalescing adjacent free blocks
metadata/header layout
```

### 16. Fragmentation wastes memory

Internal fragmentation: allocated block is larger than requested payload.

```text
request 13 bytes, allocator gives 16 or 32 bytes
```

External fragmentation: free memory exists but is split into pieces too small for a request.

```text
free blocks: 8, 16, 8 bytes
request: 24 bytes
```

### 17. Garbage collection finds unreachable heap objects

Garbage collectors automatically reclaim unreachable heap objects. The chapter introduces mark-and-sweep collection and conservative collection for C-like environments.

Basic mark-and-sweep:

```text
start from roots: registers, stack, globals
mark all reachable heap blocks
sweep heap and free unmarked blocks
```

C makes precise GC hard because raw words may look like pointers.

### 18. C memory bugs are common and dangerous

Common bugs:

```text
dereferencing bad pointers
reading uninitialized memory
stack buffer overflow
off-by-one errors
confusing pointer size with object size
wrong pointer arithmetic
returning pointer to stack variable
use-after-free
memory leaks
```

Example use-after-free:

```c
int *p = malloc(sizeof(int));
free(p);
*p = 1;   /* bug */
```

Example leak:

```c
void f(void) {
    int *p = malloc(100 * sizeof(int));
    return;  /* bug: p was never freed */
}
```

## Mini examples

### Virtual-to-physical address split

If page size is 4096 bytes:

```text
page offset bits = log2(4096) = 12
```

A virtual address is conceptually split as:

```text
[ virtual page number | 12-bit page offset ]
```

The page offset does not change during translation.

### Copy-on-write intuition

After `fork`:

```text
parent virtual page -> same physical page
child virtual page  -> same physical page
both mappings are read-only copy-on-write
```

If the child writes, the kernel creates a private copy for the child and updates its page table.

### Safer allocation size pattern

```c
int *a = malloc(n * sizeof *a);
```

Using `sizeof *a` avoids repeating the type name and reduces the chance of allocating the wrong size if the pointer type changes.

## What to remember

- Virtual memory gives each process a private, uniform virtual address space.
- Physical memory acts as a cache for virtual pages stored on disk.
- Page tables map virtual pages to physical pages and store protection metadata.
- Page faults are exceptions that the OS can handle by loading pages or reporting invalid access.
- Address translation is performed by the MMU and accelerated by the TLB.
- Multi-level page tables reduce memory overhead for sparse address spaces.
- `mmap`, shared libraries, `fork`, and `execve` all depend heavily on VM.
- Dynamic allocators manage heap blocks and must balance speed with space utilization.
- Fragmentation is a central allocator problem.
- C memory bugs such as buffer overflows, use-after-free, and leaks are frequent and serious.
