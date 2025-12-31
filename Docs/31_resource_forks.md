# Chapter 31: Resource Forks Guide

## Mac-Specific File Format Handling for Porters

> **For Porting:** This chapter clarifies a common misconception. **Most Marathon files DON'T use resource forks!** Shapes, Sounds, and Maps are all standard binary files readable with `fopen`/`fread`.

---

## 31.1 What Are Resource Forks?

Classic Macintosh files have two parts:

```
┌─────────────────────────────────────────────────────────────────┐
│                    MACINTOSH FILE STRUCTURE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                     DATA FORK                            │   │
│   │                                                          │   │
│   │   Sequential bytes, like files on other platforms        │   │
│   │   Read with: fopen(), fread(), FSRead()                  │   │
│   │   Open with: FSpOpenDF() (Data Fork)                     │   │
│   │                                                          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    RESOURCE FORK                         │   │
│   │                                                          │   │
│   │   Structured storage with typed resources                │   │
│   │   Read with: GetResource(), Get1Resource()               │   │
│   │   Open with: FSpOpenRF() (Resource Fork)                 │   │
│   │                                                          │   │
│   │   Contains: PICT, snd, STR#, DLOG, etc.                  │   │
│   │                                                          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 31.2 The Critical Discovery

**IMPORTANT**: There's widespread confusion about Marathon's file formats. Here's the truth from the source code:

### Proof from shapes_macintosh.c:299-305

```c
void open_shapes_file(FSSpec *spec)
{
    short refNum;
    OSErr error;

    error = FSpOpenDF(spec, fsRdPerm, &refNum);  // <-- Opens DATA FORK!
    // ...
}
```

`FSpOpenDF` = **Open Data Fork**. Not resource fork!

---

## 31.3 File Format Summary

| File | Format | Resource Fork? | Standard I/O Works? |
|------|--------|----------------|---------------------|
| **Shapes16** | Binary data fork | NO | YES |
| **Shapes8** | Binary data fork | NO | YES |
| **Sounds16** | Binary data fork | NO | YES |
| **Sounds8** | Binary data fork | NO | YES |
| **Map files** | Marathon WAD | NO | YES |
| **Saved games** | Marathon WAD | NO | YES |
| **Images file** | Mac resource fork | YES | Extract or stub |
| **Scenario files** | Mac resource fork | YES | Optional |
| **Music** | QuickTime/rsrc | YES | Optional |

### What This Means for Porting

```
Essential for gameplay:
  ✓ Shapes   - Standard binary file, fopen() works
  ✓ Sounds   - Standard binary file, fopen() works
  ✓ Maps     - Standard binary file, fopen() works

Optional:
  ✗ Images   - Interface graphics only
  ✗ Music    - Background music only
  ✗ Scenario - Custom campaigns only
```

**You can port Marathon without ANY resource fork handling!**

---

## 31.4 Why the Confusion Exists

The confusion stems from Marathon's development history:

```
1994-1995 Development:
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   Original Plan: Store everything in resource forks             │
│   (Standard Mac approach at the time)                           │
│                                                                  │
│   Problem: Resource forks have 16MB size limit                  │
│   (Shapes file is ~30MB for 16-bit graphics)                    │
│                                                                  │
│   Solution: Store large data in DATA forks                      │
│   Keep only interface graphics in resource fork                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

Result:
  • Core game data → Data forks (portable!)
  • Interface graphics → Resource fork (Images file only)
```

---

## 31.5 Reading Marathon's Data Fork Files

### Simple Cross-Platform Reading

```c
// This works on ALL platforms!
FILE *fp = fopen("Shapes16", "rb");
if (fp) {
    // Read collection headers
    struct collection_header headers[MAXIMUM_COLLECTIONS];
    fread(headers, sizeof(struct collection_header), MAXIMUM_COLLECTIONS, fp);

    // Seek and read collection data
    fseek(fp, headers[0].offset, SEEK_SET);
    // ... read collection

    fclose(fp);
}
```

### Byte Order Consideration

Marathon data is **big-endian** (Mac PowerPC/68K). On x86/ARM:

```c
uint16_t swap16(uint16_t val) {
    return (val << 8) | (val >> 8);
}

uint32_t swap32(uint32_t val) {
    return ((val & 0xFF) << 24) | ((val & 0xFF00) << 8) |
           ((val & 0xFF0000) >> 8) | ((val & 0xFF000000) >> 24);
}

// Usage:
header.offset = swap32(header.offset);
header.length = swap32(header.length);
```

---

## 31.6 Shapes File Format (Data Fork)

```
┌─────────────────────────────────────────────────────────────────┐
│                    SHAPES FILE STRUCTURE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Offset 0:                                                      │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  collection_header[32]  (32 headers × 12 bytes = 384)   │   │
│   │    • offset (4 bytes) - Position of 8-bit data          │   │
│   │    • length (4 bytes) - Size of 8-bit data              │   │
│   │    • offset16 (4 bytes) - Position of 16-bit data       │   │
│   │    • length16 (4 bytes) - Size of 16-bit data           │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   Variable offsets:                                              │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  collection_definition[0] (8-bit graphics)              │   │
│   │    • color tables                                        │   │
│   │    • bitmap data                                         │   │
│   │    • shape definitions                                   │   │
│   └─────────────────────────────────────────────────────────┘   │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  collection_definition[0] (16-bit graphics)             │   │
│   │    • Same structure, higher color depth                 │   │
│   └─────────────────────────────────────────────────────────┘   │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  collection_definition[1]...                            │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 31.7 Sounds File Format (Data Fork)

```
┌─────────────────────────────────────────────────────────────────┐
│                    SOUNDS FILE STRUCTURE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Header:                                                        │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  version (2 bytes)                                       │   │
│   │  tag (2 bytes) - 'snd2' for Marathon 2                   │   │
│   │  source_count (2 bytes)                                  │   │
│   │  sound_count (2 bytes)                                   │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   Sound definitions:                                             │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  sound_definition[sound_count]                           │   │
│   │    • sound_code                                          │   │
│   │    • behavior_index                                      │   │
│   │    • flags                                               │   │
│   │    • chance                                              │   │
│   │    • low/high_pitch                                      │   │
│   │    • permutations                                        │   │
│   │    • permutations_played                                 │   │
│   │    • group_offset (position in file)                     │   │
│   │    • single_length, total_length                         │   │
│   │    • sound_offsets[]                                     │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   Audio data:                                                    │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Raw audio samples (8-bit or 16-bit PCM)                │   │
│   │  Referenced by sound_definition.sound_offsets           │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 31.8 If You Must Handle Resource Forks

For the Images file (interface graphics), you have options:

### Option 1: Pre-Extract (Recommended)

```bash
# On macOS, extract PICTs from Images file:
DeRez -only PICT "Images" > images.r

# Or use Python/tool to extract raw bitmap data
# and convert to PNG during build process
```

### Option 2: Stub Out

```c
// For initial porting, just skip interface graphics
boolean load_interface_graphics(void) {
    // Return success but don't actually load
    // Game will run without title screens
    return TRUE;
}
```

### Option 3: Parse Resource Fork Binary

Resource fork structure (if you really need it):

```
┌─────────────────────────────────────────────────────────────────┐
│                    RESOURCE FORK FORMAT                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Offset 0: Resource Header (16 bytes)                          │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  data_offset (4 bytes) - Offset to resource data        │   │
│   │  map_offset (4 bytes) - Offset to resource map          │   │
│   │  data_length (4 bytes) - Length of data section         │   │
│   │  map_length (4 bytes) - Length of map section           │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   Data Section (at data_offset):                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  For each resource:                                      │   │
│   │    length (4 bytes)                                      │   │
│   │    data (length bytes)                                   │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   Map Section (at map_offset):                                   │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  header_copy (16 bytes)                                  │   │
│   │  next_map (4 bytes)                                      │   │
│   │  file_ref (2 bytes)                                      │   │
│   │  attributes (2 bytes)                                    │   │
│   │  type_list_offset (2 bytes)                              │   │
│   │  name_list_offset (2 bytes)                              │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   Type List (at map_offset + type_list_offset):                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  type_count-1 (2 bytes)                                  │   │
│   │  For each type:                                          │   │
│   │    type (4 bytes) - e.g., 'PICT', 'snd '                │   │
│   │    count-1 (2 bytes)                                     │   │
│   │    ref_list_offset (2 bytes)                             │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   Reference List:                                                │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  For each resource of type:                              │   │
│   │    id (2 bytes)                                          │   │
│   │    name_offset (2 bytes)                                 │   │
│   │    attributes (1 byte)                                   │   │
│   │    data_offset (3 bytes) - Offset in data section       │   │
│   │    handle (4 bytes)                                      │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 31.9 Resource Types in Images File

| Type Code | Description | Usage |
|-----------|-------------|-------|
| `PICT` | QuickDraw picture | Title screens, backgrounds |
| `clut` | Color lookup table | UI palettes |
| `cicn` | Color icon | UI icons |
| `CURS` | Cursor | Mouse cursors |

---

## 31.10 macOS Resource Fork Access

On modern macOS, resource forks are still accessible:

```bash
# View resource fork of a file
xattr -l "Images"

# Access resource fork directly via path
cat "Images/..namedfork/rsrc" > images_rsrc.bin

# Or use the POSIX extended attribute
# Resource fork is stored as: com.apple.ResourceFork
```

In code:

```c
// macOS: Open resource fork as regular file
FILE *fp = fopen("Images/..namedfork/rsrc", "rb");
// Or use POSIX xattr APIs
```

---

## 31.11 Decision Tree for Porters

```
Do I need to parse resource forks?
│
├─► Are you porting core gameplay?
│       │
│       └─► NO - Shapes, Sounds, Maps are data fork files!
│               Use standard fopen/fread
│
├─► Do you need interface graphics (title screens)?
│       │
│       ├─► No → Stub out, game still playable
│       │
│       └─► Yes → Pre-extract PICTs to PNG/BMP
│                 Load with your image library
│
└─► Do you need custom scenarios/music?
        │
        └─► Skip for initial port
            Add later if needed
```

---

## 31.12 Summary

**Key Takeaways:**

1. **Marathon's core files are NOT resource fork files**
2. `FSpOpenDF` in source code proves data fork usage
3. Shapes, Sounds, Maps all use standard binary format
4. Only Images file (optional) needs resource fork handling
5. You can port Marathon without ANY Mac-specific file handling

### Porting Recommendations

| Priority | File | Action |
|----------|------|--------|
| **1** | Shapes | Read data fork with fopen, swap bytes |
| **2** | Sounds | Read data fork with fopen, swap bytes |
| **3** | Maps | Read data fork with fopen, swap bytes |
| **4** | Images | Pre-extract or stub out |
| **Later** | Music | Optional, use replacement audio |

### Key Source Files

| File | Purpose |
|------|---------|
| `shapes_macintosh.c` | Shape loading (uses FSpOpenDF!) |
| `sound_macintosh.c` | Sound loading |
| `wad.c` | WAD file reading |
| `files_macintosh.c` | File system abstraction |

---

*Next: [Chapter 32: Life of a Frame](32_frame.md) - Complete frame lifecycle walkthrough*
