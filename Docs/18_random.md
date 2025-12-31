# Chapter 18: Random Number Generation

## Deterministic RNG for Networked Gameplay

> **For Porting:** The RNG in `world.c` is fully portable—just a 16-bit LFSR with no platform dependencies.

---

## 18.1 What Problem Are We Solving?

Marathon needs random numbers for:

- **Damage variation** - Weapons don't deal exact same damage every hit
- **AI decisions** - Monsters don't behave identically
- **Particle effects** - Visual variety in explosions, debris

But with networked multiplayer, all clients must see the **same** results.

---

## 18.2 The Dual RNG System

Marathon uses two separate random number generators:

```
┌─────────────────────────────────────────────────────────┐
│                RANDOM NUMBER SYSTEM                      │
├─────────────────────────┬───────────────────────────────┤
│    Synchronized RNG     │       Local RNG               │
│    (random())           │       (local_random())        │
├─────────────────────────┼───────────────────────────────┤
│ • Same seed all clients │ • Independent per machine     │
│ • Damage calculations   │ • Particle positions          │
│ • AI behavior           │ • Visual effects              │
│ • Item spawns           │ • Sound variations            │
│ • MUST stay in sync     │ • Okay to differ              │
└─────────────────────────┴───────────────────────────────┘
```

---

## 18.3 LFSR Implementation

Both RNGs use a **Linear Feedback Shift Register** (LFSR):

```c
word random_seed;
word local_random_seed;

word random(void) {
    if (random_seed & 1) {
        random_seed = (random_seed >> 1) ^ 0xb400;
    } else {
        random_seed >>= 1;
    }
    return random_seed;
}
```

### Bit-Level Operation

```
16-bit LFSR with polynomial 0xb400:

Bit positions:  15 14 13 12 11 10  9  8  7  6  5  4  3  2  1  0
                ─────────────────────────────────────────────────
Initial:         0  0  0  0  0  0  0  0  0  0  0  0  0  0  0  1

If bit 0 = 1:
  1. Shift right:  0  0  0  0  0  0  0  0  0  0  0  0  0  0  0  0
  2. XOR 0xb400:   1  0  1  1  0  1  0  0  0  0  0  0  0  0  0  0
     Result:       1  0  1  1  0  1  0  0  0  0  0  0  0  0  0  0

If bit 0 = 0:
  1. Shift right only

0xb400 = 1011 0100 0000 0000
         ↑  ↑  ↑
        15 13 12 10 (feedback taps)

Period: 65535 (2^16 - 1, maximum for 16-bit LFSR)
```

---

## 18.4 Seed Synchronization

```c
void set_random_seed(word seed) {
    random_seed = seed ? seed : 1;  // Never allow 0 (LFSR would stick)
}

word get_random_seed(void) {
    return random_seed;
}
```

### Network Sync Flow

```
Game Start:
    Host generates seed ──────────► All clients set same seed

During Game:
    All clients call random() in identical order
    Same sequence on all machines
    Deterministic simulation maintained
```

---

## 18.5 Usage Guidelines

| Function | Use Case | Network Safe? |
|----------|----------|---------------|
| `random()` | Gameplay mechanics | YES |
| `local_random()` | Visual effects | NO |

### Examples

```c
// CORRECT: Damage rolls use synchronized random
damage = base + random() % random_damage;

// CORRECT: Particle effects use local random
particle_x_offset = local_random() % spread;

// WRONG: Using local_random for gameplay
// This would cause network desync!
monster_attack = local_random() % 100;  // DON'T DO THIS
```

---

## 18.6 Why LFSR?

| Property | Benefit |
|----------|---------|
| Fast | Single shift + XOR per call |
| Deterministic | Same seed = same sequence |
| Full period | 65535 values before repeat |
| Small state | Only 16 bits to sync |

---

## 18.7 Summary

Marathon's RNG system provides:

- **Dual generators** for network safety
- **LFSR algorithm** for speed and determinism
- **Seed synchronization** at game start
- **65535 period** for adequate variety

### Key Source Files

| File | Purpose |
|------|---------|
| `world.c` | RNG implementation |
| `network.c` | Seed distribution |

---

*Next: [Chapter 19: Shape Animation System](19_shapes.md) - Sprites, collections, and animation*
