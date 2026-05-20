# Chapter 1: A Tour of Computer Systems

## Scope

This chapter introduces computer systems from a programmer's perspective by following the lifetime of a simple `hello` program. The main idea is that source code, compilers, operating systems, hardware, memory, files, and networks all cooperate to run even a tiny program.

## Key ideas

### 1. Information is bits plus context

All information in a system is represented as sequences of bits. The same bit pattern can mean different things depending on context: an integer, a floating-point number, a machine instruction, a character string, or part of an image.

Example: the byte value `0x31` can represent the ASCII character `'1'`, the integer 49, or just one byte inside a larger instruction stream.

### 2. Programs are translated into different forms

A C program does not run directly as source text. It moves through the compilation system:

```text
hello.c -> hello.i -> hello.s -> hello.o -> hello
 source     preprocessed  assembly  relocatable  executable
```

Main stages:

- Preprocessor expands headers and macros.
- Compiler translates C into assembly.
- Assembler translates assembly into relocatable object code.
- Linker combines object files and libraries into an executable.

Example:

```c
#include <stdio.h>

int main(void) {
    printf("hello, world\n");
    return 0;
}
```

The call to `printf` is not defined in `hello.c`; the linker resolves it from the C standard library.

### 3. Understanding compilation helps programmers

Knowing compilation and linking helps with:

- Program performance: simple source changes can produce very different machine code.
- Link-time errors: unresolved symbols, duplicate symbols, static vs. dynamic libraries.
- Security: buffer overflows and stack discipline depend on machine-level behavior.

### 4. Hardware organization

A typical system contains:

- CPU: executes instructions and contains the program counter, register file, and ALU.
- Main memory: temporary storage for program code and data while programs run.
- I/O devices: keyboard, display, disk, network adapter, and other external interfaces.
- Buses: transfer fixed-size chunks of bytes between components.

The CPU repeatedly performs a simple instruction cycle:

1. Read an instruction from memory at the address in the program counter.
2. Decode the instruction bits.
3. Execute the operation.
4. Update the program counter.

Typical low-level operations include load, store, arithmetic/logic operations, and jumps.

### 5. Running the `hello` program

When a user types `./hello` in a shell:

1. The shell reads the command from the keyboard.
2. The shell asks the operating system to load the executable from disk.
3. The executable's code and data are copied into main memory.
4. The CPU executes the program's machine instructions.
5. The string `hello, world\n` is copied to the display device.
6. The program exits and control returns to the shell.

The example shows that a simple program involves the shell, disk, memory, CPU, operating system, and display.

### 6. Caches matter

Systems spend a lot of time moving data. Faster storage is smaller and more expensive; larger storage is slower and cheaper. Caches bridge this gap by keeping recently or frequently used data close to the CPU.

Typical hierarchy:

```text
Registers -> L1 cache -> L2 cache -> L3 cache -> Main memory -> Local disk -> Remote storage
fast/small/expensive                                      slow/large/cheap
```

Example: if the CPU repeatedly reads elements of an array, good locality allows cache lines to serve many accesses without going back to main memory.

### 7. Storage devices form a hierarchy

Each level can act as a cache for the level below it. Main memory caches disk blocks; L1 caches L2 data; local disks can cache remote files.

The programming consequence is that data layout and access order can greatly affect performance.

### 8. The operating system manages hardware

Applications normally do not manipulate hardware directly. They use the operating system, which provides protection and abstractions.

Core abstractions:

- Process: an abstraction for a running program.
- Virtual memory: an abstraction giving each process a private address space.
- File: an abstraction for I/O devices and stored data.

### 9. Processes and context switching

A process gives each program the illusion that it has exclusive use of the CPU, memory, and I/O devices. Multiple processes share the system through context switching.

A context contains the state needed to resume a process, such as the program counter, registers, and memory-management state.

Example: when the shell starts `hello`, the operating system saves the shell's context, runs `hello`, then restores the shell when `hello` exits.

### 10. Threads

A process can contain multiple threads. Threads share the same code and global data but have separate control flows. They are useful for concurrency and can be more efficient than multiple processes.

### 11. Virtual memory

Virtual memory gives each process a private, uniform address space. A simplified Linux process address space contains:

```text
High addresses
Kernel virtual memory
User stack
Shared libraries
Heap
Read/write data
Read-only code and data
Low addresses
```

This abstraction protects processes from each other and lets the operating system manage physical memory and disk transparently.

### 12. Files

A file is a sequence of bytes. Unix-style systems model many I/O devices as files, so applications can use a small set of system calls to work with disks, terminals, and other devices.

### 13. Networks are I/O devices

A network adapter can be viewed as another I/O device. Data can be copied from memory to the network and from the network back to memory.

Example: running `hello` remotely through a telnet-like client sends the command string over the network, executes the program on a remote machine, and sends the output back to the local terminal.

## Important themes

### Amdahl's Law

Amdahl's Law explains the limit of optimizing only one part of a system.

If a fraction `alpha` of execution time is improved by factor `k`, then total speedup is:

```text
S = 1 / ((1 - alpha) + alpha / k)
```

Example: if 60% of runtime is improved by 3x:

```text
S = 1 / (0.4 + 0.6 / 3)
  = 1 / 0.6
  = 1.67x
```

Even a large improvement to one part gives limited total speedup if the rest of the system remains unchanged.

### Concurrency and parallelism

- Concurrency: multiple activities are in progress at the same time conceptually.
- Parallelism: multiple activities execute at the same time physically.

Levels discussed:

- Thread-level concurrency: multiple threads or processes.
- Instruction-level parallelism: a processor executes multiple instructions per cycle using techniques such as pipelining and superscalar execution.
- SIMD parallelism: one instruction operates on multiple data values.

### Abstraction

Computer systems depend on layered abstractions that hide complexity:

- Instruction set architecture abstracts processor hardware.
- Files abstract I/O devices.
- Virtual memory abstracts physical memory.
- Processes abstract CPU and memory execution resources.

## Minimal example to try

```bash
gcc -o hello hello.c
./hello
```

Expected output:

```text
hello, world
```

Questions to connect this example to the chapter:

1. Which tool resolves `printf`?
2. Where is the executable stored before it runs?
3. Which abstraction represents the running program?
4. Why can repeated access to nearby memory be faster?

## Summary

Chapter 1 frames the whole book: programs are not isolated source files, but participants in a layered system involving compilation, linking, instruction execution, memory hierarchy, operating-system abstractions, and networking. A programmer who understands these layers can write faster, more reliable, and more secure programs.
