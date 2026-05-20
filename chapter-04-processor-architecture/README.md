# Chapter 4: Processor Architecture

## Scope

This chapter explains how a processor can execute machine instructions. It uses a simplified instruction set, Y86-64, to connect the programmer-visible ISA from Chapter 3 with the hardware structures that fetch, decode, execute, access memory, and write back results.

The main lesson is that an instruction set architecture is an abstraction: software sees sequential instructions and registers, while hardware may use staged datapaths, clocked state, and pipelining to run parts of many instructions at once.

## Key ideas

### 1. ISA separates software behavior from hardware implementation

An instruction set architecture defines what instructions exist, how they are encoded, and what state they affect. It does not require a specific internal implementation.

Example: from the software view, `addq %rax, %rbx` updates `%rbx` after reading `%rax` and `%rbx`. A hardware implementation might split this into fetch, decode, execute, memory, and write-back stages, but it must preserve the same visible result.

### 2. Y86-64 is a simplified x86-64-like ISA

Y86-64 keeps the ideas needed to study processor design while removing many complications from real x86-64. It has programmer-visible state such as registers, condition codes, program counter, memory, and status code.

Common instruction classes include:

```text
rrmovq rA, rB    copy register to register
irmovq V, rB     copy immediate value to register
rmmovq rA, D(rB) store register to memory
mrmovq D(rB), rA load memory to register
OPq rA, rB       integer operation
jXX Dest         conditional jump
call Dest        procedure call
ret              procedure return
pushq rA         push register
popq rA          pop register
```

Instruction bytes are designed so the processor can extract fields such as opcode, function code, register IDs, and immediate constants.

### 3. Processor execution can be organized into stages

A sequential processor can execute each instruction through conceptual stages:

```text
Fetch -> Decode -> Execute -> Memory -> Write back -> PC update
```

Example for `mrmovq D(rB), rA`:

```text
Fetch:      read instruction bytes and constant D
Decode:     read base register rB
Execute:    compute address valE = R[rB] + D
Memory:     read value valM = M[valE]
Write back: write R[rA] = valM
PC update:  move to next instruction address
```

This staged view is useful because many instructions share the same hardware blocks in different ways.

### 4. HCL describes hardware control logic

The Hardware Control Language (HCL) describes combinational logic used to choose values and control hardware blocks. It is not a programming language for sequential algorithms; it expresses circuits.

Example idea:

```text
word aluA = [
  icode in { IRRMOVQ, IOPQ } : valA;
  icode in { IIRMOVQ, IRMMOVQ, IMRMOVQ } : valC;
  icode in { ICALL, IPUSHQ } : -8;
  icode in { IRET, IPOPQ } : 8;
];
```

This means the ALU input `aluA` is selected by a multiplexer based on the instruction type.

### 5. Clocked state separates combinational computations

Combinational logic continuously computes outputs from current inputs. Clocked state elements, such as registers and memories, update only at clock boundaries.

This distinction lets a processor compute many intermediate signals during a cycle, then commit architectural state in a controlled way.

### 6. SEQ is a simple sequential Y86-64 processor

The SEQ implementation executes one complete instruction per clock cycle. It is easy to reason about because each instruction goes through all stages before the next begins.

Tradeoff:

```text
Benefit: simple correctness model
Cost: clock period must be long enough for the slowest instruction path
```

For example, an instruction that fetches bytes, reads registers, computes an address, reads memory, writes a register, and updates the PC determines the minimum safe clock period even if simpler instructions need less work.

### 7. Pipelining improves throughput

Pipelining inserts registers between stages so multiple instructions can be active at the same time.

Example five-stage pipeline:

```text
Cycle 1: I1 Fetch
Cycle 2: I1 Decode   | I2 Fetch
Cycle 3: I1 Execute  | I2 Decode   | I3 Fetch
Cycle 4: I1 Memory   | I2 Execute  | I3 Decode   | I4 Fetch
Cycle 5: I1 Writeback| I2 Memory   | I3 Execute  | I4 Decode | I5 Fetch
```

Pipelining improves instruction throughput, but it does not make a single instruction finish faster. It also introduces control complexity.

### 8. Pipeline hazards must be handled explicitly

A pipeline can compute the wrong result unless hazards are detected and handled.

#### Data hazard

A later instruction needs a value that an earlier instruction has not written back yet.

```asm
irmovq $10, %rax
addq   %rax, %rbx
```

The `addq` needs `%rax`. Hardware can solve this with forwarding, stalling, or both.

#### Load/use hazard

A load result may not be available soon enough for the immediately following instruction.

```asm
mrmovq 0(%rsp), %rax
addq   %rax, %rbx
```

This often requires inserting a bubble so the consuming instruction waits.

#### Control hazard

For conditional jumps, the processor may not know the correct next PC until the branch condition is evaluated.

```asm
subq %rax, %rbx
jne  target
```

A pipeline may predict the branch direction, then cancel incorrectly fetched instructions if the prediction was wrong.

### 9. Exceptions are harder in pipelined processors

Sequential execution makes exceptions easy to locate. In a pipeline, multiple instructions are in flight, so the processor must report the correct offending instruction and avoid committing later instructions incorrectly.

The general rule is to preserve the same visible behavior that a sequential ISA model would have produced.

## Mini examples

### Manual stage trace

For this Y86-64-like instruction:

```asm
irmovq $5, %rax
```

A staged interpretation is:

```text
Fetch:      read instruction, register byte, and constant 5
Decode:     no source register needed
Execute:    pass constant through ALU as valE = 5
Memory:     no memory access
Write back: R[%rax] = 5
PC update:  PC = next instruction
```

### Pipeline intuition

If five stages each take roughly the same time, the ideal pipeline can approach one completed instruction per cycle after it fills. Real pipelines fall short because of register overhead, uneven stage delays, hazards, branch mispredictions, and exceptions.

## What to remember

- ISA defines visible behavior; microarchitecture defines how hardware implements it.
- Y86-64 is a teaching ISA used to understand x86-64-style processor design without full x86 complexity.
- SEQ is simple but slow because every instruction uses one long clock cycle.
- Pipelining improves throughput by overlapping instruction stages.
- Hazards and exceptions are the main correctness challenges in pipelined processors.
- Good hardware design preserves the illusion of sequential execution while exploiting parallelism internally.
