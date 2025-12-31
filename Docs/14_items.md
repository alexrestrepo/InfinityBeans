# Chapter 14: Items & Inventory

## Pickups, Weapons, and Powerups

> **For Porting:** The item system in `items.c` is fully portable. Item definitions are in `item_definitions.h` as static data tables.

---

## 14.1 What Problem Are We Solving?

Marathon needs a system for items scattered around levels that players can collect:

- **Weapons** - New armaments to collect
- **Ammunition** - Refills for existing weapons
- **Powerups** - Temporary abilities (invisibility, invincibility)
- **Health** - Shield and oxygen restoration
- **Key items** - Required for progression

---

## 14.2 Item Categories

```c
enum { /* item types (class) */
    _weapon,           // 0 - Pickable weapons
    _ammunition,       // 1 - Ammo pickups
    _powerup,          // 2 - Special abilities
    _item,             // 3 - Key items (keys, chips)
    _weapon_powerup,   // 4 - Extra ammo from weapon pickup
    _ball,             // 5 - Multiplayer game balls
    NUMBER_OF_ITEM_TYPES
};
```

---

## 14.3 Complete Item Types

| ID | Name | Kind | Max | Notes |
|----|------|------|-----|-------|
| 0 | Fist | weapon | 1 | Always available |
| 1 | Magnum Pistol | weapon | 2 | Dual-wield capable |
| 2 | Magnum Magazine | ammunition | 8 | 8 rounds |
| 3 | Fusion Pistol | weapon | 1 | Energy weapon |
| 4 | Fusion Battery | ammunition | 8 | 20 units |
| 5 | MA-75B Rifle | weapon | 1 | With grenade launcher |
| 6 | AR Magazine | ammunition | 8 | 52 rounds |
| 7 | AR Grenades | ammunition | 8 | 7 grenades |
| 8 | SPNKR Launcher | weapon | 1 | Rockets |
| 9 | Missile 2-Pack | ammunition | 4 | 2 rockets |
| 10 | Invisibility | powerup | 1 | ~30 seconds |
| 11 | Invincibility | powerup | 1 | ~30 seconds |
| 12 | Infravision | powerup | 1 | ~60 seconds |
| 13 | Alien Weapon | weapon | 2 | Enemy weapon |
| 15 | Flamethrower | weapon | 1 | TOZT-7 |
| 17 | Extravision | powerup | 1 | Wide FOV |
| 18 | Oxygen | item | 1 | Refills O2 |
| 19-21 | Energy x1/x2/x3 | item | 1 | Health restore |
| 22 | Shotgun | weapon | 2 | Dual-wield capable |
| 24 | S'pht Key | item | 1 | Door key |
| 25 | Uplink Chip | item | 1 | Terminal access |
| 26-33 | Balls | ball | 1 | Multiplayer |
| 34 | KKV-7 SMG | weapon | 1 | Submachine gun |

---

## 14.4 Item Definition Structure

```c
struct item_definition {
    short item_kind;               // Category (weapon, ammo, powerup)
    short singular_name_id;        // String resource ID
    short plural_name_id;          // String resource ID
    shape_descriptor base_shape;   // Visual appearance
    short maximum_count_per_player; // Inventory limit
    short invalid_environments;    // Where item cannot exist
};
```

### Environment Restrictions

```c
enum {
    _environment_normal = 0x0000,
    _environment_vacuum = 0x0001,        // No oxygen
    _environment_magnetic = 0x0002,      // Compass interference
    _environment_low_gravity = 0x0008,   // Reduced gravity
    _environment_network = 0x2000,       // Multiplayer only
    _environment_single_player = 0x4000  // Single-player only
};
```

---

## 14.5 Item Pickup System

### Pickup Detection

```c
#define MAXIMUM_ARM_REACH (3*WORLD_ONE_FOURTH)  // ~768 units

boolean try_and_get_item(short player_index, short polygon_index) {
    for (each object in polygon) {
        if (object is item && distance < MAXIMUM_ARM_REACH) {
            return get_item(player_index, object_index);
        }
    }
}
```

### Pickup Flow

```
Player approaches item:
    │
    ├─► Check distance < MAXIMUM_ARM_REACH
    │
    ├─► Validate environment compatibility
    │     └─► Network-only items blocked in single-player
    │
    ├─► Check inventory space
    │     └─► maximum_count_per_player limit
    │
    ├─► Add to inventory
    │     └─► Weapons: process_new_item_for_reloading()
    │     └─► Ammo: add to count
    │     └─► Powerups: start timer
    │
    ├─► Play pickup sound
    │
    └─► Remove item from world
        └─► May respawn later (multiplayer)
```

---

## 14.6 Powerup Timers

| Powerup | Duration | Effect |
|---------|----------|--------|
| Invisibility | ~30 seconds | Semi-transparent, AI ignores |
| Invincibility | ~30 seconds | No damage (except telefrag) |
| Infravision | ~60 seconds | See in darkness |
| Extravision | Permanent | 130° FOV |

---

## 14.7 Summary

Marathon's item system provides:

- **6 item categories** for different pickup types
- **38 item types** covering all gameplay needs
- **Simple distance check** for pickup detection
- **Environment restrictions** for level variety

### Key Source Files

| File | Purpose |
|------|---------|
| `items.c` | Item pickup logic |
| `item_definitions.h` | Item type data |
| `player.c` | Inventory management |

---

*Next: [Chapter 15: Control Panels](15_control_panels.md) - Switches, terminals, and doors*
