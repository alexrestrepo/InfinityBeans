# Chapter 16: Damage System

## Hit Detection, Damage Types, and Combat

> **Source files**: `map.h`, `marathon2.c`, `monsters.c`, `player.c`, `monster_definitions.h`
> **Related chapters**: [Chapter 8: Entities](08_entities.md), [Chapter 6: Physics](06_physics.md)

> **For Porting:** The damage system is fully portable. Damage definitions are static data, and all calculation functions use only fixed-point integer math.

---

## 16.1 What Problem Are We Solving?

Marathon needs a flexible damage system that handles:

- **Multiple damage types** with different effects
- **Monster immunities and weaknesses**
- **Difficulty scaling** for different skill levels
- **Visual feedback** through screen effects
- **Sound feedback** for pain and death

**The constraints:**
- All math must be deterministic for network sync
- Must support 24 different damage types
- Must scale with difficulty levels
- Must provide appropriate audiovisual feedback

---

## 16.2 Damage Definition Structure (`map.h:72-78`)

```c
struct damage_definition
{
    short type, flags;

    short base, random;
    fixed scale;
};
```

| Field | Purpose |
|-------|---------|
| `type` | Damage type enum (0-23) |
| `flags` | Modifier flags (e.g., `_alien_damage`) |
| `base` | Base damage amount |
| `random` | Additional random damage [0, random) |
| `scale` | Fixed-point multiplier (FIXED_ONE = 1.0) |

---

## 16.3 Damage Types Enum (`map.h:39-65`)

```c
enum /* damage types */
{
    _damage_explosion,           // 0 - Rockets, grenades
    _damage_electrical_staff,    // 1 - Staff weapon
    _damage_projectile,          // 2 - Bullets
    _damage_absorbed,            // 3 - Shield blocked
    _damage_flame,               // 4 - Flamethrower
    _damage_hound_claws,         // 5 - Hound melee
    _damage_alien_projectile,    // 6 - Alien weapons
    _damage_hulk_slap,           // 7 - Hulk melee
    _damage_compiler_bolt,       // 8 - Compiler attack
    _damage_fusion_bolt,         // 9 - Fusion pistol
    _damage_hunter_bolt,         // 10 - Hunter shot
    _damage_fist,                // 11 - Player punch
    _damage_teleporter,          // 12 - Telefrag
    _damage_defender,            // 13 - Defender attack
    _damage_yeti_claws,          // 14 - Yeti melee
    _damage_yeti_projectile,     // 15 - Yeti shot
    _damage_crushing,            // 16 - Platform/door
    _damage_lava,                // 17 - Lava contact
    _damage_suffocation,         // 18 - No oxygen
    _damage_goo,                 // 19 - Sewage/goo
    _damage_energy_drain,        // 20 - Shield drain
    _damage_oxygen_drain,        // 21 - O2 drain
    _damage_hummer_bolt,         // 22 - Hummer attack
    _damage_shotgun_projectile   // 23 - Shotgun pellets
};
```

---

## 16.4 Damage Flags (`map.h:67-70`)

```c
enum /* damage flags */
{
    _alien_damage= 0x1  /* will be decreased at lower difficulty levels */
};
```

The `_alien_damage` flag marks damage from enemies, allowing the engine to scale it down on easier difficulty levels.

---

## 16.5 Damage Calculation (`marathon2.c:378-397`)

```c
short calculate_damage(struct damage_definition *damage)
{
    short total_damage= damage->base + (damage->random ? random()%damage->random : 0);

    total_damage= FIXED_INTEGERAL_PART(total_damage*damage->scale);

    /* if this damage was caused by an alien modify it for the current difficulty level */
    if (damage->flags&_alien_damage)
    {
        switch (dynamic_world->game_information.difficulty_level)
        {
            case _wuss_level: total_damage-= total_damage>>1; break;  /* -50% */
            case _easy_level: total_damage-= total_damage>>2; break;  /* -25% */
            /* harder levels do not cause more damage */
        }
    }

    return total_damage;
}
```

### Calculation Pipeline

```
Damage Calculation:

  base (100) ─────┐
                  │
  random (50) ────┼──► Roll: 100 + rand(0,49) = 125
                  │
                  ▼
            ┌─────────────┐
            │ Apply Scale │  scale = 1.5 (FIXED_ONE*3/2)
            └─────────────┘
                  │
                  ▼
            125 × 1.5 = 187
                  │
                  ▼
            ┌─────────────┐
            │  Difficulty │  (if _alien_damage flag set)
            │  Modifier   │
            └─────────────┘
                  │
       ┌──────────┼──────────┐
    Wuss       Normal     Major
    -50%        0%         0%
       │          │          │
      93        187        187
```

---

## 16.6 Monster Damage Application (`monsters.c:1279-1329`)

```c
void damage_monster(
    short target_index,
    short aggressor_index,
    short aggressor_type,
    world_point3d *epicenter,
    struct damage_definition *damage)
{
    struct monster_data *monster= get_monster_data(target_index);
    struct monster_definition *definition= get_monster_definition(monster->type);
    short delta_vitality= calculate_damage(damage);

    /* Immunity check */
    if (!(definition->immunities&FLAG(damage->type)))
    {
        /* Weakness check - double damage */
        if (definition->weaknesses&FLAG(damage->type)) delta_vitality<<= 1;

        /* Apply damage to player differently */
        if (MONSTER_IS_PLAYER(monster))
        {
            damage_player(target_index, aggressor_index, aggressor_type, damage);
        }
        else
        {
            /* Activate sleeping monsters when hit */
            if (!MONSTER_IS_ACTIVE(monster)) activate_monster(target_index);

            /* Apply damage */
            if ((monster->vitality-= delta_vitality)>0)
            {
                set_monster_action(target_index, _monster_is_being_hit);
                /* Trigger berserk at 25% health */
                if ((definition->flags&_monster_is_berserker) &&
                    monster->vitality<(definition->vitality>>2))
                {
                    SET_MONSTER_BERSERK_STATUS(monster, TRUE);
                }
            }
            /* Death handling follows... */
        }
    }
}
```

---

## 16.7 Monster Immunities and Weaknesses

Monster definitions include bitfields for damage immunity and weakness (`monster_definitions.h:147-157`):

```c
struct monster_definition /* <128 bytes */
{
    short collection;

    short vitality;
    unsigned long immunities, weaknesses;  /* Bitfields of damage types */
    unsigned long flags;

    long monster_class;
    long friends, enemies;
    /* ... more fields ... */
};
```

### Example Monster Stats (`monster_definitions.h:196+`)

| Monster | Vitality | Immunities | Weaknesses |
|---------|----------|------------|------------|
| Marine | 20 | none | none |
| Compiler Minor | 160 | flame, lava | fusion_bolt |
| Compiler Major | 200 | flame, lava | fusion_bolt |
| Hunter Minor | 200 | flame | fusion_bolt |
| Hunter Major | 300 | flame | fusion_bolt |
| Enforcer Minor | 120 | none | none |
| Enforcer Major | 160 | none | none |
| Cyborg Minor | 300 | none | fusion_bolt |
| Cyborg Major | 450 | none | fusion_bolt |
| Lava Yeti | 200 | flame, alien_proj, fusion, lava | none |
| Juggernaut Minor | 2500 | none | fusion_bolt |
| Juggernaut Major | 5000 | none | fusion_bolt |

**Immunity**: Monster takes NO damage from that type.
**Weakness**: Monster takes DOUBLE damage from that type.

---

## 16.8 Damage Response Definition (`player.c:88-95`)

```c
struct damage_response_definition
{
    short type;               /* Damage type this responds to */
    short damage_threshhold;  /* NONE or threshold for enhanced fade */

    short fade;               /* Screen fade color index */
    short sound, death_sound, death_action;
};
```

---

## 16.9 Damage Response Definitions Array (`player.c:130-156`)

```c
#define NUMBER_OF_DAMAGE_RESPONSE_DEFINITIONS \
    (sizeof(damage_response_definitions)/sizeof(struct damage_response_definition))

static struct damage_response_definition damage_response_definitions[]=
{
    {_damage_explosion, 100, _fade_yellow, NONE, _snd_human_scream, _monster_is_dying_hard},
    {_damage_crushing, NONE, _fade_red, NONE, _snd_human_wail, _monster_is_dying_hard},
    {_damage_projectile, NONE, _fade_red, NONE, _snd_human_scream, NONE},
    {_damage_shotgun_projectile, NONE, _fade_red, NONE, _snd_human_scream, NONE},
    {_damage_electrical_staff, NONE, _fade_cyan, NONE, _snd_human_scream, NONE},
    {_damage_hulk_slap, NONE, _fade_cyan, NONE, _snd_human_scream, NONE},
    {_damage_absorbed, 100, _fade_white, _snd_absorbed, NONE, NONE},
    {_damage_teleporter, 100, _fade_white, _snd_absorbed, NONE, NONE},
    {_damage_flame, NONE, _fade_orange, NONE, _snd_human_wail, _monster_is_dying_flaming},
    {_damage_hound_claws, NONE, _fade_red, NONE, _snd_human_scream, NONE},
    {_damage_compiler_bolt, NONE, _fade_static, NONE, _snd_human_scream, NONE},
    {_damage_alien_projectile, NONE, _fade_dodge_purple, NONE, _snd_human_wail, _monster_is_dying_flaming},
    {_damage_hunter_bolt, NONE, _fade_burn_green, NONE, _snd_human_scream, NONE},
    {_damage_fusion_bolt, 60, _fade_negative, NONE, _snd_human_scream, NONE},
    {_damage_fist, 40, _fade_red, NONE, _snd_human_scream, NONE},
    {_damage_yeti_claws, NONE, _fade_burn_cyan, NONE, _snd_human_scream, NONE},
    {_damage_yeti_projectile, NONE, _fade_dodge_yellow, NONE, _snd_human_scream, NONE},
    {_damage_defender, NONE, _fade_purple, NONE, _snd_human_scream, NONE},
    {_damage_lava, NONE, _fade_long_orange, NONE, _snd_human_wail, _monster_is_dying_flaming},
    {_damage_goo, NONE, _fade_long_green, NONE, _snd_human_wail, _monster_is_dying_flaming},
    {_damage_suffocation, NONE, NONE, NONE, _snd_suffocation, _monster_is_dying_soft},
    {_damage_energy_drain, NONE, NONE, NONE, NONE, NONE},
    {_damage_oxygen_drain, NONE, NONE, NONE, NONE, NONE},
    {_damage_hummer_bolt, NONE, _fade_flicker_negative, NONE, _snd_human_scream, NONE},
};
```

### Fade Colors by Damage Type

| Damage Type | Fade Color | Death Sound |
|-------------|------------|-------------|
| Explosion | Yellow | Scream |
| Projectile | Red | Scream |
| Flame | Orange | Wail |
| Fusion | Negative | Scream |
| Compiler | Static | Scream |
| Lava | Long Orange | Wail (dying flaming) |
| Goo | Long Green | Wail (dying flaming) |
| Absorbed | White | (absorbed sound) |

---

## 16.10 Player Damage Flow (`player.c:486-599`)

```c
void damage_player(
    short monster_index,
    short aggressor_index,
    short aggressor_type,
    struct damage_definition *damage)
{
    short player_index= monster_index_to_player_index(monster_index);
    struct player_data *player= get_player_data(player_index);
    short damage_amount= calculate_damage(damage);
    short damage_type= damage->type;
    struct damage_response_definition *definition;

    /* Invincibility check - absorb all except fusion */
    if (player->invincibility_duration && damage->type!=_damage_fusion_bolt)
    {
        damage_type= _damage_absorbed;
    }

    /* Find damage response definition */
    for (i=0; definition->type!=damage_type; ++i, ++definition);

    if (damage_type!=_damage_absorbed)
    {
        switch (damage->type)
        {
            case _damage_oxygen_drain:
                /* Drain oxygen directly */
                if ((player->suit_oxygen-= damage_amount)<0)
                    player->suit_oxygen= 0;
                break;

            default:
                /* Damage shields (suit_energy) */
                if ((player->suit_energy-= damage_amount)<0)
                {
                    if (damage->type!=_damage_energy_drain)
                    {
                        if (!PLAYER_IS_DEAD(player))
                        {
                            /* Determine death action */
                            short action= definition->death_action;
                            if (action==NONE)
                            {
                                action= (damage_amount>PLAYER_MAXIMUM_SUIT_ENERGY/2) ?
                                    _monster_is_dying_hard : _monster_is_dying_soft;
                            }

                            play_object_sound(player->object_index, definition->death_sound);
                            kill_player(player_index, aggressor_player_index, action);
                        }
                    }
                    player->suit_energy= 0;
                }
                break;
        }
    }

    /* Visual and audio feedback */
    if (!PLAYER_IS_DEAD(player))
        play_object_sound(player->object_index, definition->sound);

    if (player_index==current_player_index)
    {
        /* Screen fade */
        if (definition->fade!=NONE)
        {
            start_fade((definition->damage_threshhold!=NONE &&
                damage_amount>definition->damage_threshhold) ?
                (definition->fade+1) : definition->fade);
        }
        if (damage_amount)
            mark_shield_display_as_dirty();
    }
}
```

### Player Damage Flowchart

```
Damage to Player:
    │
    ├─► Check Invincibility
    │     └─► If active (except fusion): damage_type = _damage_absorbed
    │
    ├─► Find damage_response_definition for damage_type
    │
    ├─► If NOT absorbed:
    │     │
    │     ├─► Oxygen drain? Reduce suit_oxygen
    │     │
    │     └─► Other damage? Reduce suit_energy (shields)
    │           │
    │           └─► If suit_energy < 0 and not energy_drain:
    │                 └─► kill_player()
    │
    └─► Trigger feedback:
          ├─► Play pain/death sound
          ├─► Start screen fade (red, yellow, etc.)
          └─► Mark HUD for redraw
```

---

## 16.11 Summary

Marathon's damage system provides:

- **24 damage types** covering all attack sources
- **Immunity/weakness system** with bitfield checks
- **Difficulty scaling** via `_alien_damage` flag
- **Visual feedback** through colored screen fades
- **Audio feedback** with pain and death sounds

### Key Constants

| Constant | Value | Source |
|----------|-------|--------|
| `_damage_explosion` | 0 | `map.h:41` |
| `_damage_shotgun_projectile` | 23 | `map.h:64` |
| `_alien_damage` | 0x1 | `map.h:69` |
| Wuss level reduction | 50% | `marathon2.c:390` |
| Easy level reduction | 25% | `marathon2.c:391` |

### Key Source Files

| File | Purpose |
|------|---------|
| `map.h` | `struct damage_definition`, damage type enums |
| `marathon2.c` | `calculate_damage()` function |
| `monsters.c` | `damage_monster()` function |
| `player.c` | `damage_player()`, damage response definitions |
| `monster_definitions.h` | Monster vitality, immunities, weaknesses |

---

## 16.12 See Also

- [Chapter 8: Entities](08_entities.md) — Monster and projectile entities
- [Chapter 14: Items](14_items.md) — Powerups affecting damage
- [Chapter 22: Fades](22_fades.md) — Screen fade effects

---

*Next: [Chapter 17: Multiplayer Game Types](17_multiplayer.md) - Deathmatch, CTF, and scoring*
