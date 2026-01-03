# Chapter 10: File Formats

## WAD Files, Shapes, and Sounds

> **For Porting:** Great news! All game data files (Maps, Shapes, Sounds) are readable with standard `fopen()`/`fread()`. Replace `FSSpec`/`FSRead` with stdio in `wad.c`, `game_wad.c`. Add byte swapping (files are big-endian, x86 is little-endian). Only the optional Images file uses Mac resource forks.

---

## 10.1 What Problem Are We Solving?

Marathon needs to load and store large amounts of game data efficiently:

- **Map geometry** - Polygons, lines, vertices, platforms
- **Textures and sprites** - Wall textures, monster animations, item graphics
- **Sound effects** - Weapons, monsters, ambient sounds
- **Saved games** - Complete world state restoration

**The constraints:**
- Must load quickly from 1995 hard drives
- Must minimize memory footprint
- Must support modular level packs
- Must save/restore exact game state

**Marathon's solution: Custom Binary Formats**

Marathon uses purpose-built binary formats optimized for the game engine. Despite being Mac software, the core data files use **standard binary formats** readable on any platform—only interface graphics use Mac resource forks.

---

## 10.2 Understanding Marathon's File Types

Before diving into formats, let's understand what files Marathon uses.

### File Categories

| File Type | Format | Readable on All Platforms? |
|-----------|--------|---------------------------|
| Map files | Marathon WAD | Yes - standard `fopen`/`fread` |
| Shapes (Shapes8/16) | Data fork binary | Yes - standard `fopen`/`fread` |
| Sounds (Sounds8/16) | Data fork binary | Yes - standard `fopen`/`fread` |
| Images | Mac resource fork | No - needs extraction or stub |

**Key insight**: The core game data (shapes, sounds, maps) is all in standard binary formats. Only optional interface graphics use Mac-specific resource forks.

---

## 10.3 Marathon WAD Format

Marathon uses its own WAD format for maps—**NOT** Doom WADs despite the similar name.

### WAD File Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                    WAD FILE STRUCTURE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │            WAD HEADER (128 bytes)                         │  │
│  │  ├─ version, data_version                                 │  │
│  │  ├─ file_name[64]                                         │  │
│  │  ├─ checksum (CRC32)                                      │  │
│  │  ├─ directory_offset ─────────────────────────────┐       │  │
│  │  ├─ wad_count (number of levels)                  │       │  │
│  │  └─ parent_checksum (for patches)                 │       │  │
│  └───────────────────────────────────────────────────│───────┘  │
│                                                      │          │
│  ┌───────────────────────────────────────────────────│───────┐  │
│  │         LEVEL DATA (repeated per level)           │       │  │
│  │                                                   │       │  │
│  │  [Entry Header] tag='PNTS' length=N              │       │  │
│  │  [Point Data - N bytes]                          │       │  │
│  │                                                   │       │  │
│  │  [Entry Header] tag='LINS' length=M              │       │  │
│  │  [Line Data - M bytes]                           │       │  │
│  │                                                   │       │  │
│  │  [Entry Header] tag='SIDS' length=...            │       │  │
│  │  [Side Data]                                     │       │  │
│  │                                                   │       │  │
│  │  [Entry Header] tag='POLY' length=...            │       │  │
│  │  [Polygon Data]                                  │       │  │
│  │                                                   │       │  │
│  │  ... more tagged entries ...                     │       │  │
│  └───────────────────────────────────────────────────│───────┘  │
│                                                      │          │
│  ┌───────────────────────────────────────────────────▼───────┐  │
│  │         DIRECTORY (at directory_offset)                   │  │
│  │                                                           │  │
│  │  [Directory Entry 0] offset, length, index=0              │  │
│  │  [Directory Entry 1] offset, length, index=1              │  │
│  │  ... one per level ...                                    │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### WAD Header Structure

```c
// 128 bytes
struct wad_header {
    short version;                           // 0=pre-entry, 2=M2, 4=Infinity
    short data_version;                      // Application-defined
    char file_name[64];                      // Internal filename
    unsigned long checksum;                  // CRC32 of file
    long directory_offset;                   // Offset to directory
    short wad_count;                         // Number of entries (levels)
    short application_specific_directory_data_size;  // Extra dir data
    short entry_header_size;                 // 16 bytes
    short directory_entry_base_size;         // 10 bytes
    unsigned long parent_checksum;           // Non-zero = patch file
    short unused[20];                        // Reserved
};
```

### Directory Entry Structure

```c
// 10 bytes
struct directory_entry {
    long offset_to_start;    // Where level data begins
    long length;             // Size of level data
    short index;             // Level number
};
```

### Entry Header Structure

```c
// 16 bytes
struct entry_header {
    long tag;            // 'PNTS', 'LINS', 'SIDS', 'POLY', etc.
    long next_offset;    // Relative offset to next entry
    long length;         // Data size
    long offset;         // Data offset
};
```

### Tag Types

| Tag | Description |
|-----|-------------|
| `'PNTS'` | Vertex points |
| `'LINS'` | Line segments |
| `'SIDS'` | Side definitions |
| `'POLY'` | Polygon data |
| `'LITE'` | Light sources |
| `'OBJS'` | Object placement |
| `'Minf'` | Map info |
| `'plat'` | Platforms |
| `'medi'` | Media (liquids) |
| ...many more | See `tags.h` |

### Reading a WAD File

```c
// Modern C implementation
FILE* fp = fopen("Map.wad", "rb");

// Read header
struct wad_header header;
fread(&header, sizeof(header), 1, fp);

// Byte swap on little-endian systems
header.version = swap16(header.version);
header.wad_count = swap16(header.wad_count);
header.directory_offset = swap32(header.directory_offset);

// Seek to directory
fseek(fp, header.directory_offset, SEEK_SET);

// Read directory entries
for (int i = 0; i < header.wad_count; i++) {
    struct directory_entry entry;
    fread(&entry, sizeof(entry), 1, fp);

    // Byte swap
    entry.offset_to_start = swap32(entry.offset_to_start);
    entry.length = swap32(entry.length);

    // Now seek to entry.offset_to_start to read level data
}
```

---

## 10.4 Shape Files

Shape files contain all textures and sprites. Despite Marathon being a Mac game, shapes are stored in **standard data fork** as binary files.

### Shape File Structure

```
Shape File Layout:
┌─────────────────────────────────────────────────────────────────┐
│  Collection Header Table (32 entries × 8 bytes = 256 bytes)    │
├─────────────────────────────────────────────────────────────────┤
│  Collection 0 data                                              │
├─────────────────────────────────────────────────────────────────┤
│  Collection 1 data                                              │
├─────────────────────────────────────────────────────────────────┤
│  ... (up to 32 collections)                                     │
└─────────────────────────────────────────────────────────────────┘
```

### Collection Header

```c
struct collection_header {
    long offset, length;        // 8-bit collection data
    long offset16, length16;    // 16-bit collection data

    // Runtime (not in file):
    struct collection_definition **collection;
    void **shading_tables;
};

#define MAXIMUM_COLLECTIONS 32
```

### Reading a Shape File

```c
FILE* fp = fopen("Shapes16", "rb");

// Read collection headers
struct collection_header headers[32];
fread(&headers, sizeof(struct collection_header), 32, fp);

// Byte swap (big-endian to little-endian)
for (int i = 0; i < 32; i++) {
    headers[i].offset16 = swap32(headers[i].offset16);
    headers[i].length16 = swap32(headers[i].length16);
}

// Load collection #5 at 16-bit
fseek(fp, headers[5].offset16, SEEK_SET);
byte* data = malloc(headers[5].length16);
fread(data, headers[5].length16, 1, fp);
```

### Collection Definition

```c
// 512 bytes (header portion)
struct collection_definition {
    short version;               // Should be 3
    short type;                  // Collection type
    word flags;

    short color_count, clut_count;
    long color_table_offset;

    short high_level_shape_count;
    long high_level_shape_offset_table_offset;

    short low_level_shape_count;
    long low_level_shape_offset_table_offset;

    short bitmap_count;
    long bitmap_offset_table_offset;

    short pixels_to_world;       // Scale factor
    long size;                   // Total size
};
```

### Shape Hierarchy

```
Collection (e.g., "Hunter" collection)
    │
    ├─► High-Level Shapes (animations)
    │     │
    │     ├─► "Walking" (8 views × 12 frames)
    │     ├─► "Attacking" (8 views × 6 frames)
    │     ├─► "Dying" (1 view × 8 frames)
    │     └─► ...
    │
    ├─► Low-Level Shapes (individual frames)
    │     │
    │     ├─► Frame 0: bitmap 5, origin (32, 64)
    │     ├─► Frame 1: bitmap 5, origin (33, 65), mirrored
    │     └─► ...
    │
    └─► Bitmaps (pixel data)
          │
          ├─► Bitmap 0: 64×128, RLE compressed
          ├─► Bitmap 1: 64×128, RLE compressed
          └─► ...
```

### High-Level Shape (Animation)

```c
struct high_level_shape_definition {
    short type;                     // Always 0
    word flags;                     // Animation flags

    char name[33];                  // "Walking", "Attacking", etc.

    short number_of_views;          // 1, 3, 5, or 8 viewing angles
    short frames_per_view;          // Frames in animation
    short ticks_per_frame;          // Animation speed
    short key_frame;                // Important frame (attack moment)

    short transfer_mode;            // Special rendering mode
    short transfer_mode_period;     // Effect cycle time

    short first_frame_sound;        // Sound at animation start
    short key_frame_sound;          // Sound at key frame
    short last_frame_sound;         // Sound at animation end

    short pixels_to_world;          // Scale factor
    short loop_frame;               // Where to loop back

    // Followed by: number_of_views × frames_per_view indices
    short low_level_shape_indexes[];
};
```

### Low-Level Shape (Frame)

```c
struct low_level_shape_definition {
    word flags;                     // Mirroring and obscured flags
    fixed minimum_light_intensity;  // Self-illumination minimum

    short bitmap_index;             // Which bitmap to use

    short origin_x, origin_y;       // Sprite anchor point
    short key_x, key_y;             // Registration point

    short world_left, world_right;  // World-space bounds
    short world_top, world_bottom;
    short world_x0, world_y0;       // World offset
};

// Flags
#define _X_MIRRORED_BIT 0x8000        // Horizontally flip
#define _Y_MIRRORED_BIT 0x4000        // Vertically flip
#define _KEYPOINT_OBSCURED_BIT 0x2000 // Key point hidden
```

### Bitmap Compression

Marathon supports two bitmap formats:

**Raw Format:**
- Width × Height bytes
- Direct pixel data

**RLE Compressed Format:**
- Scanline-based run-length encoding
- If byte < 128: Copy next `byte` pixels literally
- If byte >= 128: Repeat next pixel `(256 - byte)` times

### Color Tables

```c
struct rgb_color_value {
    byte flags;      // SELF_LUMINESCENT = 0x80
    byte value;      // Brightness
    word red;        // 0-65535
    word green;
    word blue;
};
```

---

## 10.5 Byte Order Considerations

All Marathon data is **big-endian** (Mac byte order). On little-endian systems (x86, ARM), byte swapping is required.

### Detecting Byte Order

At compile time, detect if byte swapping is needed:

```c
// Method 1: Use standard macros (POSIX)
#include <endian.h>  // Linux
// or <machine/endian.h> on macOS

#if __BYTE_ORDER == __LITTLE_ENDIAN
    #define MARATHON_NEEDS_SWAP 1
#else
    #define MARATHON_NEEDS_SWAP 0
#endif

// Method 2: Runtime detection (portable fallback)
static inline int is_little_endian(void) {
    uint16_t test = 1;
    return *((uint8_t*)&test) == 1;
}

// Method 3: Compiler-specific (most reliable)
#if defined(__LITTLE_ENDIAN__) || defined(_M_IX86) || defined(_M_X64) || \
    defined(__x86_64__) || defined(__i386__) || defined(__aarch64__)
    #define MARATHON_NEEDS_SWAP 1
#else
    #define MARATHON_NEEDS_SWAP 0
#endif
```

### Byte Swap Options

**Option A: Custom swap functions (portable, no dependencies):**

```c
uint16_t swap16(uint16_t val) {
    return (val << 8) | (val >> 8);
}

uint32_t swap32(uint32_t val) {
    return ((val & 0xFF) << 24) |
           ((val & 0xFF00) << 8) |
           ((val & 0xFF0000) >> 8) |
           ((val & 0xFF000000) >> 24);
}
```

**Option B: Standard hton/ntoh functions (POSIX, recommended):**

```c
#include <arpa/inet.h>  // POSIX (Linux, macOS)
// or <winsock2.h> on Windows

// hton = Host TO Network (converts to big-endian)
// ntoh = Network TO Host (converts from big-endian)

// Since Marathon data IS big-endian (network order):
header.version = ntohs(header.version);           // 16-bit: ntoh-short
header.directory_offset = ntohl(header.directory_offset);  // 32-bit: ntoh-long

// Note: htons/htonl and ntohs/ntohl are identical operations
// (swapping is symmetric), but ntoh* is semantically correct here
```

**Option C: Compiler intrinsics (fastest):**

```c
// GCC/Clang
#define swap16(x) __builtin_bswap16(x)
#define swap32(x) __builtin_bswap32(x)

// MSVC
#include <stdlib.h>
#define swap16(x) _byteswap_ushort(x)
#define swap32(x) _byteswap_ulong(x)
```

### Conditional Swap Macros

Wrap swapping so it's a no-op on big-endian systems:

```c
#if MARATHON_NEEDS_SWAP
    #define BE16(x) swap16(x)
    #define BE32(x) swap32(x)
#else
    #define BE16(x) (x)  // No swap needed on big-endian
    #define BE32(x) (x)
#endif

// Usage when loading
header.version = BE16(header.version);
header.directory_offset = BE32(header.directory_offset);
```

### When to Swap

| Field Size | Swap Function | Examples |
|------------|---------------|----------|
| 1 byte (byte, char) | No swap needed | flags, pixel indices |
| 2 bytes (short, word) | `BE16()` / `ntohs()` | version, count fields |
| 4 bytes (long, fixed) | `BE32()` / `ntohl()` | offsets, fixed-point coords |

---

## 10.6 Summary

Marathon's file formats are well-structured and portable:

**Marathon WAD Files:**
- Header → Level Data → Directory
- Tagged entries for each data type
- Custom format (NOT Doom WADs)

**Shape Files:**
- 32 collection slots
- Three-level hierarchy: High-Level → Low-Level → Bitmap
- RLE compression for sprites
- Palette-based colors

**Porting Strategy:**
- Replace Mac file APIs with `fopen`/`fread`
- Add byte swapping for little-endian systems
- Convert colors during load (8-bit palette → 32-bit ARGB)

### Key Constants

| Constant | Value | Purpose |
|----------|-------|---------|
| `MAXIMUM_COLLECTIONS` | 32 | Shape collection slots |
| WAD header size | 128 bytes | File header |
| Entry header size | 16 bytes | Data block header |
| Directory entry size | 10 bytes | Level index entry |

### Key Source Files

| File | Purpose |
|------|---------|
| `wad.c` | WAD file reading/writing |
| `game_wad.c` | Game-specific WAD handling |
| `shapes.c` | Shape loading and management |
| `tags.h` | All WAD tag definitions |

---

*Next: [Chapter 11: Performance and Optimization](11_performance.md) - Inner loops and optimization strategies*
