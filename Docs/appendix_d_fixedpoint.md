# Appendix D: Fixed-Point Conversion

## Working with Marathon's Number Format

---

## D.1 Why Fixed-Point?

Marathon uses fixed-point arithmetic throughout its engine for several reasons:

1. **Deterministic simulation** - Floating-point behavior varies across CPUs; fixed-point is identical everywhere
2. **Network synchronization** - All players compute exact same results
3. **Performance** - Integer operations were faster than FPU on 68K Macs
4. **Precision control** - Known precision at all times

---

## D.2 The 16.16 Format

Marathon uses signed 32-bit fixed-point with 16 integer bits and 16 fractional bits:

```
┌─────────────────────────────────────────────────────────────────┐
│                    FIXED-POINT FORMAT (16.16)                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Bit layout (32 bits total):                                    │
│                                                                  │
│   ┌─────────────────────────┬─────────────────────────┐         │
│   │   Integer (16 bits)     │   Fraction (16 bits)    │         │
│   │   (signed)              │   (unsigned)            │         │
│   └─────────────────────────┴─────────────────────────┘         │
│   31                      16 15                        0         │
│                                                                  │
│   Range: -32768.0 to +32767.99998474...                          │
│   Precision: 1/65536 ≈ 0.0000152587890625                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## D.3 Core Constants

```c
// From cseries.h
typedef long fixed;  // 32-bit signed

#define FIXED_FRACTIONAL_BITS 16
#define FIXED_ONE             (1L << FIXED_FRACTIONAL_BITS)  // 0x10000 = 65536
#define FIXED_ONE_HALF        (FIXED_ONE >> 1)               // 0x8000 = 32768
```

---

## D.4 Conversion Functions

### Integer ↔ Fixed

```c
// Integer to fixed (multiply by 65536)
#define INTEGER_TO_FIXED(i) ((fixed)((i) << FIXED_FRACTIONAL_BITS))

// Fixed to integer (truncate toward zero)
#define FIXED_INTEGERAL_PART(f) ((f) >> FIXED_FRACTIONAL_BITS)

// Fixed to integer (round to nearest)
#define FIXED_TO_INTEGER_ROUND(f) \
    (((f) + FIXED_ONE_HALF) >> FIXED_FRACTIONAL_BITS)

// Get fractional part only
#define FIXED_FRACTIONAL_PART(f) ((f) & (FIXED_ONE - 1))
```

### Examples

```c
fixed a = INTEGER_TO_FIXED(5);        // a = 0x50000 = 327680
fixed b = INTEGER_TO_FIXED(-3);       // b = 0xFFFD0000 = -196608

int i = FIXED_INTEGERAL_PART(a);      // i = 5
int j = FIXED_INTEGERAL_PART(b);      // j = -3

// With fractional values:
fixed c = 0x28000;                    // c = 2.5 (2 + 0.5)
int k = FIXED_INTEGERAL_PART(c);      // k = 2 (truncated)
int m = FIXED_TO_INTEGER_ROUND(c);    // m = 3 (rounded)
```

---

## D.5 Arithmetic Operations

### Addition and Subtraction

Fixed-point addition/subtraction works directly (same as integers):

```c
fixed a = INTEGER_TO_FIXED(3);    // 3.0
fixed b = INTEGER_TO_FIXED(2);    // 2.0
fixed sum = a + b;                // 5.0 (works!)
fixed diff = a - b;               // 1.0 (works!)
```

### Multiplication

Multiplication requires adjustment to avoid overflow:

```c
// Standard multiplication (may overflow for large values)
#define FIXED_MULTIPLY(a, b) \
    (((long)(a) * (long)(b)) >> FIXED_FRACTIONAL_BITS)

// Safe multiplication using 64-bit intermediate
static inline fixed fixed_multiply_safe(fixed a, fixed b) {
    return (fixed)(((int64_t)a * (int64_t)b) >> FIXED_FRACTIONAL_BITS);
}
```

**Why the shift?**
```
a = 3.0 = 0x30000 = 196608
b = 2.0 = 0x20000 = 131072

a * b = 196608 * 131072 = 25769803776 (0x600000000)

Without shift: result = 25769803776 (wrong!)
With shift:    result = 25769803776 >> 16 = 393216 = 0x60000 = 6.0 (correct!)
```

### Division

Division requires pre-shift to maintain precision:

```c
// Standard division
#define FIXED_DIVIDE(a, b) \
    ((((long)(a)) << FIXED_FRACTIONAL_BITS) / (b))

// Safe division using 64-bit
static inline fixed fixed_divide_safe(fixed a, fixed b) {
    return (fixed)(((int64_t)a << FIXED_FRACTIONAL_BITS) / b);
}
```

---

## D.6 Float Conversion (For Porting)

When modernizing code, you may want to convert to/from floats:

```c
// Fixed to float
static inline float fixed_to_float(fixed f) {
    return (float)f / (float)FIXED_ONE;
}

// Float to fixed
static inline fixed float_to_fixed(float f) {
    return (fixed)(f * (float)FIXED_ONE);
}

// Examples:
fixed a = float_to_fixed(3.5f);   // a = 229376 (0x38000)
float b = fixed_to_float(a);       // b = 3.5f
```

---

## D.7 Common Patterns in Marathon Code

### Scaling Values

```c
// Scale damage by a factor
short total_damage = damage->base + random() % damage->random;
total_damage = FIXED_INTEGERAL_PART(total_damage * damage->scale);

// Scale is a fixed-point multiplier
// If scale = FIXED_ONE (65536), result unchanged
// If scale = FIXED_ONE_HALF (32768), result halved
```

### Interpolation

```c
// Linear interpolation between two values
// t is 0..FIXED_ONE (0.0 to 1.0)
fixed lerp(fixed a, fixed b, fixed t) {
    return a + FIXED_INTEGERAL_PART((b - a) * t);
}
```

### Trigonometry

Marathon uses lookup tables with fixed-point results:

```c
// Trig tables store values in 16.16 fixed-point
// Range: -TRIG_MAGNITUDE to +TRIG_MAGNITUDE (±16384)
#define TRIG_SHIFT 14  // Convert to usable range
#define TRIG_MAGNITUDE (1 << TRIG_SHIFT)

// Usage:
fixed dx = (cosine_table[angle] * distance) >> TRIG_SHIFT;
fixed dy = (sine_table[angle] * distance) >> TRIG_SHIFT;
```

---

## D.8 Precision Visualization

```
┌─────────────────────────────────────────────────────────────────┐
│                    FIXED-POINT PRECISION                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Decimal        Hex           Binary (16 fractional bits)       │
│   ────────────────────────────────────────────────────────────   │
│   0.0            0x00000       0000 0000 0000 0000               │
│   0.0625         0x01000       0001 0000 0000 0000               │
│   0.125          0x02000       0010 0000 0000 0000               │
│   0.25           0x04000       0100 0000 0000 0000               │
│   0.5            0x08000       1000 0000 0000 0000               │
│   0.75           0x0C000       1100 0000 0000 0000               │
│   1.0            0x10000       [1] 0000 0000 0000 0000           │
│   1.5            0x18000       [1] 1000 0000 0000 0000           │
│   2.0            0x20000       [10] 0000 0000 0000 0000          │
│   -1.0           0xFFFF0000    [1111...1111] 0000...             │
│                                                                  │
│   [brackets] = integer bits, rest = fractional bits              │
│                                                                  │
│   Smallest positive value:                                       │
│   0.0000152... = 0x00001 = 1/65536                               │
│                                                                  │
│   Largest positive value:                                        │
│   32767.99998... = 0x7FFFFFFF                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

![Fixed-Point Number Format](diagrams/fixed_point.svg)

---

## D.9 World Units vs Fixed-Point

Marathon uses two related but distinct systems:

| System | Type | One Unit | Usage |
|--------|------|----------|-------|
| **World Units** | `world_distance` (short) | 1024 | Positions, distances |
| **Fixed-Point** | `fixed` (long) | 65536 | Calculations, scaling |

### Conversion

```c
// World to fixed (multiply by 64 effectively)
#define WORLD_TO_FIXED(w) ((fixed)(w) << (FIXED_FRACTIONAL_BITS - WORLD_FRACTIONAL_BITS))

// Fixed to world (divide by 64 effectively)
#define FIXED_TO_WORLD(f) ((world_distance)((f) >> (FIXED_FRACTIONAL_BITS - WORLD_FRACTIONAL_BITS)))

// Where:
#define WORLD_FRACTIONAL_BITS 10
// So shift amount = 16 - 10 = 6
```

---

## D.10 Overflow Considerations

### When Overflow Can Happen

```c
// DANGEROUS: Two large fixed values
fixed a = INTEGER_TO_FIXED(200);   // 13,107,200
fixed b = INTEGER_TO_FIXED(200);   // 13,107,200
fixed c = a * b;                    // Overflow! (32-bit can't hold 2^34)

// SAFE: Use 64-bit intermediate
fixed c = (fixed)(((int64_t)a * (int64_t)b) >> 16);
```

### Marathon's Approach

Most Marathon calculations stay within safe ranges:
- World coordinates < 32768 units
- Trig values ±16384
- Most multiplications involve at least one small value

---

## D.11 Porting Strategy

### Option 1: Keep Fixed-Point

Preserve determinism and behavior:

```c
// Just use Marathon's existing macros
// Add 64-bit intermediate for safety on modern compilers

typedef int32_t fixed;
#define FIXED_MULTIPLY(a, b) ((fixed)(((int64_t)(a) * (int64_t)(b)) >> 16))
```

### Option 2: Convert to Float

Easier to read, modern approach:

```c
// Replace all fixed with float
typedef float fixed;
#define FIXED_ONE 1.0f
#define INTEGER_TO_FIXED(i) ((float)(i))
#define FIXED_INTEGERAL_PART(f) ((int)(f))
#define FIXED_MULTIPLY(a, b) ((a) * (b))
```

**Warning:** Network play will desync if different players use different floating-point implementations!

### Option 3: Hybrid

Keep fixed for game logic, use float for rendering:

```c
// Game logic: fixed-point (deterministic)
fixed player_x, player_y;

// Rendering: convert to float
float render_x = fixed_to_float(player_x);
float render_y = fixed_to_float(player_y);
```

---

## D.12 Quick Reference

```c
// Essential conversions
fixed f = INTEGER_TO_FIXED(n);        // int → fixed
int n = FIXED_INTEGERAL_PART(f);      // fixed → int (truncate)
int n = FIXED_TO_INTEGER_ROUND(f);    // fixed → int (round)

// Arithmetic
fixed sum = a + b;                     // Direct add
fixed diff = a - b;                    // Direct subtract
fixed prod = FIXED_MULTIPLY(a, b);    // Multiply with shift
fixed quot = FIXED_DIVIDE(a, b);      // Divide with pre-shift

// Float conversion (for debugging/porting)
float fl = (float)f / 65536.0f;       // fixed → float
fixed fx = (fixed)(fl * 65536.0f);    // float → fixed

// Common values
FIXED_ONE      = 65536  = 1.0
FIXED_ONE_HALF = 32768  = 0.5
0x10000        = 65536  = 1.0
0x08000        = 32768  = 0.5
0x04000        = 16384  = 0.25
0x20000        = 131072 = 2.0
```

---

*End of Appendices*

---

*Return to: [Table of Contents](../README.md)*
