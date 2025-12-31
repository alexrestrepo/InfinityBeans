# Marathon Engine: Technical Documentation

## Learning the Engine

This directory contains tutorial-style documentation for the Marathon 2/Infinity engine, written in the style of "Crafting Interpreters" - teaching concepts progressively from simple building blocks to the actual implementation.

**How to use these chapters:**
1. Each chapter starts with "What problem are we solving?"
2. Builds understanding with simplified examples and diagrams
3. Shows pseudocode → modern C → Marathon's actual approach
4. Includes source file:line references to the real code

---

## Table of Contents

### Foundation (Chapters 1-3)

| # | Chapter | Status | Description |
|---|---------|--------|-------------|
| 1 | [**Introduction**](01_introduction.md) | ✅ Complete | What is Marathon? History, goals of this documentation |
| 2 | [**Source Code Organization**](02_source_organization.md) | ✅ Complete | File structure, module dependencies, build system |
| 3 | [**Engine Overview**](03_engine_overview.md) | ✅ Complete | High-level architecture, main loop, subsystem interactions |

### Core Engine Systems (Chapters 4-6)

| # | Chapter | Status | Description |
|---|---------|--------|-------------|
| 4 | [**World Representation**](04_world.md) | ✅ Complete | Polygon soup, line ownership, map structure, endpoints |
| 5 | [**Rendering System**](05_rendering.md) | ✅ Complete | Portal-based visibility, 3D projection, texture mapping |
| 6 | [**Physics and Collision**](06_physics.md) | ✅ Complete | Fixed-point math, collision detection, movement physics |

### Game Systems (Chapters 7-9)

| # | Chapter | Status | Description |
|---|---------|--------|-------------|
| 7 | [**Game Loop and Timing**](07_game_loop.md) | ✅ Complete | 30 Hz tick system, input processing, game state |
| 8 | [**Entity Systems**](08_entities.md) | ✅ Complete | Monsters, AI state machines, pathfinding |
| 9 | [**Networking Architecture**](09_networking.md) | ✅ Complete | Deterministic simulation, peer-to-peer sync |

### Data and I/O (Chapters 10-13)

| # | Chapter | Status | Description |
|---|---------|--------|-------------|
| 10 | [**File Formats**](10_file_formats.md) | ✅ Complete | WAD files, shape collections, sound data |
| 11 | [**Performance and Optimization**](11_performance.md) | ✅ Complete | Inner loop optimization, fixed-point tricks |
| 12 | [**Data Structures Appendix**](12_data_structures.md) | ✅ Complete | Complete struct reference |
| 13 | [**Sound System**](13_sound.md) | ✅ Complete | 3D audio, sound channels, ambient sounds |

### Game Mechanics (Chapters 14-20)

| # | Chapter | Status | Description |
|---|---------|--------|-------------|
| 14 | [**Items & Inventory**](14_items.md) | ✅ Complete | Pickups, weapons, powerups |
| 15 | [**Control Panels**](15_control_panels.md) | ✅ Complete | Switches, terminals, doors |
| 16 | [**Damage System**](16_damage.md) | ✅ Complete | Hit detection, damage types, death |
| 17 | [**Multiplayer Game Types**](17_multiplayer.md) | ✅ Complete | Deathmatch, co-op, scoring |
| 18 | [**Random Number Generation**](18_random.md) | ✅ Complete | Deterministic RNG for networking |
| 19 | [**Shape Animation System**](19_shapes.md) | ✅ Complete | Sprite animation, sequences |
| 20 | [**Automap/Overhead Map**](20_automap.md) | ✅ Complete | Map rendering, annotations |

### Interface & Effects (Chapters 21-24)

| # | Chapter | Status | Description |
|---|---------|--------|-------------|
| 21 | [**HUD Rendering System**](21_hud.md) | ✅ Complete | Motion sensor, energy bars, weapons |
| 22 | [**Screen Effects & Fades**](22_fades.md) | ✅ Complete | Damage flashes, environmental tints |
| 23 | [**View Bobbing & Camera**](23_camera.md) | ✅ Complete | FOV, pitch limits, weapon sway |
| 24 | [**cseries.lib Utility Library**](24_cseries.md) | ✅ Complete | Core types, macros, debugging |

### Environment Systems (Chapters 25-29)

| # | Chapter | Status | Description |
|---|---------|--------|-------------|
| 25 | [**Media/Liquid System**](25_media.md) | ✅ Complete | Water, lava, goo, currents |
| 26 | [**Visual Effects System**](26_effects.md) | ✅ Complete | Explosions, blood, particles |
| 27 | [**Scenery Objects**](27_scenery.md) | ✅ Complete | Decorations, destructibles |
| 28 | [**Computer Terminal System**](28_terminals.md) | ✅ Complete | Story content, checkpoints |
| 29 | [**Music/Soundtrack System**](29_music.md) | ✅ Complete | Background music, fades |

### Engine Internals (Chapters 30-32)

| # | Chapter | Status | Description |
|---|---------|--------|-------------|
| 30 | [**Error Handling & Progress**](30_errors.md) | ✅ Complete | Game errors, progress dialogs |
| 31 | [**Resource Forks Guide**](31_resource_forks.md) | ✅ Complete | Mac file format handling |
| 32 | [**Life of a Frame**](32_frame.md) | ✅ Complete | Complete frame lifecycle walkthrough |

### Appendices

| # | Appendix | Status | Description |
|---|----------|--------|-------------|
| A | [**Glossary**](appendix_a_glossary.md) | ✅ Complete | Definitions of key terms |
| B | [**Quick Reference**](appendix_b_reference.md) | ✅ Complete | Common values, formulas, limits |
| C | [**Source File Index**](appendix_c_files.md) | ✅ Complete | File-by-file reference |
| D | [**Fixed-Point Conversion**](appendix_d_fixedpoint.md) | ✅ Complete | Working with Marathon's number format |

---

## Quick Reference

For quick lookup of specific values and formulas, see:
- **[Appendix B: Quick Reference](appendix_b_reference.md)** - Constants, limits, and formulas
- **[Appendix A: Glossary](appendix_a_glossary.md)** - Definitions of key terms
- **[Appendix C: Source File Index](appendix_c_files.md)** - File-by-file reference

For porting Marathon to modern platforms, see:
- **[porting_progress.md](porting_progress.md)** - Step-by-step porting plan with milestones
- **[Chapter 31: Resource Forks Guide](31_resource_forks.md)** - Clarifies what files need special handling
- **[Chapter 24: cseries.lib](24_cseries.md)** - Core type definitions with modern stdint equivalents

---

## Learning Paths

Different readers have different goals. Here are recommended reading paths:

### 🎮 "I want to port Marathon to a new platform"

**Essential reading (in order):**
1. **[Chapter 24: cseries.lib](24_cseries.md)** - Type definitions, stdint equivalents
2. **[Chapter 10: File Formats](10_file_formats.md)** - WAD and shape file parsing
3. **[Chapter 31: Resource Forks](31_resource_forks.md)** - What files need special handling
4. **[Chapter 5: Rendering](05_rendering.md)** - The core rendering pipeline
5. **[Appendix D: Fixed-Point](appendix_d_fixedpoint.md)** - Number format conversion
6. **[porting_progress.md](porting_progress.md)** - Milestone-by-milestone guide

### 🔧 "I want to understand how a 1990s game engine works"

**Foundation path:**
1. **[Chapter 3: Engine Overview](03_engine_overview.md)** - Architecture and main loop
2. **[Chapter 4: World Representation](04_world.md)** - Polygon soup and map structure
3. **[Chapter 5: Rendering](05_rendering.md)** - Portal culling and texture mapping
4. **[Chapter 6: Physics](06_physics.md)** - Collision detection
5. **[Chapter 32: Life of a Frame](32_frame.md)** - Complete frame walkthrough

### 👾 "I want to understand Marathon's gameplay systems"

**Gameplay path:**
1. **[Chapter 7: Game Loop](07_game_loop.md)** - 30 Hz tick system
2. **[Chapter 8: Entity Systems](08_entities.md)** - Monsters, weapons, AI
3. **[Chapter 16: Damage System](16_damage.md)** - Combat mechanics
4. **[Chapter 9: Networking](09_networking.md)** - Deterministic multiplayer

### 🛠 "I want to mod or create content for Marathon"

**Content creation path:**
1. **[Chapter 10: File Formats](10_file_formats.md)** - Map and shape formats
2. **[Chapter 19: Shape Animation](19_shapes.md)** - Sprite and animation system
3. **[Chapter 28: Terminals](28_terminals.md)** - Story content markup
4. **[Chapter 13: Sound](13_sound.md)** - Audio system

### 📚 "I want to read everything"

**Complete path (chapter order is designed for this):**
- Start with Chapter 1 and read sequentially through Chapter 32
- Use Appendices A-D as reference while reading

---

## Chapter Format

Each chapter follows this structure:

```
## X.1 What Problem Are We Solving?
Brief motivation - why does Marathon need this system?

## X.2 Understanding [Concept]
High-level explanation with diagrams.
No code yet - just mental model building.

## X.3 Let's Build: A Simple Version
Standalone code snippet that demonstrates the core concept.
Pseudocode → Modern C → "Marathon's approach" callouts.

## X.4 Marathon's [Full Implementation]
Real code from the source with explanations.
Source file:line references throughout.

## X.5 Summary
Key points and source file reference tables.
```

---

## Contributing / Editing Chapters

When editing or adding chapters:

1. **Follow the established numbering** from existing chapters
2. **Start with the problem** - what does this system do?
3. **Build incrementally** - simple examples before complex reality
4. **Include source references** - `file.c:123` format for all code
5. **Add diagrams** - ASCII art diagrams explaining concepts
6. **Update this README** - add the chapter to the table above

### Chapter Verification Checklist

**IMPORTANT**: Before marking a chapter complete, verify content accuracy:

- [ ] **All diagrams** - Ensure each ASCII diagram is clear and complete
- [ ] **All tables** - Check every table has all rows AND all columns
- [ ] **All code blocks** - Verify comments, struct fields, enum values match source
- [ ] **All source references** - Every `file.c:line` reference should be accurate
- [ ] **Tutorial flow** - Content should build understanding progressively

**Verification method**:
```bash
# Search source code for key terms, verify they match chapter
grep -n "keyword" marathon2/*.c | head -20
grep -n "keyword" Docs/XX_chapter.md
```

---

## Key Source Files Quick Reference

| File | Lines | Primary Topics |
|------|-------|----------------|
| `render.c` | 3,879 | Visibility, projection, render tree |
| `scottish_textures.c` | 1,200+ | Texture mapping routines |
| `physics.c` | ~800 | Movement, collision detection |
| `monsters.c` | 3,000+ | AI, monster behavior |
| `map.c` | ~1,500 | Level geometry, polygon data |
| `world.c` | ~400 | Coordinate transforms, trig tables |
| `weapons.c` | ~1,200 | Weapon mechanics |
| `projectiles.c` | ~600 | Projectile physics |

---

*These chapters are derived from analysis of the original Marathon 2/Infinity source code, released under GPL by Bungie.*
