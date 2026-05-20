# Chapter 7: Linking

## Scope

This chapter explains how separately compiled pieces of code and data are combined into an executable program. Linking can happen at compile time, load time, or run time, and it is what lets large programs be split into many source files and libraries.

The chapter focuses on object files, symbol tables, symbol resolution, relocation, static libraries, shared libraries, position-independent code, and library interpositioning.

## Key ideas

### 1. Linking enables separate compilation

Instead of compiling a whole application as one giant source file, we can compile files separately:

```bash
gcc -c main.c      # produces main.o
gcc -c sum.c       # produces sum.o
gcc -o prog main.o sum.o
```

The linker combines `main.o`, `sum.o`, and needed library code into an executable.

Why this matters:

```text
- smaller modules are easier to maintain
- only changed files need recompilation
- libraries can be reused across programs
- unresolved references are caught before execution or at load/run time
```

### 2. Compiler drivers hide the build pipeline

A command such as:

```bash
gcc -o hello hello.c
```

usually invokes multiple tools behind the scenes:

```text
preprocessor -> compiler -> assembler -> linker
```

The linker receives relocatable object files and libraries, resolves symbol references, relocates code/data, and emits an executable object file.

### 3. Object files store code, data, symbols, and relocation information

Common object-file forms:

```text
relocatable object file  (.o)  can be linked with others
executable object file         can be loaded and run
shared object file       (.so) can be loaded/linked dynamically
```

On Linux/x86-64, object files use ELF format.

Typical ELF sections:

```text
.text      machine code
.rodata    read-only constants, string literals
.data      initialized global/static variables
.bss       uninitialized global/static variables
.symtab    symbol table
.rel.*     relocation entries
.debug     debug info, if compiled with debug flags
```

### 4. Symbols are names the linker reasons about

Linkers work with symbols, not C variables in the source-level sense.

Main symbol categories:

```text
global symbols defined by this module     e.g. non-static function f
external symbols referenced by this module e.g. printf
local symbols defined by this module      e.g. static helper function
```

Example:

```c
int global = 1;          // global symbol
static int hidden = 2;   // local symbol inside object file
extern int other;        // external reference

static void helper(void) {}
void f(void) { helper(); }
```

### 5. Static variables are not the same as local stack variables

A C `static` variable inside a function has storage in the object file, not on the stack. It has local symbol visibility but persistent lifetime.

```c
int counter(void) {
    static int x = 0;
    return ++x;
}
```

The linker may create a local symbol for `x`, even though source code outside the function cannot name it.

### 6. Symbol resolution matches references to definitions

The linker resolves each symbol reference to exactly one symbol definition.

Common problems:

```text
undefined reference     no definition was found
multiple definition     more than one strong definition exists
type/size mismatch      declarations disagree across files
library order issue     static library searched too early
```

Example undefined reference:

```bash
gcc -o prog main.o
# main.o calls sum, but sum.o was not provided
# => undefined reference to `sum'
```

Fix:

```bash
gcc -o prog main.o sum.o
```

### 7. Strong and weak symbols affect duplicate definitions

At a high level:

```text
functions and initialized globals are strong symbols
uninitialized globals are weak symbols in traditional Unix linking behavior
```

Resolution rules:

```text
- multiple strong definitions are an error
- one strong + weak definitions selects the strong one
- multiple weak definitions choose one, which can hide bugs
```

This is why global variables should usually be defined in exactly one `.c` file and declared with `extern` in headers.

Bad header pattern:

```c
/* bad.h */
int counter;   // definition in every file that includes it
```

Better:

```c
/* good.h */
extern int counter;

/* one .c file */
int counter = 0;
```

### 8. Static libraries are archives of object files

A static library such as `libm.a` is an archive of `.o` files. The linker copies only the needed object modules into the executable.

Example:

```bash
gcc -o prog main.o libvector.a
```

Library order matters for static linking because the linker scans from left to right:

```bash
# often wrong if main.o needs symbols from libfoo.a but appears after it
gcc -lfoo main.o

# better
gcc main.o -lfoo
```

### 9. Relocation assigns final addresses

Compilers and assemblers do not know final memory addresses for all code and data. Relocation patches object code and data once the linker decides where sections and symbols will live.

Relocation does two main things:

```text
1. merges sections of the same type from input object files
2. patches references so they point to final symbol addresses
```

Example idea: a `call sum` instruction in `main.o` may contain a placeholder offset. The linker replaces it with the correct relative address once `sum` has a final location.

### 10. Executables are loaded into memory by the loader

An executable object file contains enough information for the operating system loader to create a process image.

Typical runtime memory layout includes:

```text
code/text segment
read-only data
initialized data
BSS
heap
shared libraries
stack
kernel virtual memory area
```

When you run:

```bash
./prog
```

the shell asks the kernel to load and execute the program.

### 11. Dynamic linking uses shared libraries

Shared libraries (`.so`) are linked at load time or run time. Code from a shared library can be shared by many processes and updated independently from executables.

Example:

```bash
gcc -o prog main.o -lm
```

The executable records that it needs the math shared library. The dynamic linker/loader maps it when the program starts.

Benefits:

```text
- smaller executables
- shared code in memory
- easier library updates
- plugins and run-time loading via dlopen/dlsym
```

Costs:

```text
- more complex loading
- versioning problems
- some indirection overhead
```

### 12. PIC supports shared libraries

Position-independent code (PIC) can run correctly no matter where it is loaded in virtual memory. Shared libraries need this because different processes may map them at different addresses.

Typical mechanisms include:

```text
GOT  Global Offset Table for global data references
PLT  Procedure Linkage Table for function calls
```

These structures let code refer to symbols indirectly instead of hard-coding absolute addresses.

### 13. Library interpositioning can replace or wrap functions

Interpositioning lets one definition override another. It can be used for debugging, profiling, tracing, or custom memory allocation.

Example idea: wrap `malloc` to log allocations.

```c
void *malloc(size_t size) {
    /* record size, then call the real malloc */
}
```

This can happen at compile time, link time, or run time, but it must be used carefully because replacing library functions can introduce subtle bugs.

### 14. Object-file tools are essential for debugging linking problems

Useful tools:

```bash
objdump -d prog       # disassemble code
readelf -a prog       # inspect ELF headers/sections/symbols
nm prog               # list symbols
ldd prog              # show shared library dependencies
ar t libfoo.a         # list archive members
```

These tools help explain unresolved symbols, duplicate symbols, relocation entries, and shared-library dependencies.

## Mini examples

### Two-file program

```c
/* main.c */
long sum(long *a, long n);

int main(void) {
    long a[2] = {1, 2};
    return (int) sum(a, 2);
}
```

```c
/* sum.c */
long sum(long *a, long n) {
    long s = 0;
    for (long i = 0; i < n; i++)
        s += a[i];
    return s;
}
```

Build:

```bash
gcc -c main.c
gcc -c sum.c
gcc -o prog main.o sum.o
```

`main.o` contains an unresolved reference to `sum`; the linker resolves it using the definition in `sum.o`.

### Header declaration pattern

```c
/* counter.h */
extern int counter;
void inc_counter(void);
```

```c
/* counter.c */
#include "counter.h"
int counter = 0;
void inc_counter(void) { counter++; }
```

This avoids defining `counter` separately in every file that includes the header.

## What to remember

- Linking combines object files and libraries into executable or shared object files.
- Object files contain sections, symbols, and relocation records.
- Symbol resolution connects references to definitions; missing or duplicate symbols cause linker errors.
- Avoid defining global variables in headers; use `extern` declarations and one definition.
- Static libraries are searched left-to-right, so command-line order can matter.
- Relocation patches addresses after the linker decides final layout.
- Dynamic linking uses shared libraries, dynamic loaders, GOT, PLT, and PIC.
- Interpositioning is powerful for tracing/profiling but can be dangerous if misused.
- Tools such as `nm`, `objdump`, `readelf`, `ldd`, and `ar` are the practical way to inspect linking behavior.
