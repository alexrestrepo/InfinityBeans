# Chapter 17: Multiplayer Game Types

## Deathmatch, Cooperative, and Team Games

> **Source files**: `network_games.c`, `network_games.h`, `map.h`
> **Related chapters**: [Chapter 9: Network](09_network.md), [Chapter 16: Damage](16_damage.md)

> **For Porting:** The game type logic in `network_games.c` is fully portable—pure C with no platform dependencies. Network transport in `network_ddp.c` and `network_adsp.c` needs replacement with modern sockets (TCP/UDP).

---

## 17.1 What Problem Are We Solving?

Marathon supports multiple multiplayer modes beyond simple deathmatch:

- **Cooperative play** - Story mode with friends
- **Every Man for Himself** - Classic deathmatch
- **Team games** - Capture the Flag, King of the Hill
- **Objective modes** - Tag, Rugby, Defense
- **Ball carrier games** - Kill Man With Ball

**The constraints:**
- All game logic must be deterministic for network sync
- Compass must help locate objectives
- Different scoring for different modes
- Special polygon types for objectives

---

## 17.2 Game Type Enumeration (`map.h:727-738`)

```c
/* Game types! */
enum {
    _game_of_kill_monsters,       // 0 - Single player & deathmatch
    _game_of_cooperative_play,    // 1 - Multiple players working together
    _game_of_capture_the_flag,    // 2 - Team game with flags
    _game_of_king_of_the_hill,    // 3 - Control the hill polygon
    _game_of_kill_man_with_ball,  // 4 - Ball carrier hunted
    _game_of_defense,             // 5 - Attack/Defend
    _game_of_rugby,               // 6 - Score goals with ball
    _game_of_tag,                 // 7 - Avoid being "it"
    NUMBER_OF_GAME_TYPES
};

#define GET_GAME_TYPE() (dynamic_world->game_information.game_type)
#define GET_GAME_OPTIONS() (dynamic_world->game_information.game_options)
#define GET_GAME_PARAMETER(x) (dynamic_world->game_information.parameters[(x)])
```

---

## 17.3 Game Options Flags (`map.h:700-717`)

```c
enum /* game options.. */
{
    _multiplayer_game= 0x0001,               // Multi or single player
    _ammo_replenishes= 0x0002,               // Ammo respawns
    _weapons_replenish= 0x0004,              // Weapons respawn
    _specials_replenish= 0x0008,             // Powerups respawn
    _monsters_replenish= 0x0010,             // Monsters respawn
    _motion_sensor_does_not_work= 0x0020,    // Disable motion sensor
    _overhead_map_is_omniscient= 0x0040,     // Only show teammates on map
    _burn_items_on_death= 0x0080,            // Lose items on death
    _live_network_stats= 0x0100,             // Show live scores
    _game_has_kill_limit= 0x0200,            // End game at kill limit
    _force_unique_teams= 0x0400,             // Every player unique team
    _dying_is_penalized= 0x0800,             // Time penalty for death
    _suicide_is_penalized= 0x1000,           // Time penalty for suicide
    _overhead_map_shows_items= 0x2000,       // Items visible on map
    _overhead_map_shows_monsters= 0x4000,    // Monsters visible on map
    _overhead_map_shows_projectiles= 0x8000  // Projectiles visible on map
};
```

---

## 17.4 Game Data Structure (`map.h:749-761`)

```c
struct game_data
{
    /* Remaining game time in ticks (LONG_MAX for single-player) */
    long game_time_remaining;

    short game_type;          /* One of game type enum values */
    short game_options;       /* Bitfield of game options */
    short kill_limit;         /* Kill limit if enabled */
    short initial_random_seed;/* Seed for deterministic RNG */
    short difficulty_level;   /* Current difficulty */
    short parameters[2];      /* Game-type-specific parameters */
};
```

---

## 17.5 Net Game Parameters (`network_games.c:22-48`)

Game-type-specific parameter indices:

```c
enum { /* for king of the hill */
    _king_of_hill_time= 0
};

enum { /* for kill the man with the ball */
    _ball_carrier_time= 0
};

enum { /* for offense/defense */
    _offender_time_in_base= 0,       // for player->netgame_parameters
    _defending_team= 0,              // for game_information->parameters
    _maximum_offender_time_in_base= 1 // for game_information->parameters
};

enum { /* for rugby */
    _points_scored= 0
};

enum { /* for tag */
    _time_spent_it= 0
};

enum { /* for capture the flag */
    _flag_pulls= 0,    // for player->netgame_parameters
    _winning_team= 0   // for game_information->parameters[]
};
```

---

## 17.6 Player Net Ranking (`network_games.c:56-131`)

Scoring varies by game type:

```c
long get_player_net_ranking(
    short player_index,
    short *kills,
    short *deaths,
    boolean game_is_over)
{
    struct player_data *player= get_player_data(player_index);
    long ranking;

    /* Calculate kills and deaths */
    *deaths = player->monster_damage_taken.kills;
    for (index= 0; index<dynamic_world->player_count; ++index)
    {
        if (index!=player_index)
        {
            (*kills)+= get_player_data(index)->damage_taken[player_index].kills;
        }
        (*deaths)+= player->damage_taken[index].kills;
    }

    switch(GET_GAME_TYPE())
    {
        case _game_of_kill_monsters:
            ranking= (*kills)-(*deaths);
            break;

        case _game_of_cooperative_play:
            /* Percentage of total monster damage */
            ranking= total_monster_damage ?
                (100*monster_damage)/total_monster_damage : 0;
            break;

        case _game_of_capture_the_flag:
            ranking= player->netgame_parameters[_flag_pulls];
            break;

        case _game_of_king_of_the_hill:
            ranking= player->netgame_parameters[_king_of_hill_time];
            break;

        case _game_of_kill_man_with_ball:
            ranking= player->netgame_parameters[_ball_carrier_time];
            break;

        case _game_of_tag:
            ranking= -player->netgame_parameters[_time_spent_it]; /* Negative! */
            break;

        case _game_of_defense:
            ranking= (*kills)-(*deaths);
            if(game_is_over && GET_GAME_PARAMETER(_winning_team)==player->team)
            {
                ranking += 50;  /* Bonus for winning team */
            }
            break;

        case _game_of_rugby:
            ranking= (*kills)-(*deaths);
            break;
    }

    return ranking;
}
```

---

## 17.7 Special Polygon Types (`map.h:537-550`)

Objective-based game types use special polygon types:

```c
enum /* polygon types */
{
    /* ... other types ... */
    _polygon_is_hill,     /* for king-of-the-hill */
    _polygon_is_base,     /* for capture the flag, rugby (team in .permutation) */
    _polygon_is_platform, /* platform index in .permutation */
    /* ... */
    _polygon_is_goal,
    /* ... */
};
```

---

## 17.8 Network Compass System (`network_games.h:35-47`)

```c
enum
{
    _network_compass_all_off= 0,

    _network_compass_nw= 0x0001,
    _network_compass_ne= 0x0002,
    _network_compass_sw= 0x0004,
    _network_compass_se= 0x0008,

    _network_compass_all_on= 0x000f
};

short get_network_compass_state(short player_index);
```

### Compass State Function (`network_games.c:176-237`)

```c
#define NETWORK_COMPASS_SLOP SIXTEENTH_CIRCLE

short get_network_compass_state(short player_index)
{
    short state= _network_compass_all_off;
    world_point2d *beacon= (world_point2d *) NULL;

    switch (GET_GAME_TYPE())
    {
        case _game_of_king_of_the_hill:
            /* Am I on the hill? */
            if (get_polygon_data(player->supporting_polygon_index)->type==_polygon_is_hill)
            {
                state= _network_compass_all_on;  /* All lights on = you're there */
            }
            else
            {
                beacon= &dynamic_world->game_beacon;  /* Point to hill center */
            }
            break;

        case _game_of_tag:
            if (dynamic_world->game_player_index==player_index)
            {
                state= _network_compass_all_on;  /* You're "it" */
            }
            else if (dynamic_world->game_player_index!=NONE)
            {
                beacon= &get_player_data(dynamic_world->game_player_index)->location;
            }
            break;

        case _game_of_kill_man_with_ball:
            if (player_has_ball(player_index, SINGLE_BALL_COLOR))
            {
                state= _network_compass_all_on;  /* You have the ball */
            }
            else if (dynamic_world->game_player_index!=NONE)
            {
                beacon= &get_player_data(dynamic_world->game_player_index)->location;
            }
            break;
    }

    /* Calculate compass quadrants from beacon angle */
    if (beacon)
    {
        angle theta= NORMALIZE_ANGLE(facing - arctangent(origin->x-beacon->x, origin->y-beacon->y));

        if (theta>FULL_CIRCLE-NETWORK_COMPASS_SLOP || theta<QUARTER_CIRCLE+NETWORK_COMPASS_SLOP)
            state|= _network_compass_se;
        if (theta>QUARTER_CIRCLE-NETWORK_COMPASS_SLOP && theta<HALF_CIRCLE+NETWORK_COMPASS_SLOP)
            state|= _network_compass_ne;
        if (theta>HALF_CIRCLE-NETWORK_COMPASS_SLOP && theta<HALF_CIRCLE+QUARTER_CIRCLE+NETWORK_COMPASS_SLOP)
            state|= _network_compass_nw;
        if (theta>HALF_CIRCLE+QUARTER_CIRCLE-NETWORK_COMPASS_SLOP || theta<NETWORK_COMPASS_SLOP)
            state|= _network_compass_sw;
    }

    return state;
}
```

### Compass Visualization

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

All quadrants lit = you ARE the objective (hill/it/ball carrier)
```

---

## 17.9 Game Type Details

### Every Man for Himself (`_game_of_kill_monsters`)

```
Objective: Highest kill count wins
Scoring:   kills - deaths
Compass:   Not used
```

### Cooperative Play (`_game_of_cooperative_play`)

```
Objective: Complete level together
Scoring:   Percentage of total monster damage dealt
           ranking = (100 * player_monster_damage) / total_monster_damage
Compass:   Not used
```

### Capture the Flag (`_game_of_capture_the_flag`)

```
Objective: Capture enemy flags
Scoring:   Number of flag pulls (netgame_parameters[_flag_pulls])

Mechanics:
  - Each team has base polygon (_polygon_is_base)
  - Ball items represent flags
  - Carry enemy flag to your base
  - polygon.permutation = team index
```

### King of the Hill (`_game_of_king_of_the_hill`)

```
Objective: Occupy hill polygon longest
Scoring:   Ticks spent in hill (netgame_parameters[_king_of_hill_time])

Initialization (network_games.c:138-158):
  - Calculate center of all _polygon_is_hill polygons
  - Store in dynamic_world->game_beacon

Compass:   Points to hill center; all-on when ON hill
```

### Kill Man With Ball (`_game_of_kill_man_with_ball`)

```
Objective: Kill the ball carrier
Scoring:   Ticks holding ball (netgame_parameters[_ball_carrier_time])

Mechanics:
  - Single ball spawns (SINGLE_BALL_COLOR = 1)
  - Carrier earns points over time
  - dynamic_world->game_player_index = ball carrier

Compass:   Points to ball carrier; all-on when YOU have ball
```

### Tag (`_game_of_tag`)

```
Objective: Avoid being "it"
Scoring:   NEGATIVE time spent "it" (-netgame_parameters[_time_spent_it])
           Lowest time wins (highest negative score)

Mechanics:
  - One player is "it" (dynamic_world->game_player_index)
  - Kill someone to transfer "it" status
  - Plays _snd_you_are_it when becoming "it"

Compass:   Points to "it"; all-on when YOU are "it"
```

### Defense (`_game_of_defense`)

```
Objective: Attack or defend based on team
Scoring:   (kills - deaths) + 50 (if on winning team)

Mechanics:
  - parameters[_defending_team] = defending team index
  - Attackers try to spend time in base
  - Defenders try to prevent this
```

### Rugby (`_game_of_rugby`)

```
Objective: Score goals with ball
Scoring:   kills - deaths + goals (netgame_parameters[_points_scored])

Mechanics:
  - Carry ball to goal polygon
  - Similar to CTF but with goals instead of flags
```

---

## 17.10 Tag "It" Transfer (`network_games.c:240-281`)

```c
boolean player_killed_player(
    short dead_player_index,
    short aggressor_player_index)
{
    boolean attribute_kill= TRUE;

    if (dynamic_world->player_count>1)
    {
        switch (GET_GAME_TYPE())
        {
            case _game_of_tag:
                /* Transfer "it" if killed by "it", suicide, or no "it" yet */
                if (aggressor_player_index==dynamic_world->game_player_index ||
                    dead_player_index==aggressor_player_index ||
                    dynamic_world->game_player_index==NONE)
                {
                    if (dynamic_world->game_player_index!=dead_player_index)
                    {
                        /* Change of "it" */
                        struct player_data *player= get_player_data(dead_player_index);
                        play_object_sound(player->object_index, _snd_you_are_it);
                        dynamic_world->game_player_index= dead_player_index;
                    }
                }
                break;
        }
    }

    return attribute_kill;
}
```

---

## 17.11 Summary

Marathon's multiplayer system provides:

- **8 game types** covering varied gameplay styles
- **Team support** with team-specific polygons
- **Compass system** for objective tracking
- **Flexible scoring** customized per mode
- **Dynamic objectives** (moving ball carriers, "it" players)

### Key Constants

| Constant | Value | Source |
|----------|-------|--------|
| `_game_of_kill_monsters` | 0 | `map.h:729` |
| `NUMBER_OF_GAME_TYPES` | 8 | `map.h:737` |
| `_network_compass_all_on` | 0x000f | `network_games.h:44` |
| `NETWORK_COMPASS_SLOP` | SIXTEENTH_CIRCLE | `network_games.c:174` |
| `SINGLE_BALL_COLOR` | 1 | `network_games.c:18` |

### Key Source Files

| File | Purpose |
|------|---------|
| `network_games.c` | Game type logic, scoring, compass |
| `network_games.h` | Compass enums, function prototypes |
| `map.h` | Game type enums, game_data structure |
| `network.c` | Core networking (transport layer) |
| `player.c` | netgame_parameters, score tracking |

---

## 17.12 See Also

- [Chapter 9: Network](09_network.md) — Network architecture and synchronization
- [Chapter 16: Damage](16_damage.md) — Kill attribution for scoring
- [Chapter 14: Items](14_items.md) — Ball items for game modes

---

*Next: [Chapter 18: Random Number Generation](18_random.md) - Deterministic RNG for networking*
