# Chapter 24: cseries.lib Utility Library

## Foundation Types, Macros, and Portable Utilities

> **For Porting:** Core definitions in `cseries.h` are fully portable. Replace Mac-specific files (`macintosh_cseries.h`, `macintosh_utilities.c`) with your platform equivalents. Use `<stdint.h>` types for guaranteed sizes: `uint16_t` instead of `unsigned short`, `int32_t` instead of `long`.

---

## 24.1 What Problem Are We Solving?

Marathon needs a foundation layer providing:

- **Portable types** - Consistent sizes across compilers
- **Math utilities** - Fixed-point operations
- **Debug support** - Assertions and diagnostics
- **Memory management** - Allocation wrappers
- **Common macros** - MIN, MAX, FLAG operations

---

## 24.2 Core Types

```c
// Basic types with guaranteed sizes
typedef unsigned short word;   // 16 bits
typedef unsigned char byte;    // 8 bits
typedef byte boolean;          // TRUE/FALSE

typedef long fixed;            // 32-bit fixed-point (16.16)

typedef void *handle;          // Relocatable memory (legacy)
```

### Type Mapping Reference (Marathon → stdint.h)

For porting, use `<stdint.h>` types for guaranteed sizes across platforms:

| Marathon Type | Bits | Signedness | stdint.h Equivalent | Range |
|---------------|------|------------|---------------------|-------|
| `byte` | 8 | unsigned | `uint8_t` | 0 to 255 |
| `word` | 16 | unsigned | `uint16_t` | 0 to 65,535 |
| `boolean` | 8 | unsigned | `uint8_t` or `bool` | 0 or 1 |
| `short` | 16 | signed | `int16_t` | -32,768 to 32,767 |
| `unsigned short` | 16 | unsigned | `uint16_t` | 0 to 65,535 |
| `long` | 32 | signed | `int32_t` | -2,147,483,648 to 2,147,483,647 |
| `unsigned long` | 32 | unsigned | `uint32_t` | 0 to 4,294,967,295 |
| `fixed` | 32 | signed | `int32_t` | -32,768.0 to 32,767.99998 (16.16 fixed-point) |

```c
// Modern type definitions
#include <stdint.h>
#include <stdbool.h>

typedef uint8_t  byte;
typedef uint16_t word;
typedef int32_t  fixed;
typedef uint8_t  boolean;  // or use bool from stdbool.h
```

---

## 24.3 Essential Constants

```c
#define TRUE 1
#define FALSE 0
#define NONE -1                              // Invalid index marker

#define KILO 1024
#define MEG (KILO*KILO)                      // 1,048,576
#define GIG (KILO*MEG)                       // 1,073,741,824

#define MACHINE_TICKS_PER_SECOND 60          // Mac VBL rate
```

### Limit Constants

```c
enum {
    UNSIGNED_LONG_MAX = 4294967295,
    LONG_MAX = 2147483647L,
    LONG_MIN = (-2147483648L),
    LONG_BITS = 32,

    UNSIGNED_SHORT_MAX = 65535,
    SHORT_MAX = 32767,
    SHORT_MIN = (-32768),
    SHORT_BITS = 16,

    UNSIGNED_CHAR_MAX = 255,
    CHAR_MAX = 127,
    CHAR_MIN = (-128),
    CHAR_BITS = 8
};
```

---

## 24.4 Fixed-Point Mathematics

### Constants

```c
#define FIXED_FRACTIONAL_BITS 16
#define FIXED_ONE ((fixed)(1<<FIXED_FRACTIONAL_BITS))      // 65536
#define FIXED_ONE_HALF ((fixed)(1<<(FIXED_FRACTIONAL_BITS-1)))  // 32768
```

### Conversion Macros

```c
// Integer to fixed-point
#define INTEGER_TO_FIXED(s) (((fixed)(s))<<FIXED_FRACTIONAL_BITS)
// 5 → 327680 (5.0 in fixed)

// Fixed-point to integer (truncate)
#define FIXED_INTEGERAL_PART(f) ((short)((f)>>FIXED_FRACTIONAL_BITS))
#define FIXED_TO_INTEGER(f) FIXED_INTEGERAL_PART(f)
// 327680 → 5

// Fixed-point to integer (round)
#define FIXED_TO_INTEGER_ROUND(f) FIXED_INTEGERAL_PART((f)+FIXED_ONE_HALF)
// 360448 (5.5) → 6

// Get fractional part only
#define FIXED_FRACTIONAL_PART(f) (((fixed)(f))&(FIXED_ONE-1))
// 360448 → 32768 (0.5)

// Floating-point conversion (for debugging)
#define FIXED_TO_FLOAT(f) (((double)(f))/FIXED_ONE)
#define FLOAT_TO_FIXED(f) ((fixed)((f)*FIXED_ONE))
```

### Fixed-Point Visualization

```
Fixed-Point Format (16.16):

        Integer Part           Fractional Part
    ├────────────────────┼────────────────────────┤
    │    16 bits         │       16 bits          │
    │   (signed)         │     (unsigned)         │
    └────────────────────┴────────────────────────┘

    Bit 31 ──────────────────────────────────► Bit 0

Example values:
    FIXED_ONE      = 0x00010000 = 65536   = 1.0
    FIXED_ONE_HALF = 0x00008000 = 32768   = 0.5
    0x00028000     = 163840               = 2.5
    0xFFFF0000     = -65536               = -1.0
```

---

## 24.5 Utility Macros

### Math Operations

```c
#define SGN(x) ((x)?((x)<0?-1:1):0)          // Sign: -1, 0, or 1
#define ABS(x) ((x>=0) ? (x) : -(x))         // Absolute value
#define MIN(a,b) ((a)>(b)?(b):(a))           // Minimum
#define MAX(a,b) ((a)>(b)?(a):(b))           // Maximum
```

### Clamping

```c
#define FLOOR(n,floor) ((n)<(floor)?(floor):(n))
// FLOOR(3, 5) = 5  (raise to minimum)

#define CEILING(n,ceiling) ((n)>(ceiling)?(ceiling):(n))
// CEILING(10, 7) = 7  (lower to maximum)

#define PIN(n,floor,ceiling) ((n)<(floor) ? (floor) : CEILING(n,ceiling))
// PIN(3, 5, 10) = 5   (clamp to range)
// PIN(8, 5, 10) = 8
// PIN(15, 5, 10) = 10
```

### Bit Swapping

```c
#define SWAP(a,b) a^= b, b^= a, a^= b
// XOR swap without temporary variable
```

---

## 24.6 Flag Operations

```c
#define FLAG(b) (1<<(b))
// FLAG(3) = 0x08 = 8 = 0b00001000

// 16-bit flag operations
#define TEST_FLAG16(f, b) ((f)&(word)FLAG(b))
#define SWAP_FLAG16(f, b) ((f)^=(word)FLAG(b))
#define SET_FLAG16(f, b, v) ((v) ? ((f)|=(word)FLAG(b)) : ((f)&=(word)~FLAG(b)))

// 32-bit flag operations
#define TEST_FLAG32(f, b) ((f)&(unsigned long)FLAG(b))
#define SWAP_FLAG32(f, b) ((f)^=(unsigned long)FLAG(b))
#define SET_FLAG32(f, b, v) ((v) ? ((f)|=(unsigned long)FLAG(b)) : ((f)&=(unsigned long)~FLAG(b)))
```

### Flag Usage Example

```c
// Define flags for monster state
enum {
    _monster_is_active = 0,
    _monster_is_blind = 1,
    _monster_is_deaf = 2,
    _monster_is_flying = 3
};

word monster_flags = 0;

// Set a flag
SET_FLAG16(monster_flags, _monster_is_active, TRUE);   // monster_flags = 0x0001

// Test a flag
if (TEST_FLAG16(monster_flags, _monster_is_flying)) {  // FALSE
    // Monster can fly
}

// Toggle a flag
SWAP_FLAG16(monster_flags, _monster_is_blind);         // monster_flags = 0x0003
```

---

## 24.7 Debug Support

### Assertions

```c
#ifdef DEBUG
    #define halt() _assertion_failure(NULL, __FILE__, __LINE__, TRUE)
    #define vhalt(diag) _assertion_failure(diag, __FILE__, __LINE__, TRUE)

    #define assert(expr) if (!(expr)) _assertion_failure(#expr, __FILE__, __LINE__, TRUE)
    #define vassert(expr,diag) if (!(expr)) _assertion_failure(diag, __FILE__, __LINE__, TRUE)

    #define warn(expr) if (!(expr)) _assertion_failure(#expr, __FILE__, __LINE__, FALSE)
    #define vwarn(expr,diag) if (!(expr)) _assertion_failure(diag, __FILE__, __LINE__, FALSE)

    #define pause() _assertion_failure(NULL, __FILE__, __LINE__, FALSE)
    #define vpause(diag) _assertion_failure(diag, __FILE__, __LINE__, FALSE)
#else
    // All debug macros become no-ops in release builds
    #define halt()
    #define assert(expr)
    // ... etc
#endif
```

### Debug Output

```c
void initialize_debugger(boolean force_debugger_on);
int dprintf(const char *format, ...);                    // Debug printf
char *csprintf(char *buffer, char *format, ...);         // sprintf variant
int rsprintf(char *s, short resource_number, short string_number, ...);  // Resource string printf
```

---

## 24.8 Memory Management

```c
// Marathon redefines malloc/free to use its own allocator
#define malloc(size) new_pointer(size)
#define free(ptr) dispose_pointer(ptr)

void *new_pointer(long size);
void dispose_pointer(void *pointer);
```

### Platform Implementation

For porting, these can be simple wrappers:

```c
// Modern implementation
void *new_pointer(long size) {
    return malloc(size);  // Use standard malloc
}

void dispose_pointer(void *pointer) {
    free(pointer);        // Use standard free
}
```

---

## 24.9 Array Manipulation

```c
#define compact_array(array, element, nmemb, size) \
    if ((element)<(nmemb)-1) \
        memcpy(((byte*)(array))+(element)*(size), \
               ((byte*)(array))+((element)+1)*(size), \
               ((nmemb)-(element)-1)*(size))
```

### Usage

```
Remove element 2 from array of 5:

Before: [A][B][C][D][E]
             ↑
         remove this

compact_array(arr, 2, 5, sizeof(element)):
  memcpy from [D][E] to position of [C]

After:  [A][B][D][E][?]
                    ↑
              garbage (not used)
```

---

## 24.10 Timing

```c
unsigned long machine_tick_count(void);
// Returns ticks since system start (60 Hz on Mac)

// Usage for timing
unsigned long start = machine_tick_count();
// ... do work ...
unsigned long elapsed = machine_tick_count() - start;
// elapsed is in 1/60th second units
```

---

## 24.11 String Utilities

```c
char *strupr(char *string);  // Convert to uppercase
char *strlwr(char *string);  // Convert to lowercase

char *getcstr(char *buffer, short collection_number, short string_number);
// Get C string from resource
```

---

## 24.12 Pixel Type Definitions (textures.h)

Defines pixel formats and bitmap structures for rendering:

```c
typedef unsigned char pixel8;    // 8-bit indexed color
typedef unsigned short pixel16;  // 16-bit RGB (5-5-5)
typedef unsigned long pixel32;   // 32-bit RGB (8-8-8)

#define PIXEL8_MAXIMUM_COLORS   256
#define PIXEL16_MAXIMUM_COLORS  32768
#define PIXEL32_MAXIMUM_COLORS  16777216
```

### Pixel Format Diagrams

```
pixel16 (RGB 5-5-5, 15-bit color):
┌───┬─────────────┬─────────────┬─────────────┐
│ X │    RED      │   GREEN     │    BLUE     │
│ 1 │   5 bits    │   5 bits    │   5 bits    │
└───┴─────────────┴─────────────┴─────────────┘
  Total: 32,768 colors (bit 15 unused)

pixel32 (RGB 8-8-8, 24-bit color):
┌─────────────┬─────────────┬─────────────┬─────────────┐
│   (unused)  │    RED      │   GREEN     │    BLUE     │
│   8 bits    │   8 bits    │   8 bits    │   8 bits    │
└─────────────┴─────────────┴─────────────┴─────────────┘
  Total: 16,777,216 colors

pixel8 (Indexed, 8-bit):
┌─────────────────────────────────┐
│       PALETTE INDEX             │
│          8 bits                 │
└─────────────────────────────────┘
  Points to color_table entry (256 colors)
```

### Pixel Macros

```c
// 16-bit pixel operations (RGB 5-5-5)
#define RED16(p)    ((p)>>10)
#define GREEN16(p)  (((p)>>5)&0x1f)
#define BLUE16(p)   ((p)&0x1f)
#define BUILD_PIXEL16(r,g,b) (((r)<<10)|((g)<<5)|(b))

// 32-bit pixel operations (RGB 8-8-8)
#define RED32(p)    ((p)>>16)
#define GREEN32(p)  (((p)>>8)&0xff)
#define BLUE32(p)   ((p)&0xff)
#define BUILD_PIXEL32(r,g,b) (((r)<<16)|((g)<<8)|(b))
```

### Bitmap Definition

```c
enum {  // bitmap flags
    _COLUMN_ORDER_BIT = 0x8000,  // Data stored column-major (for walls)
    _TRANSPARENT_BIT  = 0x4000   // Has transparent pixels
};

struct bitmap_definition {
    short width, height;      // Dimensions in pixels
    short bytes_per_row;      // NONE = RLE compressed
    short flags;              // _COLUMN_ORDER_BIT, _TRANSPARENT_BIT
    short bit_depth;          // Always 8 for indexed color
    short unused[8];
    pixel8 *row_addresses[1]; // Flexible array of row pointers
};
```

---

## 24.13 RLE Compression (rle.c)

Run-length encoding for sprite/texture compression:

### Encoding Format

```
First 4 bytes: Uncompressed size (long)
Then opcodes:
  0 ≤ n < 128   : Repeat next byte (n+3) times
  128 ≤ n ≤ 255 : Copy next (n-127) bytes literally
```

### API

```c
// Get uncompressed size from compressed data
long get_destination_size(byte *compressed);

// Compress data, returns compressed size or -1 if expansion
long compress_bytes(byte *raw, long raw_size,
                   byte *compressed, long maximum_compressed_size);

// Decompress (caller must allocate destination)
void uncompress_bytes(byte *compressed, byte *raw);
```

### Compression Visualization

```
Original Data (16 bytes):
┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
│ 41 │ 42 │ 43 │ FF │ FF │ FF │ FF │ FF │ FF │ FF │ 10 │ 20 │ 30 │ 30 │ 30 │ 30 │
└────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘
  A    B    C   ←── 7 repeated FFs ──→              ←─ 4 repeated 30s ─→

Compressed (12 bytes):
┌──────────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
│ 00000010 │ 83 │ 41 │ 42 │ 43 │ 04 │ FF │ 83 │ 10 │ 20 │ 01 │ 30 │
│(size=16) │lit │ A  │ B  │ C  │run │byte│lit │    │    │run │byte│
└──────────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘

Opcode interpretation:
  0-127:   Run of (n+3) copies of next byte
  128-255: Literal run of (n-127) bytes following
```

---

## 24.14 Byte Swapping (byte_swapping.c)

Handles endianness conversion between big-endian (Mac) and little-endian (x86):

### Swap Macros

```c
#define SWAP2(q) (((q)>>8) | (((q)<<8)&0xff00))

#define SWAP4(q) (((q)>>24) | \
                  (((q)>>8)&0xff00) | \
                  (((q)<<8)&0x00ff00) | \
                  (((q)<<24)&0xff000000))
```

### Byte Order Visualization

```
Big-Endian (Mac)                 Little-Endian (x86)
Most significant byte first      Least significant byte first

16-bit value 0x1234:
┌──────┬──────┐                  ┌──────┬──────┐
│  12  │  34  │     SWAP2 →      │  34  │  12  │
└──────┴──────┘                  └──────┴──────┘

32-bit value 0x12345678:
┌──────┬──────┬──────┬──────┐    ┌──────┬──────┬──────┬──────┐
│  12  │  34  │  56  │  78  │ →  │  78  │  56  │  34  │  12  │
└──────┴──────┴──────┴──────┘    └──────┴──────┴──────┴──────┘
```

### Field Descriptors

For swapping entire structures:

```c
enum {
    _byte  = 1,    // Skip 1 byte (no swap)
    _2byte = -2,   // Swap 2-byte value
    _4byte = -4    // Swap 4-byte value
};

// Example: swap structure with mixed field sizes
_bs_field example_fields[] = { _2byte, _byte, _byte, _4byte, 0 };
// Swaps: 2-byte short, skips 2 bytes, swaps 4-byte long
```

---

## 24.15 Summary

The cseries library provides:

- **Portable type definitions** for cross-platform consistency
- **Fixed-point math** macros for 16.16 format
- **Utility macros** (MIN, MAX, PIN, FLAG operations)
- **Debug infrastructure** with assertions and logging
- **Memory wrappers** abstracting platform allocation

### Key Source Files

| File | Purpose | Portable? |
|------|---------|-----------|
| `cseries.h` | Core types and macros | ✓ Yes |
| `textures.h` | Pixel type definitions | ✓ Yes |
| `rle.c/h` | Run-length compression | ✓ Yes |
| `checksum.c/h` | CRC calculations | ✓ Yes |
| `byte_swapping.c/h` | Endianness utilities | ✓ Yes |
| `macintosh_cseries.h` | Mac extensions | ✗ Replace |
| `macintosh_utilities.c` | Mac helpers | ✗ Replace |

---

*Next: [Chapter 25: Media/Liquid System](25_media.md) - Water, lava, and environmental liquids*
