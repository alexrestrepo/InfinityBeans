# Chapter 14: Items & Inventory

## Pickups, Weapons, and Powerups

> **Source files**: `items.c`, `items.h`, `item_definitions.h`
> **Related chapters**: [Chapter 8: Entities](08_entities.md), [Chapter 15: Control Panels](15_control_panels.md)

> **For Porting:** The item system in `items.c` is fully portable. Item definitions are in `item_definitions.h` as static data tables. No Mac-specific code.

---

## 14.1 What Problem Are We Solving?

Marathon needs a system for items scattered around levels that players can collect:

- **Weapons** - New armaments to collect
- **Ammunition** - Refills for existing weapons
- **Powerups** - Temporary abilities (invisibility, invincibility)
- **Health** - Shield and oxygen restoration
- **Key items** - Required for progression
- **Balls** - Multiplayer game objects

**The constraints:**
- Items must be visible in the 3D world
- Pickup requires proximity check
- Some items invalid in certain environments
- Inventory limits per item type

---

## 14.2 Item Categories (`items.h:8-19`)

```c
enum /* item types (class) */
{
    _weapon,            // 0 - Pickable weapons
    _ammunition,        // 1 - Ammo pickups
    _powerup,           // 2 - Special abilities and health
    _item,              // 3 - Key items (keys, chips)
    _weapon_powerup,    // 4 - Extra ammo from weapon pickup
    _ball,              // 5 - Multiplayer game balls

    NUMBER_OF_ITEM_TYPES,
    _network_statistics= NUMBER_OF_ITEM_TYPES  // Used in game_window.c
};
```

---

## 14.3 Item Type Enum (`items.h:21-64`)

```c
enum /* item types */
{
    _i_knife,                       // 0
    _i_magnum,                      // 1
    _i_magnum_magazine,             // 2
    _i_plasma_pistol,               // 3
    _i_plasma_magazine,             // 4
    _i_assault_rifle,               // 5
    _i_assault_rifle_magazine,      // 6
    _i_assault_grenade_magazine,    // 7
    _i_missile_launcher,            // 8
    _i_missile_launcher_magazine,   // 9
    _i_invisibility_powerup,        // 10
    _i_invincibility_powerup,       // 11
    _i_infravision_powerup,         // 12
    _i_alien_shotgun,               // 13
    _i_alien_shotgun_magazine,      // 14
    _i_flamethrower,                // 15
    _i_flamethrower_canister,       // 16
    _i_extravision_powerup,         // 17
    _i_oxygen_powerup,              // 18
    _i_energy_powerup,              // 19
    _i_double_energy_powerup,       // 20
    _i_triple_energy_powerup,       // 21
    _i_shotgun,                     // 22
    _i_shotgun_magazine,            // 23
    _i_spht_door_key,               // 24
    _i_uplink_chip,                 // 25

    BALL_ITEM_BASE,                 // 26
    _i_light_blue_ball= BALL_ITEM_BASE,
    _i_red_ball,                    // 27
    _i_violet_ball,                 // 28
    _i_yellow_ball,                 // 29
    _i_brown_ball,                  // 30
    _i_orange_ball,                 // 31
    _i_blue_ball,                   // 32
    _i_green_ball,                  // 33

    _i_smg,                         // 34
    _i_smg_ammo,                    // 35

    NUMBER_OF_DEFINED_ITEMS         // 36
};
```

---

## 14.4 Item Definition Structure (`item_definitions.h:8-16`)

```c
struct item_definition
{
    short item_kind;               // Category (_weapon, _ammunition, etc.)
    short singular_name_id;        // String table ID for "a pistol"
    short plural_name_id;          // String table ID for "pistols"
    shape_descriptor base_shape;   // Visual appearance in world
    short maximum_count_per_player; // Inventory limit
    short invalid_environments;    // Bitfield of environments where item doesn't work
};
```

---

## 14.5 Item Definitions Array (`item_definitions.h:20-84`)

The static `item_definitions[]` array defines all 36 items. Here's a selection:

```c
struct item_definition item_definitions[]=
{
    /* Knife - always available, no pickup shape */
    {_weapon, 0, 0, NONE, 1, 0},

    /* Pistol and ammo */
    {_weapon, 1, 2, BUILD_DESCRIPTOR(_collection_items, 0), 2, 0},
    {_ammunition, 3, 4, BUILD_DESCRIPTOR(_collection_items, 3), 50, 0},

    /* Fusion pistol and battery */
    {_weapon, 5, 5, BUILD_DESCRIPTOR(_collection_items, 1), 1, 0},
    {_ammunition, 6, 7, BUILD_DESCRIPTOR(_collection_items, 4), 25, 0},

    /* Assault rifle - invalid in vacuum */
    {_weapon, 8, 8, BUILD_DESCRIPTOR(_collection_items, 2), 1, _environment_vacuum},
    {_ammunition, 9, 10, BUILD_DESCRIPTOR(_collection_items, 5), 15, _environment_vacuum},
    {_ammunition, 11, 12, BUILD_DESCRIPTOR(_collection_items, 6), 8, _environment_vacuum},

    /* Rocket launcher - invalid in vacuum */
    {_weapon, 13, 13, BUILD_DESCRIPTOR(_collection_items, 12), 1, _environment_vacuum},
    {_ammunition, 14, 15, BUILD_DESCRIPTOR(_collection_items, 7), 4, _environment_vacuum},

    /* Powerups */
    {_powerup, NONE, NONE, BUILD_DESCRIPTOR(_collection_items, 8), 1, 0},   // invisibility
    {_powerup, NONE, NONE, BUILD_DESCRIPTOR(_collection_items, 9), 1, 0},   // invincibility
    {_powerup, NONE, NONE, BUILD_DESCRIPTOR(_collection_items, 14), 1, 0},  // infravision

    /* Shotgun - can dual-wield (max 2) */
    {_weapon, 27, 28, BUILD_DESCRIPTOR(_collection_items, 18), 2, 0},
    {_ammunition, 17, 18, BUILD_DESCRIPTOR(_collection_items, 19), 80, 0},

    /* Key items */
    {_item, 29, 30, BUILD_DESCRIPTOR(_collection_items, 17), 8, 0},  // S'pht door key
    {_item, 31, 32, BUILD_DESCRIPTOR(_collection_items, 16), 1, 0},  // Uplink chip

    /* Multiplayer balls - single-player only environment restriction */
    {_ball, 33, 33, BUILD_DESCRIPTOR(BUILD_COLLECTION(_collection_player, 0), 29), 1, _environment_single_player},
    /* ... 7 more ball colors ... */

    /* SMG (Infinity addition) */
    {_weapon, 41, 41, BUILD_DESCRIPTOR(_collection_items, 25), 1, 0},
    {_ammunition, 42, 43, BUILD_DESCRIPTOR(_collection_items, 24), 8, 0},
};
```

---

## 14.6 Environment Flags (`map.h:672-679`)

Items can be restricted from certain environments:

```c
enum /* environment flags */
{
    _environment_normal= 0x0000,
    _environment_vacuum= 0x0001,        // prevents certain weapons, player uses oxygen
    _environment_magnetic= 0x0002,      // motion sensor works poorly
    _environment_rebellion= 0x0004,     // makes civilians fight pfhor
    _environment_low_gravity= 0x0008,   // low gravity

    _environment_network= 0x2000,       // item only in multiplayer
    _environment_single_player= 0x4000  // item only in single-player
};
```

The `item_valid_in_current_environment()` function checks this (`items.c:345-357`):

```c
boolean item_valid_in_current_environment(short item_type)
{
    boolean valid= TRUE;
    struct item_definition *definition= get_item_definition(item_type);

    if (definition->invalid_environments & static_world->environment_flags)
    {
        valid= FALSE;
    }

    return valid;
}
```

---

## 14.7 Pickup Constants (`items.c:42`)

```c
#define MAXIMUM_ARM_REACH (3*WORLD_ONE_FOURTH)  // ~768 units
```

This is the maximum distance from which a player can pick up an item (about 3/4 of a world unit, or roughly arm's length).

---

## 14.8 Item Pickup Flow

### Swipe Nearby Items (`items.c:259-313`)

Called each tick to check for pickable items:

```c
void swipe_nearby_items(short player_index)
{
    // Get player's location and polygon
    player_object= get_object_data(get_monster_data(player->monster_index)->object_index);
    polygon= get_polygon_data(player_object->polygon);

    // Check all neighboring polygons
    for (i=0; i<polygon->neighbor_count; ++i)
    {
        // For each object in neighboring polygon
        for (object in neighboring_polygon)
        {
            if (GET_OBJECT_OWNER(object)==_object_is_item && !OBJECT_IS_INVISIBLE(object))
            {
                // Distance check (2D first, then height)
                if (guess_distance2d(&player->location, &object->location) <= MAXIMUM_ARM_REACH)
                {
                    // Height check
                    if (object->location.z >= player->location.z - MAXIMUM_ARM_REACH &&
                        object->location.z <= player->location.z + height)
                    {
                        // Line-of-sight check
                        if (test_item_retrieval(polygon_index, &player_location, &object->location))
                        {
                            get_item(player_index, object_index);
                        }
                    }
                }
            }
        }
    }
}
```

### Pickup Flowchart

```
Player moves:
    │
    ├─► swipe_nearby_items() called
    │
    ├─► For each item in player's polygon and neighbors:
    │     │
    │     ├─► Check distance < MAXIMUM_ARM_REACH (~768 units)
    │     │
    │     ├─► Check height within reach
    │     │
    │     ├─► Check line-of-sight (no solid walls between)
    │     │
    │     └─► If all pass: get_item()
    │
    └─► get_item() calls try_and_add_player_item()
```

---

## 14.9 Adding Items to Inventory (`items.c:375-459`)

```c
boolean try_and_add_player_item(short player_index, short type)
{
    struct item_definition *definition= get_item_definition(type);
    struct player_data *player= get_player_data(player_index);
    short grabbed_sound_index= NONE;
    boolean success= FALSE;

    switch (definition->item_kind)
    {
        case _powerup:
            /* Powerups don't get added to inventory - they apply immediately */
            if (legal_player_powerup(player_index, type))
            {
                process_player_powerup(player_index, type);
                object_was_just_destroyed(_object_is_item, type);
                grabbed_sound_index= _snd_got_powerup;
                success= TRUE;
            }
            break;

        case _ball:
            /* Can only carry ONE ball at a time */
            if (find_player_ball_color(player_index)==NONE)
            {
                player->items[type]= 1;
                process_new_item_for_reloading(player_index, _i_red_ball);
                mark_player_inventory_as_dirty(player_index, type);
                success= TRUE;
            }
            grabbed_sound_index= NONE;
            break;

        case _weapon:
        case _ammunition:
        case _item:
            /* Check inventory limit */
            if (player->items[type]==NONE)
            {
                player->items[type]= 1;
                success= TRUE;
            }
            else if (player->items[type]+1 <= definition->maximum_count_per_player ||
                (difficulty==_total_carnage_level && definition->item_kind==_ammunition))
            {
                /* Total Carnage allows unlimited ammo */
                player->items[type]++;
                success= TRUE;
            }

            grabbed_sound_index= _snd_got_item;

            if (success)
            {
                process_new_item_for_reloading(player_index, type);
                mark_player_inventory_as_dirty(player_index, type);
            }
            break;
    }

    /* Play sound and flash screen on pickup */
    if (success && player_index==current_player_index)
    {
        play_local_sound(grabbed_sound_index);
        start_fade(_fade_bonus);
    }

    return success;
}
```

---

## 14.10 Item Removal (`items.c:491-508`)

When a player successfully picks up an item:

```c
static boolean get_item(short player_index, short object_index)
{
    struct object_data *object= get_object_data(object_index);

    assert(GET_OBJECT_OWNER(object)==_object_is_item);

    if (success= try_and_add_player_item(player_index, object->permutation))
    {
        /* Remove item from world */
        remove_map_object(object_index);
    }

    return success;
}
```

In multiplayer, items can respawn after a delay (handled by `placement.c`).

---

## 14.11 Creating Items (`items.c:67-123`)

Items are placed in the world via `new_item()`:

```c
short new_item(struct object_location *location, short type)
{
    struct item_definition *definition= get_item_definition(type);
    boolean add_item= TRUE;

    /* Environment validation */
    if (dynamic_world->player_count > 1)
    {
        /* Multiplayer: skip network-blocked items */
        if (definition->invalid_environments & _environment_network) add_item= FALSE;
        if (get_item_kind(type)==_ball && !current_game_has_balls()) add_item= FALSE;
    }
    else
    {
        /* Single-player: skip single-player-blocked items */
        if (definition->invalid_environments & _environment_single_player) add_item= FALSE;
    }

    if (add_item)
    {
        /* Add object to map */
        object_index= new_map_object(location, definition->base_shape);
        if (object_index != NONE)
        {
            SET_OBJECT_OWNER(object, _object_is_item);
            object->permutation= type;  /* Store item type */
        }
    }

    return object_index;
}
```

---

## 14.12 Complete Item Reference Table

| ID | Enum | Kind | Max | Invalid Envs | Notes |
|----|------|------|-----|--------------|-------|
| 0 | `_i_knife` | weapon | 1 | - | Always available (no pickup) |
| 1 | `_i_magnum` | weapon | 2 | - | Dual-wield capable |
| 2 | `_i_magnum_magazine` | ammunition | 50 | - | 8 rounds per magazine |
| 3 | `_i_plasma_pistol` | weapon | 1 | - | Fusion pistol |
| 4 | `_i_plasma_magazine` | ammunition | 25 | - | Energy cells |
| 5 | `_i_assault_rifle` | weapon | 1 | vacuum | MA-75B |
| 6 | `_i_assault_rifle_magazine` | ammunition | 15 | vacuum | 52 rounds |
| 7 | `_i_assault_grenade_magazine` | ammunition | 8 | vacuum | 7 grenades |
| 8 | `_i_missile_launcher` | weapon | 1 | vacuum | SPNKR |
| 9 | `_i_missile_launcher_magazine` | ammunition | 4 | vacuum | 2 rockets |
| 10 | `_i_invisibility_powerup` | powerup | 1 | - | ~30 seconds |
| 11 | `_i_invincibility_powerup` | powerup | 1 | - | ~30 seconds |
| 12 | `_i_infravision_powerup` | powerup | 1 | - | ~60 seconds |
| 13 | `_i_alien_shotgun` | weapon | 1 | - | Enemy weapon |
| 14 | `_i_alien_shotgun_magazine` | ammunition | 999 | - | No shape (NONE) |
| 15 | `_i_flamethrower` | weapon | 1 | vacuum | TOZT-7 |
| 16 | `_i_flamethrower_canister` | ammunition | 3 | vacuum | Napalm units |
| 17 | `_i_extravision_powerup` | powerup | 1 | - | Wide FOV |
| 18 | `_i_oxygen_powerup` | powerup | 1 | - | Refills O2 |
| 19 | `_i_energy_powerup` | powerup | 1 | - | 1x health restore |
| 20 | `_i_double_energy_powerup` | powerup | 1 | - | 2x health restore |
| 21 | `_i_triple_energy_powerup` | powerup | 1 | - | 3x health restore |
| 22 | `_i_shotgun` | weapon | 2 | - | Dual-wield capable |
| 23 | `_i_shotgun_magazine` | ammunition | 80 | - | Shotgun shells |
| 24 | `_i_spht_door_key` | item | 8 | - | Door key |
| 25 | `_i_uplink_chip` | item | 1 | - | Terminal access |
| 26-33 | `_i_*_ball` | ball | 1 | single_player | Multiplayer only |
| 34 | `_i_smg` | weapon | 1 | - | Submachine gun |
| 35 | `_i_smg_ammo` | ammunition | 8 | - | SMG magazines |

---

## 14.13 Summary

Marathon's item system provides:

- **6 item categories** covering all pickup types
- **36 item types** for complete gameplay
- **Proximity-based pickup** with arm reach check (~768 units)
- **Environment restrictions** for level variety
- **Inventory limits** per item type
- **Special handling** for powerups and balls

### Key Constants

| Constant | Value | Source |
|----------|-------|--------|
| `MAXIMUM_ARM_REACH` | 768 (3×WORLD_ONE_FOURTH) | `items.c:42` |
| `NUMBER_OF_ITEM_TYPES` | 6 | `items.h:17` |
| `NUMBER_OF_DEFINED_ITEMS` | 36 | `items.h:63` |
| `BALL_ITEM_BASE` | 26 | `items.h:50` |

### Key Source Files

| File | Purpose |
|------|---------|
| `items.c` | Item pickup logic, creation, validation |
| `items.h` | Item type enums, function prototypes |
| `item_definitions.h` | Static item definition table |
| `player.c` | Inventory management |
| `weapons.c` | `process_new_item_for_reloading()` |

---

## 14.14 See Also

- [Chapter 8: Entities](08_entities.md) — Items as world objects
- [Chapter 15: Control Panels](15_control_panels.md) — Switches and terminals
- [Chapter 16: Damage](16_damage.md) — How powerups affect damage

---

*Next: [Chapter 15: Control Panels](15_control_panels.md) - Switches, terminals, and doors*
