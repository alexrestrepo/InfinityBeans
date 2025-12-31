# Chapter 17: Multiplayer Game Types

## Deathmatch, Cooperative, and Team Games

> **For Porting:** The game type logic in `network_games.c` is portable. Network transport in `network_ddp.c` and `network_adsp.c` needs replacement with modern sockets.

---

## 17.1 What Problem Are We Solving?

Marathon supports multiple multiplayer modes beyond simple deathmatch:

- **Cooperative play** - Story mode with friends
- **Every Man for Himself** - Classic deathmatch
- **Team games** - Capture the Flag, King of the Hill
- **Objective modes** - Tag, Rugby, Defense

---

## 17.2 Game Type Enumeration

```c
enum {
    _game_of_kill_monsters,      // Co-op: Kill aliens
    _game_of_cooperative_play,   // Co-op: Story mode
    _game_of_capture_the_flag,   // CTF
    _game_of_king_of_the_hill,   // KOTH
    _game_of_kill_man_with_ball, // Ball carrier hunted
    _game_of_tag,                // Avoid being "it"
    _game_of_defense,            // Attack/Defend
    _game_of_rugby               // Score goals
};
```

---

## 17.3 Game Type Details

### Every Man for Himself

```
Objective: Highest kill count wins
Scoring: kills - deaths
```

### Cooperative Play

```
Objective: Complete level together
Scoring: Percentage of total monster damage dealt
         ranking = (100 * player_monster_damage) / total_monster_damage
```

### Capture the Flag

```
Objective: Capture enemy flags
Scoring: Number of flag pulls

Mechanics:
  - Each team has base polygon (_polygon_is_base)
  - Ball items represent flags
  - Carry enemy flag to your base
```

### King of the Hill

```
Objective: Occupy hill polygon longest
Scoring: Ticks spent in hill

Hill Location:
  - Polygons marked _polygon_is_hill
  - Compass points to hill center
```

### Kill Man With Ball

```
Objective: Kill the ball carrier
Scoring: Ticks holding ball

Mechanics:
  - Single ball spawns
  - Carrier visible to all (compass)
  - Carrier earns points over time
  - Killing carrier drops ball
```

### Tag

```
Objective: Avoid being "it"
Scoring: NEGATIVE time spent "it"

Mechanics:
  - One player is "it"
  - Tag another to transfer
  - "It" visible on compass
  - Lowest time wins
```

### Defense

```
Objective: Attack or defend based on team
Scoring: kills - deaths + 50 (if winning team)
```

### Rugby

```
Objective: Score goals with ball
Scoring: Goals scored + (kills - deaths)
```

---

## 17.4 Network Compass System

The compass helps locate objectives:

```c
short get_network_compass_state(short player_index) {
    // Returns bitmask of compass quadrants
    // _network_compass_ne, _network_compass_nw
    // _network_compass_se, _network_compass_sw
}
```

### Visualization

```
Compass Display:
    ┌───────┐
    │ NW NE │
    │   ●   │   ● = objective direction
    │ SW SE │
    └───────┘

Example: Objective is northeast
    ┌───────┐
    │ ░░ ▓▓ │   NE quadrant highlighted
    │   ●   │
    │ ░░ ░░ │
    └───────┘
```

---

## 17.5 Summary

Marathon's multiplayer system provides:

- **8 game types** for varied gameplay
- **Team support** with compass tracking
- **Special polygons** for objectives (base, hill)
- **Flexible scoring** per mode

### Key Source Files

| File | Purpose |
|------|---------|
| `network_games.c` | Game type logic |
| `network.c` | Core networking |
| `player.c` | Score tracking |

---

*Next: [Chapter 18: Random Number Generation](18_random.md) - Deterministic RNG for networking*
