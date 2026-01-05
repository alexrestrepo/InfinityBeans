# Chapter 18: Random Number Generation

## Deterministic RNG for Networked Gameplay

> **Source files**: `world.c`, `world.h`
> **Related chapters**: [Chapter 9: Network](09_network.md), [Chapter 17: Multiplayer](17_multiplayer.md)

> **For Porting:** The RNG in `world.c` is fully portable—just a 16-bit LFSR with no platform dependencies. Copy the functions directly.

---

## 18.1 What Problem Are We Solving?

Marathon needs random numbers for:

- **Damage variation** - Weapons don't deal exact same damage every hit
- **AI decisions** - Monsters don't behave identically
- **Particle effects** - Visual variety in explosions, debris
- **Sound variations** - Pitch/volume randomization

But with networked multiplayer, all clients must see the **same** results for gameplay decisions.

**The constraint:**
- Gameplay-affecting random calls must be deterministic across all clients
- Visual-only random calls can differ between machines

---

## 18.2 The Dual RNG System

Marathon uses two separate random number generators:

```
┌─────────────────────────────────────────────────────────┐
│                RANDOM NUMBER SYSTEM                     │
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

## 18.3 Global State (`world.c:40-41`)

```c
static word random_seed= 0x1;
static word local_random_seed= 0x1;
```

Both seeds are initialized to 1 to ensure the LFSR never gets stuck at 0.

---

## 18.4 Default Seed (`world.h:36`)

```c
#define DEFAULT_RANDOM_SEED ((word)0xfded)
```

The default seed `0xfded` is used when no seed is provided (e.g., single-player games).

---

## 18.5 LFSR Implementation (`world.c:273-305`)

Both RNGs use a **Linear Feedback Shift Register** (LFSR):

### Synchronized Random (`world.c:273-288`)

```c
word random(void)
{
    word seed= random_seed;

    if (seed&1)
    {
        seed= (seed>>1)^0xb400;
    }
    else
    {
        seed>>= 1;
    }

    return (random_seed= seed);
}
```

### Local Random (`world.c:290-305`)

```c
word local_random(void)
{
    word seed= local_random_seed;

    if (seed&1)
    {
        seed= (seed>>1)^0xb400;
    }
    else
    {
        seed>>= 1;
    }

    return (local_random_seed= seed);
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

## 18.6 Seed Management (`world.c:259-271`)

### Set Seed (`world.c:259-265`)

```c
void set_random_seed(word seed)
{
    random_seed= seed ? seed : DEFAULT_RANDOM_SEED;

    return;
}
```

**Note**: If seed is 0, uses `DEFAULT_RANDOM_SEED` because LFSR with seed 0 would produce only 0s forever.

### Get Seed (`world.c:267-271`)

```c
word get_random_seed(void)
{
    return random_seed;
}
```

---

## 18.7 Function Prototypes (`world.h:138-142`)

```c
void set_random_seed(word seed);
word get_random_seed(void);
word random(void);

word local_random(void);
```

---

## 18.8 Network Synchronization Flow

```
Game Start:
    Host generates initial_random_seed ──► stored in game_data
                                        ──► sent to all clients

Level Load:
    set_random_seed(game_information.initial_random_seed);

During Game:
    All clients call random() in identical order
    (guaranteed by deterministic game loop)

    Same sequence on all machines
    ──► Deterministic simulation maintained
```

The seed is stored in `game_data.initial_random_seed` (`map.h:758`) and distributed as part of network game setup.

---

## 18.9 Usage Guidelines

| Function | Use Case | Network Safe? |
|----------|----------|---------------|
| `random()` | Gameplay mechanics | YES - must stay in sync |
| `local_random()` | Visual effects | NO - can differ per client |

### Correct Usage Examples

```c
/* CORRECT: Damage rolls use synchronized random */
damage = base + random() % random_damage;

/* CORRECT: Monster AI decisions use synchronized random */
if (random() < attack_chance) attack();

/* CORRECT: Particle effects use local random */
particle_x_offset = local_random() % spread;

/* CORRECT: Sound pitch variation uses local random */
pitch = base_pitch + (local_random() % variation);
```

### Incorrect Usage (Causes Desync!)

```c
/* WRONG: Using local_random for gameplay */
monster_attack = local_random() % 100;  // DON'T DO THIS

/* WRONG: Using random() for visuals (wastes sync) */
particle_color = random() % 256;  // Wasteful, use local_random()
```

---

## 18.10 Why LFSR?

| Property | Benefit |
|----------|---------|
| **Fast** | Single shift + conditional XOR per call |
| **Deterministic** | Same seed = same sequence always |
| **Full period** | 65535 values before repeat |
| **Small state** | Only 16 bits to synchronize |
| **No external deps** | No library calls, portable C |

### Performance

```c
/* The entire RNG is just: */
if (seed & 1)
    seed = (seed >> 1) ^ 0xb400;
else
    seed >>= 1;
```

This compiles to approximately 4-6 assembly instructions on most architectures.

---

## 18.11 Summary

Marathon's RNG system provides:

- **Dual generators** for network safety
- **LFSR algorithm** for speed and determinism
- **Seed synchronization** at game start
- **65535 period** (2^16 - 1) for adequate variety
- **Zero-protection** with DEFAULT_RANDOM_SEED fallback

### Key Constants

| Constant | Value | Source |
|----------|-------|--------|
| `DEFAULT_RANDOM_SEED` | 0xfded | `world.h:36` |
| LFSR polynomial | 0xb400 | `world.c:280,297` |
| Initial seed | 0x1 | `world.c:40-41` |
| Period | 65535 | (2^16 - 1) |

### Key Source Files

| File | Purpose |
|------|---------|
| `world.c` | RNG implementation (lines 40-41, 259-305) |
| `world.h` | RNG prototypes and DEFAULT_RANDOM_SEED |
| `map.h` | game_data.initial_random_seed storage |
| `network.c` | Seed distribution during game setup |

---

## 18.12 See Also

- [Chapter 9: Network](09_network.md) — How seed is distributed
- [Chapter 17: Multiplayer](17_multiplayer.md) — game_data structure
- [Chapter 16: Damage](16_damage.md) — Uses random() for damage variation

---

*Next: [Chapter 19: Shape Animation System](19_shapes.md) - Sprites, collections, and animation*
