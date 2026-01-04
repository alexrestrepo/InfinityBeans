# Chapter 1: Introduction

## Welcome to the Marathon Engine

---

## 1.1 About Marathon 2 & Infinity

Marathon 2: Durandal and Marathon Infinity represent the pinnacle of Bungie's classic FPS trilogy, both built on the same sophisticated game engine.

### Marathon 2: Durandal
- **Released**: November 24, 1995
- **Developer**: Bungie Software
- **Platform**: Macintosh (68K and PowerPC)
- **Source Release**: January 2000 (GPL v2, incomplete)

### Marathon Infinity
- **Released**: October 15, 1996
- **Developer**: Bungie Software
- **Built on**: Marathon 2 engine with refinements
- **Source Release**: July 2011 (GPL v3, comprehensive)

### Technical Achievements

Both games feature:
- Full 3D environments with varying floor/ceiling heights
- True room-over-room geometry
- Portal-based visibility culling
- Deterministic peer-to-peer networking (up to 8 players)
- 30 Hz fixed-timestep physics
- Software texture mapping without FPU requirements

---

## 1.2 Source Code Statistics

**Code Metrics:**
- ~68,000 lines of C code (marathon2/)
- ~4,400 lines of utility code (cseries.lib/)
- 78 source files
- Built with MPW (Macintosh Programmer's Workshop)

**Key Files:**

| File | Lines | Purpose |
|------|-------|---------|
| `render.c` | 3,879 | Rendering pipeline |
| `map.c` | 3,456 | World representation |
| `physics.c` | 2,234 | Movement and collision |
| `scottish_textures.c` | 1,458 | Texture mapper |
| `monsters.c` | ~3,000 | AI and entity behavior |
| `weapons.c` | ~2,500 | Combat system |

---

## 1.3 About This Documentation

This documentation provides comprehensive technical analysis of the Marathon game engine, inspired by Fabien Sanglard's excellent "Black Book" series on Doom and Wolfenstein 3D.

### Goals

1. **Explain the engineering** - Document how Marathon achieved its technical feats on 1995 hardware
2. **Enable porting** - Provide clear guidance for bringing Marathon to modern platforms
3. **Teach by building** - Start with concepts, build to implementation

### How to Read These Chapters

Each chapter follows a consistent structure:

1. **What Problem Are We Solving?** - Motivation and constraints
2. **Understanding the Concept** - High-level explanation with diagrams
3. **Let's Build** - Simplified examples before the real code
4. **Marathon's Implementation** - The actual code with explanations
5. **Summary** - Key points and source file references

---

## 1.4 Marathon 2 vs. Infinity: What Changed?

While Marathon Infinity used the same core engine as Marathon 2, Bungie made several refinements.

### Engine Improvements

**Physics Models:**
- Infinity introduced selectable physics models per-level
- Maps could specify which physics model to use
- Allowed scenario designers to fine-tune gameplay

**Networking:**
- Refined netcode for better synchronization
- Improved lag compensation
- Better handling of packet loss

### Source Code Completeness

| Metric | Marathon 2 (2000) | Marathon Infinity (2011) |
|--------|-------------------|--------------------------|
| License | GPL v2 | GPL v3 |
| Completeness | Partial (edited) | Complete |
| Physics Models | 1 (hardcoded) | Multiple (data-driven) |

**Why Infinity Source Matters:**
The Infinity source release is the definitive reference because:
1. It's complete and unredacted
2. It represents the final, most refined version
3. Modern ports (Aleph One) are based on it

---

## 1.5 Core Design Principles

Marathon's engine is built on several fundamental principles that run throughout the codebase.

### Fixed-Point Math Everywhere

From `cseries.lib/cseries.h:110-122`:
```c
typedef long fixed;  // 16.16 fixed-point
#define FIXED_ONE ((fixed)(1<<FIXED_FRACTIONAL_BITS))  // 65536 = 1.0
```

No floating-point operations—crucial for:
- Deterministic behavior across platforms
- Performance on CPUs without FPUs
- Network consistency

### Fixed 30 Hz Timestep

From `map.h:13`:
```c
#define TICKS_PER_SECOND 30
```

Game logic runs at exactly 30 ticks/second:
- Physics deterministic
- Rendering decoupled
- Network-friendly

### Portal-Based Rendering

- No BSP trees (unlike Doom/Quake)
- Recursive visibility through portals
- Typical: 50-100 polygons visible out of 500-1000 total

### Deterministic Simulation

- Same inputs → same outputs
- Critical for networking
- Replay system possible

---

## 1.6 Chapter Overview

### Foundation (Chapters 1-3)
- **Chapter 1**: Introduction (this chapter)
- **Chapter 2**: Source Code Organization
- **Chapter 3**: Engine Overview

### Core Engine Systems (Chapters 4-6)
- **Chapter 4**: World Representation
- **Chapter 5**: Rendering System
- **Chapter 6**: Physics and Collision

### Game Systems (Chapters 7-9)
- **Chapter 7**: Game Loop and Timing
- **Chapter 8**: Entity Systems
- **Chapter 9**: Networking Architecture

### Data and I/O (Chapters 10-13)
- **Chapter 10**: File Formats
- **Chapter 11**: Performance and Optimization
- **Chapter 12**: Data Structures Appendix
- **Chapter 13**: Sound System

### Game Mechanics (Chapters 14-20)
- **Chapter 14**: Items & Inventory
- **Chapter 15**: Control Panels
- **Chapter 16**: Damage System
- **Chapter 17**: Multiplayer Game Types
- **Chapter 18**: Random Number Generation
- **Chapter 19**: Shape Animation System
- **Chapter 20**: Automap/Overhead Map

---

*Next: [Chapter 2: Source Code Organization](02_source_organization.md) - File structure and module dependencies*
