# Cross-Reference Index

> Systematic links between chapters organized by topic. Use this to find related information across the documentation.

---

## By System

### Rendering Pipeline
| Topic | Primary | Related |
|-------|---------|---------|
| Portal rendering | [Ch 5: Rendering](05_rendering.md) | [Ch 4: World](04_world.md), [Ch 32: Frame](32_frame.md) |
| Texture mapping | [Ch 5: Rendering](05_rendering.md) | [Ch 19: Shapes](19_shapes.md), [Ch 11: Performance](11_performance.md) |
| Shading/lighting | [Ch 5: Rendering](05_rendering.md) | [Ch 4: World](04_world.md) (lights) |
| HUD display | [Ch 21: HUD](21_hud.md) | [Ch 32: Frame](32_frame.md) |
| Screen effects | [Ch 22: Fades](22_fades.md) | [Ch 21: HUD](21_hud.md), [Ch 25: Media](25_media.md) |
| Camera/view | [Ch 23: Camera](23_camera.md) | [Ch 5: Rendering](05_rendering.md) |
| Automap | [Ch 20: Automap](20_automap.md) | [Ch 4: World](04_world.md), [Ch 21: HUD](21_hud.md) |

### Physics & Collision
| Topic | Primary | Related |
|-------|---------|---------|
| Movement physics | [Ch 6: Physics](06_physics.md) | [App G: Physics File](appendix_g_physics_file.md) |
| Collision detection | [Ch 6: Physics](06_physics.md) | [Ch 4: World](04_world.md), [Ch 8: Entities](08_entities.md) |
| Fixed-point math | [App D: Fixed-Point](appendix_d_fixedpoint.md) | [Ch 6: Physics](06_physics.md), [Ch 5: Rendering](05_rendering.md) |
| Gravity/falling | [Ch 6: Physics](06_physics.md) | [App G: Physics File](appendix_g_physics_file.md) |
| Platform movement | [Ch 4: World](04_world.md) | [Ch 6: Physics](06_physics.md) |

### Game Logic
| Topic | Primary | Related |
|-------|---------|---------|
| Game loop | [Ch 7: Game Loop](07_game_loop.md) | [Ch 32: Frame](32_frame.md), [Ch 9: Networking](09_networking.md) |
| Monster AI | [Ch 8: Entities](08_entities.md) | [Ch 6: Physics](06_physics.md), [Ch 16: Damage](16_damage.md) |
| Weapons | [Ch 8: Entities](08_entities.md) | [Ch 16: Damage](16_damage.md), [App G: Physics File](appendix_g_physics_file.md) |
| Projectiles | [Ch 8: Entities](08_entities.md) | [Ch 6: Physics](06_physics.md), [Ch 26: Effects](26_effects.md) |
| Items/pickups | [Ch 14: Items](14_items.md) | [Ch 8: Entities](08_entities.md) |
| Damage system | [Ch 16: Damage](16_damage.md) | [Ch 8: Entities](08_entities.md), [Ch 25: Media](25_media.md) |
| Random numbers | [Ch 18: Random](18_random.md) | [Ch 9: Networking](09_networking.md), [Ch 8: Entities](08_entities.md) |

### Networking & Recording
| Topic | Primary | Related |
|-------|---------|---------|
| Network sync | [Ch 9: Networking](09_networking.md) | [Ch 7: Game Loop](07_game_loop.md), [Ch 18: Random](18_random.md) |
| Action flags | [Ch 9: Networking](09_networking.md) | [App H: Film Format](appendix_h_film_format.md) |
| Film recording | [App H: Film Format](appendix_h_film_format.md) | [Ch 9: Networking](09_networking.md) |
| Multiplayer modes | [Ch 17: Multiplayer](17_multiplayer.md) | [Ch 9: Networking](09_networking.md) |

### Data & Files
| Topic | Primary | Related |
|-------|---------|---------|
| WAD format | [Ch 10: File Formats](10_file_formats.md) | [Ch 31: Resource Forks](31_resource_forks.md) |
| Map loading | [Ch 10: File Formats](10_file_formats.md) | [Ch 4: World](04_world.md) |
| Shapes/sprites | [Ch 19: Shapes](19_shapes.md) | [App L: Shapes Format](appendix_l_shapes_format.md), [Ch 5: Rendering](05_rendering.md) |
| Shapes file parsing | [App L: Shapes Format](appendix_l_shapes_format.md) | [Ch 19: Shapes](19_shapes.md), [Ch 10: File Formats](10_file_formats.md) |
| Sound files | [Ch 13: Sound](13_sound.md) | [Ch 10: File Formats](10_file_formats.md) |
| Physics files | [App G: Physics File](appendix_g_physics_file.md) | [Ch 10: File Formats](10_file_formats.md) |
| Resource forks | [Ch 31: Resource Forks](31_resource_forks.md) | [Ch 10: File Formats](10_file_formats.md) |
| Save games | [App K: Save Game Format](appendix_k_savegame_format.md) | [Ch 10: File Formats](10_file_formats.md), [Ch 12: Data Structures](12_data_structures.md) |
| Film recordings | [App H: Film Format](appendix_h_film_format.md) | [Ch 9: Networking](09_networking.md) |

### Environment
| Topic | Primary | Related |
|-------|---------|---------|
| Liquids/media | [Ch 25: Media](25_media.md) | [Ch 16: Damage](16_damage.md), [Ch 6: Physics](06_physics.md) |
| Scenery objects | [Ch 27: Scenery](27_scenery.md) | [Ch 8: Entities](08_entities.md) |
| Terminals | [Ch 28: Terminals](28_terminals.md) | [Ch 15: Control Panels](15_control_panels.md) |
| Control panels | [Ch 15: Control Panels](15_control_panels.md) | [Ch 4: World](04_world.md) |
| Visual effects | [Ch 26: Effects](26_effects.md) | [Ch 8: Entities](08_entities.md) |

### Audio
| Topic | Primary | Related |
|-------|---------|---------|
| Sound system | [Ch 13: Sound](13_sound.md) | [Ch 10: File Formats](10_file_formats.md) |
| Music | [Ch 29: Music](29_music.md) | [Ch 13: Sound](13_sound.md) |
| Ambient sounds | [Ch 13: Sound](13_sound.md) | [Ch 4: World](04_world.md) |

---

## By Source File

| File | Topics Covered In |
|------|-------------------|
| `render.c` | [Ch 5](05_rendering.md), [Ch 32](32_frame.md) |
| `scottish_textures.c` | [Ch 5](05_rendering.md), [Ch 11](11_performance.md) |
| `map.c`, `map.h` | [Ch 4](04_world.md), [Ch 10](10_file_formats.md) |
| `physics.c` | [Ch 6](06_physics.md), [App G](appendix_g_physics_file.md) |
| `marathon2.c` | [Ch 7](07_game_loop.md), [Ch 3](03_engine_overview.md) |
| `monsters.c` | [Ch 8](08_entities.md) |
| `projectiles.c` | [Ch 8](08_entities.md), [Ch 16](16_damage.md) |
| `weapons.c` | [Ch 8](08_entities.md) |
| `network.c` | [Ch 9](09_networking.md), [Ch 17](17_multiplayer.md) |
| `vbl.c` | [Ch 7](07_game_loop.md), [App H](appendix_h_film_format.md) |
| `wad.c` | [Ch 10](10_file_formats.md) |
| `shapes.c` | [Ch 19](19_shapes.md), [Ch 10](10_file_formats.md), [App L](appendix_l_shapes_format.md) |
| `shape_definitions.h` | [Ch 19](19_shapes.md), [App L](appendix_l_shapes_format.md) |
| `collection_definition.h` | [App L](appendix_l_shapes_format.md) |
| `game_wad.c` | [Ch 10](10_file_formats.md), [App K](appendix_k_savegame_format.md) |
| `sound.c` | [Ch 13](13_sound.md) |
| `media.c` | [Ch 25](25_media.md) |
| `platforms.c` | [Ch 4](04_world.md) |
| `lightsource.c` | [Ch 4](04_world.md), [Ch 5](05_rendering.md) |
| `player.c`, `player.h` | [Ch 6](06_physics.md), [Ch 9](09_networking.md), [App H](appendix_h_film_format.md) |
| `items.c` | [Ch 14](14_items.md) |
| `world.h` | [App D](appendix_d_fixedpoint.md), [App I](appendix_i_cheatsheet.md) |
| `tags.h` | [Ch 10](10_file_formats.md), [App G](appendix_g_physics_file.md), [App I](appendix_i_cheatsheet.md) |
| `extensions.h` | [App G](appendix_g_physics_file.md) |
| `vbl_definitions.h` | [App H](appendix_h_film_format.md) |

---

## By Task

### "I want to..."

| Task | Start Here | Also See |
|------|------------|----------|
| Understand how rendering works | [Ch 5: Rendering](05_rendering.md) | [Ch 32: Frame](32_frame.md) |
| Parse map files | [Ch 10: File Formats](10_file_formats.md) | [Ch 4: World](04_world.md) |
| Parse save game files | [App K: Save Game Format](appendix_k_savegame_format.md) | [Ch 10: File Formats](10_file_formats.md) |
| Parse shapes files | [App L: Shapes Format](appendix_l_shapes_format.md) | [Ch 19: Shapes](19_shapes.md) |
| Modify monster behavior | [Ch 8: Entities](08_entities.md) | [App G: Physics File](appendix_g_physics_file.md) |
| Change weapon stats | [App G: Physics File](appendix_g_physics_file.md) | [Ch 8: Entities](08_entities.md) |
| Understand network sync | [Ch 9: Networking](09_networking.md) | [Ch 18: Random](18_random.md) |
| Parse film recordings | [App H: Film Format](appendix_h_film_format.md) | [Ch 9: Networking](09_networking.md) |
| Add new textures | [Ch 19: Shapes](19_shapes.md) | [App L: Shapes Format](appendix_l_shapes_format.md) |
| Understand the game loop | [Ch 7: Game Loop](07_game_loop.md) | [Ch 32: Frame](32_frame.md) |
| Port to new platform | [CLAUDE.md](../CLAUDE.md) | [Ch 31: Resource Forks](31_resource_forks.md), [Porting Progress](porting_progress.md) |
| Understand fixed-point | [App D: Fixed-Point](appendix_d_fixedpoint.md) | [App I: Cheat Sheet](appendix_i_cheatsheet.md) |
| Quick lookup of constants | [App I: Cheat Sheet](appendix_i_cheatsheet.md) | [App B: Reference](appendix_b_reference.md) |
| Mod the game | [App J: Cookbook](appendix_j_cookbook.md) | [App G](appendix_g_physics_file.md), [Ch 10](10_file_formats.md) |

---

## Concept Dependencies

Understanding these topics requires prior knowledge:

```
┌─────────────────────────────────────────────────────────┐
│                    FOUNDATION                           │
│  Fixed-Point (App D) → All numeric code                 │
│  World Structure (Ch 4) → Rendering, Physics, AI        │
│  File Formats (Ch 10) → All data loading                │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                    CORE SYSTEMS                         │
│  Rendering (Ch 5) ← World, Fixed-Point                  │
│  Physics (Ch 6) ← World, Fixed-Point                    │
│  Game Loop (Ch 7) ← All systems                         │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                  GAME MECHANICS                         │
│  Entities (Ch 8) ← Physics, World                       │
│  Damage (Ch 16) ← Entities                              │
│  Items (Ch 14) ← Entities, World                        │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                  ADVANCED TOPICS                        │
│  Networking (Ch 9) ← Game Loop, Random                  │
│  Film Format (App H) ← Networking                       │
│  Physics File (App G) ← Entities, File Formats          │
└─────────────────────────────────────────────────────────┘
```

---

## Diagram Index

| Diagram | Location | Topics Illustrated |
|---------|----------|-------------------|
| `monster_states.svg` | `diagrams/` | AI state machine, target modes |
| `polygon_winding.svg` | `diagrams/` | Clockwise rule, line ownership |
| `portal_rendering.svg` | `diagrams/` | Visibility culling |
| `side_types.svg` | `diagrams/` | Wall texture types |
| `exclusion_zones.svg` | `diagrams/` | Collision detection |
| `game_loop.svg` | `diagrams/` | 30 tick/sec phases |
| `wad_structure.svg` | `diagrams/` | WAD file layout |
| `frame_lifecycle.svg` | `diagrams/` | Render pipeline stages |
| `weapon_states.svg` | `diagrams/` | Weapon state machine |
| `fixed_point.svg` | `diagrams/` | 16.16 bit layout |
| `shape_collection.svg` | `diagrams/` | Sprite memory layout |
| `coordinate_systems.svg` | `diagrams/` | World/screen/texture transforms |
| `platform_states.svg` | `diagrams/` | Moving platform states |
| `network_action_queue.svg` | `diagrams/` | P2P sync, buffering |
| `media_layers.svg` | `diagrams/` | Liquid system |
| `map_topology.svg` | `diagrams/` | Polygon/line/endpoint hierarchy |
| `shading_tables.svg` | `diagrams/` | Pre-computed lighting |

---

## Glossary Cross-Reference

See [Appendix A: Glossary](appendix_a_glossary.md) for term definitions.

| Term | Defined In | Used Extensively In |
|------|-----------|---------------------|
| Portal | App A | Ch 5, Ch 32 |
| WAD | App A | Ch 10, App G, App H, App K |
| Fixed-point | App A, App D | Ch 5, Ch 6, App I |
| Action flags | App A | Ch 9, App H |
| CLUT | App A | Ch 19, Ch 5, App L |
| Tick | App A | Ch 7, Ch 9, App H |
| Shading table | App A | Ch 5, Ch 11, App L |
| Determinism | App A | Ch 9, Ch 18, App H |
| Collection | App A | Ch 19, App L |
| RLE | App A | App L |
