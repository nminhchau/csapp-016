# Chapter 2: Representing and Manipulating Information

## Scope

This chapter explains how computers encode and operate on data. The central theme is that every value is a sequence of bits, and the meaning of those bits depends on the representation: unsigned integer, signed integer, floating-point number, character data, pointer, or machine instruction.

Understanding these representations helps programmers avoid overflow bugs, write portable code, reason about bit-level operations, and understand why floating-point arithmetic is approximate.

## Key ideas

### 1. Information storage

A machine-level program views memory as a very large array of bytes. Each byte has an address, and programs use addresses to refer to data.

A byte is 8 bits, so it can represent 256 different patterns:

```text
00000000 through 11111111
0x00     through 0xFF
```

Hexadecimal notation is convenient because each hex digit represents exactly 4 bits:

```text
0x2A = 0010 1010
0xF0 = 1111 0000
```

### 2. Words and data sizes

A machine has a word size, which is the nominal size of pointer data. On a 64-bit machine, virtual addresses are typically represented with 64-bit words.

Common C data sizes on many 64-bit Linux systems:

```text
char      1 byte
short     2 bytes
int       4 bytes
long      8 bytes
pointer   8 bytes
float     4 bytes
double    8 bytes
```

Portable programs should not assume every platform uses exactly the same size for every type. Use `sizeof` when the actual size matters.

### 3. Byte ordering

Multi-byte objects must be stored in memory as a sequence of bytes. The order depends on the machine convention.

- Little endian: least significant byte comes first.
- Big endian: most significant byte comes first.

Example for integer `0x01234567` stored at increasing addresses:

```text
Little endian: 67 45 23 01
Big endian:    01 23 45 67
```

Byte ordering matters when reading raw memory, exchanging binary data across machines, or inspecting object code.

### 4. Bit-level operations

C supports bitwise Boolean operations on integer types:

```c
unsigned x = 0x55;  // 0101 0101
unsigned y = 0x0F;  // 0000 1111

x & y;  // 0x05: bits set in both
x | y;  // 0x5F: bits set in either
x ^ y;  // 0x5A: bits set in exactly one
~x;     // bitwise complement
```

These operations are useful for masks, flags, packed data, and low-level systems code.

Example: test whether the low byte of `x` is all ones:

```c
int low_byte_all_ones(unsigned x) {
    return (x & 0xFF) == 0xFF;
}
```

### 5. Logical operations are different from bitwise operations

Logical operators treat values as true or false:

```c
!0      // 1
!42     // 0
5 && 0  // 0
5 || 0  // 1
```

Bitwise operators operate on every bit:

```c
~0x00  // all bits become 1
0x55 & 0x0F  // 0x05
```

Confusing logical and bitwise operators is a common source of bugs.

### 6. Shift operations

Left shift moves bits toward more significant positions and fills with zeros:

```text
0x03 << 2 = 0x0C
0000 0011 -> 0000 1100
```

Right shift has two forms:

- Logical right shift fills the left side with zeros.
- Arithmetic right shift fills the left side with copies of the sign bit.

For unsigned values, C uses logical right shift. For signed negative values, the behavior is implementation-defined, though most machines use arithmetic right shift.

### 7. Unsigned integer representation

An unsigned integer uses all bits to represent a nonnegative value. For a `w`-bit unsigned value, the range is:

```text
0 to 2^w - 1
```

For 4 bits:

```text
0000 = 0
0001 = 1
...
1111 = 15
```

Unsigned arithmetic is modular. For `w` bits, operations are computed modulo `2^w`.

Example with 4-bit unsigned arithmetic:

```text
15 + 1 = 0
1111 + 0001 = 0000  modulo 16
```

### 8. Two's-complement signed representation

Most modern machines represent signed integers using two's complement. For `w` bits, the range is:

```text
-2^(w-1) to 2^(w-1) - 1
```

For 4 bits:

```text
0000 =  0
0001 =  1
0111 =  7
1000 = -8
1001 = -7
1111 = -1
```

The most significant bit has negative weight. In 4-bit two's complement:

```text
1011 = -8 + 0 + 2 + 1 = -5
```

Important asymmetry:

```text
TMin = -2^(w-1)
TMax =  2^(w-1) - 1
```

There is one more negative value than positive value.

### 9. Signed and unsigned conversions

C can convert between signed and unsigned integers without changing the bit pattern. The interpretation changes.

Example on a 32-bit machine:

```c
int x = -1;
unsigned ux = (unsigned) x;
```

The bit pattern is all ones. As signed, it means `-1`; as unsigned, it means `4294967295`.

This can create surprising comparisons:

```c
-1 < 0U
```

Because `0U` is unsigned, `-1` is converted to an unsigned value, so the comparison becomes approximately:

```text
4294967295 < 0
```

which is false on a 32-bit `unsigned` system.

### 10. Integer overflow

Fixed-size integer arithmetic can overflow.

Unsigned overflow wraps around modulo `2^w`:

```text
8-bit unsigned: 255 + 1 = 0
```

Two's-complement signed overflow exceeds the representable range. In C, signed overflow is undefined behavior, so compilers may assume it never happens when optimizing.

Example danger:

```c
int positive_sum(int x, int y) {
    return x + y > 0;
}
```

This is not a reliable overflow test because `x + y` may overflow before the comparison.

### 11. Multiplication by powers of two

Left shifting an integer by `k` positions often corresponds to multiplying by `2^k`, provided the result is representable:

```text
x << k  means  x * 2^k
```

Compilers commonly replace multiplication by constants with shifts, additions, and subtractions when profitable.

Example:

```text
x * 14 = (x << 4) - (x << 1)
       = 16x - 2x
```

### 12. Division by powers of two

Right shifting can approximate division by powers of two.

For nonnegative integers:

```text
x >> k  means  floor(x / 2^k)
```

For signed negative integers, arithmetic right shift rounds downward, while C integer division rounds toward zero. Compilers may add a bias before shifting to preserve C semantics.

Example:

```text
-7 / 2 = -3      C rounds toward zero
-7 >> 1 = -4     arithmetic shift rounds downward
```

### 13. Floating-point representation

Floating-point numbers approximate real numbers using a fixed number of bits. IEEE floating point represents values using three fields:

```text
sign | exponent | fraction
```

For a normalized value, the form is roughly:

```text
(-1)^sign * significand * 2^exponent
```

The IEEE single-precision format uses 32 bits:

```text
1 sign bit | 8 exponent bits | 23 fraction bits
```

The IEEE double-precision format uses 64 bits:

```text
1 sign bit | 11 exponent bits | 52 fraction bits
```

### 14. Normalized and denormalized floating-point values

Floating-point encodings are divided into cases:

- Normalized values represent most ordinary nonzero numbers.
- Denormalized values represent numbers very close to zero.
- Special values represent infinities and NaN.

Special values include:

```text
+infinity
-infinity
NaN  // Not a Number
```

Examples that can produce special values:

```c
1.0 / 0.0   // +infinity in IEEE floating point
0.0 / 0.0   // NaN
```

### 15. Floating-point rounding

Most real numbers cannot be represented exactly with a fixed number of bits. Floating-point operations round results to a representable value.

The default IEEE rounding mode is round-to-even: choose the nearest representable value, and if exactly halfway, choose the one with an even low-order digit.

A key consequence is that floating-point arithmetic is not always associative:

```c
(1e20 + -1e20) + 3.14  // 3.14
1e20 + (-1e20 + 3.14)  // 0.0 on typical systems
```

The mathematical expressions are equivalent, but finite precision changes the result.

### 16. Floating-point in C

C provides `float` and `double`. Conversions between integers and floating-point values can lose precision.

Examples:

```c
int x = 16777217;
float f = (float) x;
```

A 32-bit `float` does not have enough fraction bits to represent every large integer exactly, so `f` may round to `16777216`.

Useful rule: `double` has more precision than `float`, but neither represents all real numbers exactly.

## Examples to try

### Inspect bytes of an integer

```c
#include <stdio.h>

int main(void) {
    unsigned int x = 0x01234567;
    unsigned char *p = (unsigned char *) &x;

    for (size_t i = 0; i < sizeof x; i++) {
        printf("%02x ", p[i]);
    }
    printf("\n");
}
```

On a little-endian machine, this prints:

```text
67 45 23 01
```

### Observe unsigned wraparound

```c
#include <stdio.h>
#include <limits.h>

int main(void) {
    unsigned int x = UINT_MAX;
    printf("%u\n", x);
    printf("%u\n", x + 1);
}
```

Typical output:

```text
4294967295
0
```

### Floating-point precision surprise

```c
#include <stdio.h>

int main(void) {
    float f = 1e20f;
    printf("%.1f\n", (f + 1.0f) - f);
}
```

The result is typically `0.0`, because `1.0f` is too small to change a value as large as `1e20f` in single precision.

## Common pitfalls

1. Assuming signed integer overflow wraps around. In C, it is undefined behavior.
2. Mixing signed and unsigned values in comparisons.
3. Assuming right shift of signed negative values is portable.
4. Treating binary data as portable without specifying byte order.
5. Expecting floating-point arithmetic to behave exactly like real-number arithmetic.
6. Comparing floating-point values for exact equality after arithmetic.

## Summary

Chapter 2 shows that bit patterns only gain meaning through interpretation. Unsigned integers, two's-complement integers, and floating-point numbers all use finite bit patterns, so arithmetic can wrap, overflow, round, or produce special values. Programmers who understand these representations can write safer C code, debug low-level behavior, and reason about machine-level data more accurately.

## Review questions

1. Why is hexadecimal notation convenient for representing bytes?
2. How would `0x01234567` appear in memory on a little-endian machine?
3. What is the range of a 4-bit two's-complement integer?
4. Why can `-1 < 0U` evaluate to false in C?
5. What is the difference between logical and arithmetic right shift?
6. Why is signed integer overflow dangerous in C?
7. Why can `(a + b) + c` differ from `a + (b + c)` for floating-point values?
