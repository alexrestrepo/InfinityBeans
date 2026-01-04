# Appendix I: Quick Reference Cheat Sheet

> A single-page reference for commonly-needed values, constants, and sizes.
> Print this out or keep it open while working with Marathon source code.

---

## Fixed-Point Math

```c
FIXED_ONE           = 65536      // 1.0 in 16.16 fixed-point
FIXED_ONE_HALF      = 32768      // 0.5
WORLD_ONE           = 1024       // 1 game unit in world coords
WORLD_ONE_HALF      = 512        // 0.5 game units
WORLD_ONE_FOURTH    = 256        // 0.25 game units

TRIG_SHIFT          = 10         // trig table precision
TRIG_MAGNITUDE      = 16384      // trig table scale (2^14)
NUMBER_OF_ANGLES    = 512        // angle units in full circle
QUARTER_CIRCLE      = 128        // 90 degrees
HALF_CIRCLE         = 256        // 180 degrees
FULL_CIRCLE         = 512        // 360 degrees

// conversions
degrees_to_marathon(d) = d * 512 / 360
marathon_to_degrees(a) = a * 360 / 512
fixed_to_float(f)      = f / 65536.0
float_to_fixed(f)      = (long)(f * 65536)
world_to_fixed(w)      = w << 6
fixed_to_world(f)      = f >> 6
```

---

## Map Limits

| Limit | Value | Memory |
|-------|-------|--------|
| `MAXIMUM_ENDPOINTS_PER_MAP` | 2048 | 32 KB |
| `MAXIMUM_LINES_PER_MAP` | 4096 | 128 KB |
| `MAXIMUM_SIDES_PER_MAP` | 8192 | 512 KB |
| `MAXIMUM_POLYGONS_PER_MAP` | 1024 | 128 KB |
| `MAXIMUM_VERTICES_PER_POLYGON` | 8 | — |
| `MAXIMUM_OBJECTS_PER_MAP` | 384 | — |
| `MAXIMUM_LIGHTS_PER_MAP` | 64 | — |
| `MAXIMUM_PLATFORMS_PER_MAP` | 64 | — |
| `MAXIMUM_MEDIAS_PER_MAP` | 16 | — |

---

## Entity Limits

| Limit | Value |
|-------|-------|
| `MAXIMUM_NUMBER_OF_PLAYERS` | 8 |
| `MAXIMUM_MONSTERS_PER_MAP` | 220 |
| `MAXIMUM_PROJECTILES_PER_MAP` | 32 |
| `MAXIMUM_EFFECTS_PER_MAP` | 64 |
| `NUMBER_OF_MONSTER_TYPES` | 47 |
| `NUMBER_OF_PROJECTILE_TYPES` | 43 |
| `NUMBER_OF_EFFECT_TYPES` | 60 |
| `MAXIMUM_NUMBER_OF_WEAPONS` | 10 |
| `NUMBER_OF_ITEMS` | 64 |

---

## Structure Sizes (bytes)

| Structure | Size | Source |
|-----------|------|--------|
| `endpoint_data` | 16 | `map.h:378` |
| `line_data` | 32 | `map.h:416` |
| `side_data` | 64 | `map.h:499` |
| `polygon_data` | 128 | `map.h:571` |
| `object_data` | 32 | `map.h:229` |
| `player_data` | ~500 | `player.h` |
| `monster_data` | 64 | `monsters.h` |
| `projectile_data` | 32 | `projectiles.h` |
| `platform_data` | 32 | `platforms.h` |
| `light_data` | 32 | `lightsource.h` |
| `media_data` | 32 | `media.h` |

---

## Timing

```c
TICKS_PER_SECOND    = 30         // game tick rate
TICK_MS             = 33.33      // milliseconds per tick
MACHINE_TICKS_PER_SECOND = 60    // Mac VBL rate

// common durations
TICKS_PER_MINUTE    = 1800
TICKS_PER_5_SECONDS = 150
```

---

## File Type Codes (FourCC)

| Type | Code | Description |
|------|------|-------------|
| `SCENARIO_FILE_TYPE` | `'sce2'` | Map/scenario file |
| `SAVE_GAME_TYPE` | `'sga∞'` | Saved game |
| `FILM_FILE_TYPE` | `'fil∞'` | Recording/replay |
| `PHYSICS_FILE_TYPE` | `'phy∞'` | Physics model |
| `SHAPES_FILE_TYPE` | `'shp∞'` | Shapes/sprites |
| `SOUNDS_FILE_TYPE` | `'snd∞'` | Sound effects |
| `PATCH_FILE_TYPE` | `'pat2'` | Physics patch |

---

## WAD Tags (Map Data)

| Tag | FourCC | Contents |
|-----|--------|----------|
| `POINT_TAG` | `'PNTS'` | Endpoints |
| `LINE_TAG` | `'LINS'` | Lines |
| `SIDE_TAG` | `'SIDS'` | Sides (wall textures) |
| `POLYGON_TAG` | `'POLY'` | Polygons |
| `OBJECT_TAG` | `'OBJS'` | Objects (items/monsters) |
| `LIGHTSOURCE_TAG` | `'LITE'` | Lights |
| `PLATFORM_STATIC_DATA_TAG` | `'plat'` | Platforms |
| `MEDIA_TAG` | `'medi'` | Liquids |
| `TERMINAL_DATA_TAG` | `'term'` | Terminals |
| `MAP_INFO_TAG` | `'Minf'` | Map metadata |
| `ENDPOINT_DATA_TAG` | `'EPNT'` | Extended endpoints |

---

## Physics Tags

| Tag | FourCC | Contents |
|-----|--------|----------|
| `MONSTER_PHYSICS_TAG` | `'MNpx'` | Monster definitions |
| `EFFECTS_PHYSICS_TAG` | `'FXpx'` | Effect definitions |
| `PROJECTILE_PHYSICS_TAG` | `'PRpx'` | Projectile definitions |
| `PHYSICS_PHYSICS_TAG` | `'PXpx'` | Player physics |
| `WEAPONS_PHYSICS_TAG` | `'WPpx'` | Weapon definitions |

---

## Player Physics Defaults

| Constant | Walking | Running |
|----------|---------|---------|
| Forward velocity | `1/14` | `1/8` |
| Backward velocity | `1/17` | `1/12` |
| Strafe velocity | `1/20` | `1/13` |
| Acceleration | `1/200` | `1/100` |
| Gravity | `1/400` | `1/400` |
| Terminal velocity | `1/7` | `1/7` |
| Turn speed (max) | `6.0` | `10.0` |
| Player radius | `0.25` | `0.25` |
| Player height | `0.8` | `0.8` |
| Camera height | `0.2` | `0.2` |

*(All values as fractions of FIXED_ONE)*

---

## Shape Collections

| ID | Name | Contents |
|----|------|----------|
| 0 | `_collection_walls` | Wall textures |
| 1 | `_collection_scenery` | Scenery objects |
| 2-5 | `_collection_landscape` | Sky/landscape |
| 6 | `_collection_weapons_in_hand` | First-person weapons |
| 7 | `_collection_interface` | HUD elements |
| 8 | `_collection_player` | Player sprites |
| 9 | `_collection_items` | Pickup items |
| 10-31 | `_collection_monsters` | Monster sprites |

---

## Difficulty Levels

| Value | Name | Monster multiplier |
|-------|------|-------------------|
| 0 | Kindergarten | 0.25× |
| 1 | Easy | 0.5× |
| 2 | Normal | 1.0× |
| 3 | Major Damage | 1.5× |
| 4 | Total Carnage | 2.0× |

---

## Damage Types

| ID | Name | Description |
|----|------|-------------|
| 0 | `_damage_explosion` | Grenade, rocket |
| 1 | `_damage_electrical_staff` | Alien weapon |
| 2 | `_damage_projectile` | Bullets |
| 3 | `_damage_absorbed` | Shield damage |
| 4 | `_damage_flame` | Fire |
| 5 | `_damage_hound_claws` | Melee |
| 6 | `_damage_alien_projectile` | Pfhor bolts |
| 7 | `_damage_hulk_slap` | Heavy melee |
| 8 | `_damage_compiler_bolt` | Compiler attack |
| 9 | `_damage_fusion_bolt` | Fusion pistol |
| 10 | `_damage_hunter_bolt` | Hunter attack |
| 11 | `_damage_fist` | Punch |
| 12 | `_damage_teleporter` | Teleport damage |
| 13 | `_damage_defender` | Defender attack |
| 14 | `_damage_yeti_claws` | Yeti melee |
| 15 | `_damage_yeti_projectile` | Yeti ranged |
| 16 | `_damage_crushing` | Platform crush |
| 17 | `_damage_lava` | Lava damage |
| 18 | `_damage_suffocation` | Drowning |
| 19 | `_damage_goo` | Goo damage |
| 20 | `_damage_energy_drain` | Energy weapon |
| 21 | `_damage_oxygen_drain` | Vacuum |
| 22 | `_damage_hummer_bolt` | Hummer attack |
| 23 | `_damage_shotgun_projectile` | Shotgun |

---

## Media Types

| ID | Name | Damage | Frequency |
|----|------|--------|-----------|
| 0 | `_media_water` | None | — |
| 1 | `_media_lava` | 16 | Every 16 ticks |
| 2 | `_media_goo` | 8 | Every 8 ticks |
| 3 | `_media_sewage` | None | — |
| 4 | `_media_jjaro` | None | — |

---

## Weapon IDs

| ID | Name | Dual-wield |
|----|------|------------|
| 0 | Fist | No |
| 1 | Pistol | Yes |
| 2 | Fusion Pistol | No (chargeable) |
| 3 | Assault Rifle | No (+grenade) |
| 4 | Rocket Launcher | No |
| 5 | Flamethrower | No |
| 6 | Alien Weapon | No |
| 7 | Shotgun | Yes |
| 8 | Ball | No |
| 9 | SMG | No |

---

## Action Flag Bits

```c
// movement
_moving_forward         = bit 15
_moving_backward        = bit 16
_sidestepping_left      = bit 22
_sidestepping_right     = bit 23
_run_dont_walk          = bit 17

// turning
_turning_left           = bit 1
_turning_right          = bit 2
_looking_up             = bit 9
_looking_down           = bit 10

// actions
_left_trigger_state     = bit 24  // primary fire
_right_trigger_state    = bit 25  // secondary fire
_action_trigger_state   = bit 26  // use/activate
_cycle_weapons_forward  = bit 27
_cycle_weapons_backward = bit 28
_toggle_map             = bit 29
```

---

## Quick Formulas

```c
// distance between two points
distance = integer_square_root((x2-x1)² + (y2-y1)²)

// angle from point1 to point2
angle = arctangent((y2-y1), (x2-x1))

// check if point in polygon (simplified)
for each line in polygon:
    cross = (line.dx * (point.y - line.y0)) - (line.dy * (point.x - line.x0))
    if cross < 0: return false
return true

// light intensity to height (media)
height = low + ((high - low) * light_intensity / FIXED_ONE)
```

---

## Common Byte Order Swaps

```c
// 16-bit swap
swap16(x) = ((x & 0xFF) << 8) | ((x >> 8) & 0xFF)

// 32-bit swap
swap32(x) = ((x & 0xFF) << 24) | ((x & 0xFF00) << 8) |
            ((x >> 8) & 0xFF00) | ((x >> 24) & 0xFF)
```

---

## Key Source File Locations

| Topic | File | Key Line |
|-------|------|----------|
| Fixed-point defs | `world.h` | 1 |
| Map structures | `map.h` | 378, 416, 499, 571 |
| Player physics | `physics_models.h` | 17 |
| Monster defs | `monster_definitions.h` | 1 |
| Weapon defs | `weapon_definitions.h` | 1 |
| WAD format | `wad.h` | 1 |
| Tags | `tags.h` | 1 |
| Rendering | `render.c` | 1 |
| Physics engine | `physics.c` | 1 |
| Game loop | `marathon2.c` | 1 |
