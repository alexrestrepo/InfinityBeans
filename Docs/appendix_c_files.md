# Appendix C: Source File Index

## File-by-File Reference

---

## marathon2/ Directory

### Core Game Files

| File | Lines | Purpose | Portable |
|------|-------|---------|----------|
| `marathon2.c` | ~480 | Main game initialization, `update_world()` | YES |
| `shell.c` | ~1500 | Application shell, event loop | NO (Mac) |
| `interface.c` | ~1400 | Game state machine, menu handling | Partial |

### Rendering System

| File | Lines | Purpose | Portable |
|------|-------|---------|----------|
| `render.c` | ~3900 | 3D rendering engine, visibility | YES |
| `screen.c` | ~900 | Screen management, `render_screen()` | Partial |
| `screen_drawing.c` | ~400 | 2D drawing utilities | NO (Mac) |
| `scottish_textures.c` | ~1500 | Texture mapping algorithms | YES |
| `low_level_textures.c` | ~200 | Pixel-level texture functions | YES |
| `textures.c` | ~200 | Texture management | YES |
| `overhead_map.c` | ~400 | Automap rendering logic | YES |
| `overhead_map_macintosh.c` | ~200 | Automap Mac specifics | NO (Mac) |
| `fades.c` | ~400 | Screen fade effects | Partial |

### Map System

| File | Lines | Purpose | Portable |
|------|-------|---------|----------|
| `map.c` | ~1200 | Map loading, structure management | YES |
| `map_accessors.c` | ~300 | Map data access functions | YES |
| `map_constructors.c` | ~400 | Map building utilities | YES |
| `preprocess_map_mac.c` | ~200 | Mac map preprocessing | NO (Mac) |
| `world.c` | ~200 | World initialization, trig tables | YES |

### Game Objects

| File | Lines | Purpose | Portable |
|------|-------|---------|----------|
| `player.c` | ~2200 | Player state, physics, weapons | YES |
| `monsters.c` | ~3500 | Monster AI, movement, combat | YES |
| `projectiles.c` | ~1200 | Projectile physics, collision | YES |
| `effects.c` | ~400 | Visual effects (explosions, etc.) | YES |
| `scenery.c` | ~200 | Static/animated decorations | YES |
| `items.c` | ~400 | Item pickups, spawning | YES |

### Physics System

| File | Lines | Purpose | Portable |
|------|-------|---------|----------|
| `physics.c` | ~900 | Movement physics, collision | YES |
| `physics_patches.c` | ~100 | Physics modifications | YES |
| `pathfinding.c` | ~300 | Monster pathfinding | YES |
| `flood_map.c` | ~400 | Flood fill algorithms | YES |

### Environment Systems

| File | Lines | Purpose | Portable |
|------|-------|---------|----------|
| `platforms.c` | ~800 | Moving platforms, doors | YES |
| `lightsource.c` | ~400 | Light animation | YES |
| `media.c` | ~300 | Liquids (water, lava, etc.) | YES |

### Weapons

| File | Lines | Purpose | Portable |
|------|-------|---------|----------|
| `weapons.c` | ~2000 | Weapon logic, firing, ammunition | YES |

### Interface

| File | Lines | Purpose | Portable |
|------|-------|---------|----------|
| `game_window.c` | ~400 | HUD logic | YES |
| `game_window_macintosh.c` | ~600 | HUD Mac rendering | NO (Mac) |
| `interface_macintosh.c` | ~400 | Mac interface code | NO (Mac) |
| `motion_sensor.c` | ~300 | Radar display | YES |
| `computer_interface.c` | ~800 | Terminal system | YES |

### Shapes/Graphics

| File | Lines | Purpose | Portable |
|------|-------|---------|----------|
| `shapes.c` | ~600 | Shape management | YES |
| `shapes_macintosh.c` | ~650 | Mac shape loading | NO (Mac) |
| `images.c` | ~400 | Interface images | Partial |

### Sound/Music

| File | Lines | Purpose | Portable |
|------|-------|---------|----------|
| `game_sound.c` | ~800 | Sound playback logic | Partial |
| `sound_macintosh.c` | ~800 | Mac Sound Manager | NO (Mac) |
| `music.c` | ~400 | Music playback | NO (Mac) |

### Network

| File | Lines | Purpose | Portable |
|------|-------|---------|----------|
| `network.c` | ~600 | Network game logic | YES |
| `network_games.c` | ~400 | Network game modes | YES |
| `network_dialogs.c` | ~600 | Network UI | NO (Mac) |
| `network_ddp.c` | ~400 | AppleTalk DDP | NO (Mac) |
| `network_adsp.c` | ~400 | AppleTalk ADSP | NO (Mac) |
| `network_lookup.c` | ~300 | Network discovery | NO (Mac) |
| `network_names.c` | ~200 | Player names | YES |
| `network_speaker.c` | ~200 | Voice chat | NO (Mac) |
| `network_microphone.c` | ~200 | Voice input | NO (Mac) |
| `network_modem.c` | ~400 | Modem support | NO (Mac) |
| `network_modem_protocol.c` | ~300 | Modem protocol | NO (Mac) |
| `network_stream.c` | ~200 | Stream handling | Partial |

### Files/Data

| File | Lines | Purpose | Portable |
|------|-------|---------|----------|
| `wad.c` | ~800 | WAD file reading | YES |
| `wad_macintosh.c` | ~200 | Mac WAD specifics | NO (Mac) |
| `wad_prefs.c` | ~200 | WAD preferences | YES |
| `wad_prefs_macintosh.c` | ~100 | Mac WAD prefs | NO (Mac) |
| `game_wad.c` | ~800 | Game state WADs | YES |
| `files_macintosh.c` | ~400 | Mac file system | NO (Mac) |
| `find_files.c` | ~200 | File enumeration | Partial |

### Input

| File | Lines | Purpose | Portable |
|------|-------|---------|----------|
| `vbl.c` | ~400 | Input handling logic | Partial |
| `vbl_macintosh.c` | ~300 | Mac VBL interrupt | NO (Mac) |
| `mouse.c` | ~200 | Mouse input | Partial |
| `keyboard_dialog.c` | ~300 | Key configuration | NO (Mac) |

### Preferences/Settings

| File | Lines | Purpose | Portable |
|------|-------|---------|----------|
| `preferences.c` | ~300 | Game preferences | Partial |
| `serial_numbers.c` | ~100 | Serial validation | NO (Mac) |

### Devices

| File | Lines | Purpose | Portable |
|------|-------|---------|----------|
| `devices.c` | ~200 | Control panel interactions | YES |

### Definition Files

| File | Lines | Purpose | Portable |
|------|-------|---------|----------|
| `export_definitions.c` | ~100 | Definition export | YES |
| `import_definitions.c` | ~200 | Definition import | YES |

### Utilities

| File | Lines | Purpose | Portable |
|------|-------|---------|----------|
| `game_errors.c` | ~50 | Error handling | YES |
| `progress.c` | ~140 | Progress dialogs | NO (Mac) |
| `crc.c` | ~100 | CRC checksums | YES |
| `placement.c` | ~200 | Object placement | YES |
| `game_dialogs.c` | ~300 | Game dialogs | NO (Mac) |

### Special Hardware

| File | Lines | Purpose | Portable |
|------|-------|---------|----------|
| `valkyrie.c` | ~300 | Valkyrie accelerator | NO (Mac) |

### Extract Tools

| File | Lines | Purpose | Portable |
|------|-------|---------|----------|
| `extract/shapeextract.c` | ~200 | Shape extraction | YES |
| `extract/sndextract.c` | ~200 | Sound extraction | YES |

---

## cseries.lib/ Directory

| File | Lines | Purpose | Portable |
|------|-------|---------|----------|
| `cseries.h` | ~500 | Core types, macros | YES |
| `macintosh_cseries.h` | ~200 | Mac-specific types | NO (Mac) |
| `byte_swapping.c` | ~100 | Endian conversion | YES |
| `checksum.c` | ~100 | Data validation | YES |
| `rle.c` | ~150 | Run-length encoding | YES |
| `proximity_strcmp.c` | ~100 | Fuzzy string compare | YES |
| `mytm.c` | ~200 | Timer management | NO (Mac) |
| `my32bqd.c` | ~200 | 32-bit QuickDraw | NO (Mac) |
| `devices.c` | ~200 | Device management | NO (Mac) |
| `device_dialog.c` | ~200 | Device UI | NO (Mac) |
| `dialogs.c` | ~300 | Dialog utilities | NO (Mac) |
| `preferences.c` | ~300 | Preference files | NO (Mac) |
| `macintosh_utilities.c` | ~300 | Mac utilities | NO (Mac) |
| `macintosh_interfaces.c` | ~100 | Mac interfaces | NO (Mac) |

---

## Portability Summary

### Fully Portable (60+ files)

These files need only header changes and can compile on any platform:

- `render.c`, `scottish_textures.c` - Core rendering
- `monsters.c`, `player.c`, `projectiles.c` - Game logic
- `physics.c`, `pathfinding.c` - Physics/AI
- `map.c`, `wad.c` - Data loading
- `weapons.c`, `items.c`, `effects.c` - Game objects
- All definition/data files

### Partial (Need Abstraction Layer)

These files need platform abstraction but logic is portable:

- `screen.c` - Replace framebuffer handling
- `game_sound.c` - Replace audio API
- `vbl.c` - Replace input/timing
- `fades.c` - Replace color manipulation

### Mac-Specific (Replace Entirely)

These files must be replaced for porting:

- `*_macintosh.c` - 11 files total
- Network files using AppleTalk
- Dialog/UI files using Mac toolbox
- File system using FSSpec

---

## File Counts by Category

| Category | Files | Lines (approx) |
|----------|-------|----------------|
| Rendering | 10 | 7,500 |
| Game Logic | 8 | 10,000 |
| Map/World | 5 | 2,500 |
| Sound/Music | 3 | 2,000 |
| Network | 12 | 3,500 |
| Interface | 8 | 3,000 |
| Files/Data | 7 | 2,500 |
| cseries.lib | 14 | 3,000 |
| **Total** | **~78** | **~68,000** |

---

*Next: [Appendix D: Fixed-Point Conversion](appendix_d_fixedpoint.md) - Working with Marathon's number format*
