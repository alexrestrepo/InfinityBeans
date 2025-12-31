# Chapter 16: Damage System

## Hit Detection, Damage Types, and Combat

> **For Porting:** The damage system is fully portable. Damage definitions are static data, and the calculation functions use only integer math.

---

## 16.1 What Problem Are We Solving?

Marathon needs a flexible damage system that handles:

- **Multiple damage types** with different effects
- **Monster immunities and weaknesses**
- **Difficulty scaling** for different skill levels
- **Visual feedback** through screen effects

---

## 16.2 Damage Definition Structure

```c
struct damage_definition {
    short type;    // Damage type (see types below)
    short flags;   // Modifier flags

    short base;    // Base damage amount
    short random;  // Random additional damage [0, random)
    fixed scale;   // Multiplier (FIXED_ONE = 1.0)
};
```

---

## 16.3 Damage Calculation

```c
short calculate_damage(struct damage_definition *damage) {
    // Step 1: Base + random
    short total = damage->base + (damage->random ? random() % damage->random : 0);

    // Step 2: Apply scale
    total = FIXED_INTEGERAL_PART(total * damage->scale);

    // Step 3: Difficulty modifier (alien damage only)
    if (damage->flags & _alien_damage) {
        switch (difficulty_level) {
            case _wuss_level: total -= total >> 1; break;  // -50%
            case _easy_level: total -= total >> 2; break;  // -25%
            // Normal and above: no reduction
        }
    }

    return total;
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
            │ Apply Scale │  scale = 1.5
            └─────────────┘
                  │
                  ▼
            125 × 1.5 = 187
                  │
                  ▼
            ┌─────────────┐
            │  Difficulty │  (if alien damage)
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

## 16.4 Monster Damage Application

```c
void damage_monster(short target_index, short aggressor_index,
                    struct damage_definition *damage) {
    short delta_vitality = calculate_damage(damage);

    // Immunity check
    if (definition->immunities & FLAG(damage->type)) {
        return;  // No damage
    }

    // Weakness check (2x damage)
    if (definition->weaknesses & FLAG(damage->type)) {
        delta_vitality <<= 1;
    }

    // Apply damage
    monster->vitality -= delta_vitality;

    // Check for death
    if (monster->vitality <= 0) {
        kill_monster(target_index, aggressor_index, damage);
    }
}
```

---

## 16.5 Damage Types

| Type ID | Name | Typical Source |
|---------|------|----------------|
| 0 | `_damage_explosion` | Rockets, grenades |
| 1 | `_damage_electrical_staff` | Staff weapon |
| 2 | `_damage_projectile` | Bullets |
| 3 | `_damage_absorbed` | Shield blocked |
| 4 | `_damage_flame` | Flamethrower |
| 5 | `_damage_hound_claws` | Hound melee |
| 6 | `_damage_alien_projectile` | Alien weapons |
| 7 | `_damage_hulk_slap` | Hulk melee |
| 8 | `_damage_compiler_bolt` | Compiler attack |
| 9 | `_damage_fusion_bolt` | Fusion pistol |
| 10 | `_damage_hunter_bolt` | Hunter shot |
| 11 | `_damage_fist` | Player punch |
| 12 | `_damage_teleporter` | Telefrag |
| 13 | `_damage_defender` | Defender attack |
| 14 | `_damage_yeti_claws` | Yeti melee |
| 15 | `_damage_yeti_projectile` | Yeti shot |
| 16 | `_damage_crushing` | Platform/door |
| 17 | `_damage_lava` | Lava contact |
| 18 | `_damage_suffocation` | No oxygen |
| 19 | `_damage_goo` | Sewage/goo |
| 20 | `_damage_energy_drain` | Shield drain |
| 21 | `_damage_oxygen_drain` | O2 drain |
| 22 | `_damage_hummer_bolt` | Hummer attack |
| 23 | `_damage_shotgun_projectile` | Shotgun pellets |

---

## 16.6 Player Damage Response

```c
struct damage_response_definition {
    short type;               // Damage type this responds to
    short damage_threshhold;  // Minimum damage for response
    word fade;                // Screen fade color
    short sound;              // Pain sound
    fixed death_sound_chance;
    short death_action;
};
```

### Player Shield System

```
Damage to Player:
    │
    ├─► Check Invincibility
    │     └─► If active: absorb all
    │
    ├─► Apply to Shields first
    │     └─► shields -= damage
    │
    ├─► If shields < 0:
    │     └─► Overflow goes to health
    │     └─► dead_player() if health <= 0
    │
    └─► Trigger damage response
          ├─► Screen fade (red flash)
          ├─► Pain sound
          └─► Knockback
```

---

## 16.7 Summary

Marathon's damage system provides:

- **24 damage types** for variety
- **Immunity/weakness system** for tactical combat
- **Difficulty scaling** for accessibility
- **Screen feedback** for player awareness

### Key Source Files

| File | Purpose |
|------|---------|
| `monsters.c` | Monster damage application |
| `player.c` | Player damage and death |
| `projectiles.c` | Damage on impact |
| `damage_definitions.h` | Damage type definitions |

---

*Next: [Chapter 17: Multiplayer Game Types](17_multiplayer.md) - Deathmatch, CTF, and scoring*
