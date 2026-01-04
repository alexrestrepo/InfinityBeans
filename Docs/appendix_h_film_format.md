# Appendix H: Film/Replay File Format

> **Source files**: `vbl.c`, `vbl.h`, `vbl_definitions.h`, `player.h`
> **Related chapters**: [Chapter 7: Game Loop](07_game_loop.md), [Chapter 9: Networking](09_networking.md)

Marathon can record and replay gameplay through **film files** (`.filA` or `.fil∞`). This appendix documents the complete file format for creating or parsing film recordings.

---

## Overview

Film files store a **stream of action flags** — the same 32-bit input values used in networked games. Since Marathon's simulation is fully deterministic, replaying the same inputs on the same map produces identical gameplay.

```
Film File Structure:
┌─────────────────────────────────┐
│ Recording Header (variable)     │
│   Game setup information        │
├─────────────────────────────────┤
│ Run-Length Encoded Action Flags │
│   [count][flags] pairs          │
│   Interleaved per player        │
├─────────────────────────────────┤
│ END_OF_RECORDING marker         │
└─────────────────────────────────┘
```

**File type** (`tags.h:19`):
```c
#define FILM_FILE_TYPE 'fil∞'
```

---

## Recording Header

The header contains all information needed to restore game state (`vbl_definitions.h:17-26`):

```c
struct recording_header {
    long length;                    // total file size in bytes
    short num_players;              // 1-8 players
    short level_number;             // map index
    unsigned long map_checksum;     // verify correct map
    short version;                  // recording format version
    struct player_start_data starts[MAXIMUM_NUMBER_OF_PLAYERS];  // 8 entries
    struct game_data game_information;
};
```

### Player Start Data

Each player's initial configuration (`map.h:149-155`):

```c
struct player_start_data {
    short team;                     // team color enum (0-7)
    short identifier;               // unique player ID
    short color;                    // player color enum
    char name[MAXIMUM_PLAYER_START_NAME_LENGTH+1];  // 32+1 bytes
};
```

**Team colors** (`player.h:38-49`):
```c
enum {
    _violet_team,      // 0
    _red_team,         // 1
    _tan_team,         // 2
    _light_blue_team,  // 3
    _yellow_team,      // 4
    _brown_team,       // 5
    _blue_team,        // 6
    _green_team,       // 7
    NUMBER_OF_TEAM_COLORS
};
```

### Game Data

Game rules and settings (`map.h:749-761`):

```c
struct game_data {
    long game_time_remaining;   // ticks until game ends (LONG_MAX for campaign)
    short game_type;            // see game type enum
    short game_options;         // bitfield of options
    short kill_limit;           // kills to win (net games)
    short initial_random_seed;  // RNG seed for determinism
    short difficulty_level;     // 0-4 (kindergarten to total carnage)
    short parameters[2];        // reserved
};
```

---

## Action Flags Format

Each tick, every player generates a 32-bit action flags value (`player.h:68-106`):

```
Action Flags (32 bits):
┌─────────────────────────────────────────────────────────────────┐
│ Bit 31                                                    Bit 0 │
├─────────┬─────────┬─────────────────────┬───────────────────────┤
│ Actions │ Movement│ Look/Pitch          │ Turn/Yaw              │
│ (8 bits)│ (7 bits)│ (6 bits)            │ (7 bits)              │
└─────────┴─────────┴─────────────────────┴───────────────────────┘
```

### Bit Assignments

| Bit | Name | Description |
|-----|------|-------------|
| 0 | `_absolute_yaw_mode` | Using absolute yaw (mouse) |
| 1 | `_turning_left` | Keyboard turn left |
| 2 | `_turning_right` | Keyboard turn right |
| 3 | `_sidestep_dont_turn` | Strafe modifier active |
| 4 | `_looking_left` | Look left (head turn) |
| 5 | `_looking_right` | Look right (head turn) |
| 6-7 | `_absolute_yaw_bit0-1` | Part of 7-bit absolute yaw |
| 8 | `_absolute_pitch_mode` | Using absolute pitch |
| 9 | `_looking_up` | Keyboard look up |
| 10 | `_looking_down` | Keyboard look down |
| 11 | `_looking_center` | Auto-center view |
| 12-13 | `_absolute_pitch_bit0-1` | Part of 5-bit absolute pitch |
| 14 | `_absolute_position_mode` | Using absolute position |
| 15 | `_moving_forward` | Move forward |
| 16 | `_moving_backward` | Move backward |
| 17 | `_run_dont_walk` | Run modifier |
| 18 | `_look_dont_turn` | Mouse look modifier |
| 19-21 | `_absolute_position_bit0-2` | Part of 7-bit position |
| 22 | `_sidestepping_left` | Strafe left |
| 23 | `_sidestepping_right` | Strafe right |
| 24 | `_left_trigger_state` | Primary fire |
| 25 | `_right_trigger_state` | Secondary fire |
| 26 | `_action_trigger_state` | Action key (use/activate) |
| 27 | `_cycle_weapons_forward` | Next weapon |
| 28 | `_cycle_weapons_backward` | Previous weapon |
| 29 | `_toggle_map` | Toggle automap |
| 30 | `_microphone_button` | Voice chat (network) |
| 31 | `_swim` | Swimming modifier |

### Absolute Value Extraction

For mouse input, absolute values are packed into the flags (`player.h:53-66`):

```c
// yaw: 7 bits (0-127), centered at 64
#define ABSOLUTE_YAW_BITS 7
#define GET_ABSOLUTE_YAW(i) (((i)>>(_absolute_yaw_mode_bit+1))&(MAXIMUM_ABSOLUTE_YAW-1))

// pitch: 5 bits (0-31), centered at 16
#define ABSOLUTE_PITCH_BITS 5
#define GET_ABSOLUTE_PITCH(i) (((i)>>(_absolute_pitch_mode_bit+1))&(MAXIMUM_ABSOLUTE_PITCH-1))

// position: 7 bits (for analog movement, rarely used)
#define ABSOLUTE_POSITION_BITS 7
#define GET_ABSOLUTE_POSITION(i) (((i)>>(_absolute_position_mode_bit+1))&(MAXIMUM_ABSOLUTE_POSITION-1))
```

---

## Run-Length Encoding

Action flags are stored using run-length encoding to compress repeated inputs (`vbl.c:416-484`):

```c
// Storage format: [count:short][flags:long] pairs
struct rle_entry {
    short run_count;    // number of consecutive ticks with same flags
    long action_flags;  // the action flags value
};
```

**Encoding algorithm** (`vbl.c:440-476`):
```c
for (i = 0; i < max_flags; i++) {
    flag = *(queue->buffer + queue->read_index);
    INCREMENT_QUEUE_COUNTER(queue->read_index);

    if (i && flag != last_flag) {
        // write previous run
        *(short*)location = run_count;
        ((short*)location)++;
        *location++ = last_flag;
        run_count = 1;
    } else {
        run_count++;
    }
    last_flag = flag;
}
// write final run
*(short*)location = run_count;
*location++ = last_flag;
```

**End marker** (`vbl.c:82-83`):
```c
#define RECORD_CHUNK_SIZE           (MAXIMUM_QUEUE_SIZE/2)  // 256
#define END_OF_RECORDING_INDICATOR  (RECORD_CHUNK_SIZE+1)   // 257
```

When `run_count == 257`, the recording has ended.

---

## File Layout

For a game with N players, action flags are interleaved in chunks:

```
┌──────────────────────────────────────────┐
│ Header                                   │
├──────────────────────────────────────────┤
│ Chunk 0:                                 │
│   Player 0: [count][flags]...            │  256 ticks worth
│   Player 1: [count][flags]...            │  256 ticks worth
│   ...                                    │
│   Player N-1: [count][flags]...          │  256 ticks worth
├──────────────────────────────────────────┤
│ Chunk 1:                                 │
│   Player 0: [count][flags]...            │
│   ...                                    │
├──────────────────────────────────────────┤
│ ... more chunks ...                      │
├──────────────────────────────────────────┤
│ Final Chunk:                             │
│   Player 0: [count][flags]...[257][0]    │  END marker
│   ...                                    │
└──────────────────────────────────────────┘
```

---

## Recording Process

**Starting a recording** (`vbl.c:647-676`):
```c
void start_recording(void) {
    replay.valid = TRUE;
    error = create_file(&recording_file, FILM_FILE_TYPE);
    replay.recording_file_refnum = open_file_for_writing(&recording_file);
    replay.game_is_being_recorded = TRUE;

    // write header
    write_file(replay.recording_file_refnum, sizeof(struct recording_header), &replay.header);
}
```

**During gameplay** (`vbl.c:376-387`):
```c
void process_action_flags(short player_identifier, long *action_flags, short count) {
    if (replay.game_is_being_recorded) {
        record_action_flags(player_identifier, action_flags, count);
    }
    queue_action_flags(player_identifier, action_flags, count);
}
```

**Stopping a recording** (`vbl.c:678-711`):
```c
void stop_recording(void) {
    // flush remaining queues
    for (player_index = 0; player_index < dynamic_world->player_count; player_index++) {
        save_recording_queue_chunk(player_index);
    }

    // rewrite header with final length
    set_fpos(replay.recording_file_refnum, 0);
    write_file(replay.recording_file_refnum, sizeof(struct recording_header), &replay.header);

    close_file(replay.recording_file_refnum);
    replay.game_is_being_recorded = FALSE;
}
```

---

## Playback Process

**Setup** (`vbl.c:602-644`):
```c
boolean setup_for_replay_from_file(FileDesc *file, unsigned long map_checksum) {
    replay.recording_file_refnum = open_file_for_reading(file);
    replay.game_is_being_replayed = TRUE;

    // read header
    read_file(replay.recording_file_refnum, sizeof(struct recording_header), &replay.header);

    // verify map matches
    if (!use_map_file(replay.header.map_checksum)) {
        alert_user(infoError, strERRORS, cantFindReplayMap, 0);
        return FALSE;
    }

    // allocate read cache
    replay.fsread_buffer = malloc(DISK_CACHE_SIZE);
    replay.replay_speed = 1;
    return TRUE;
}
```

**Per-tick playback** (`vbl.c:493-532`):
```c
static boolean pull_flags_from_recording(short count) {
    // verify all players have flags available
    for (player_index = 0; player_index < dynamic_world->player_count; player_index++) {
        if (get_recording_queue_size(player_index) == 0)
            return FALSE;
    }

    // dequeue flags for each player
    for (player_index = 0; player_index < dynamic_world->player_count; player_index++) {
        queue = get_player_recording_queue(player_index);
        for (index = 0; index < count; index++) {
            queue_action_flags(player_index, queue->buffer + queue->read_index, 1);
            INCREMENT_QUEUE_COUNTER(queue->read_index);
        }
    }
    return TRUE;
}
```

---

## Replay Speed Control

Films support variable-speed playback (`vbl.c:234-248`, `vbl.h:87-88`):

```c
#define MAXIMUM_REPLAY_SPEED  5   // 5x fast forward
#define MINIMUM_REPLAY_SPEED -5   // pause (negative = slow motion)

void increment_replay_speed(void) {
    if (replay.replay_speed < MAXIMUM_REPLAY_SPEED)
        replay.replay_speed++;
}

void decrement_replay_speed(void) {
    if (replay.replay_speed > MINIMUM_REPLAY_SPEED)
        replay.replay_speed--;
}
```

Speed interpretation (`vbl.c:335-357`):
- `speed > 0`: Process `speed` ticks per frame (fast forward)
- `speed == 0`: Process 1 tick every 2 frames (half speed)
- `speed < 0`: Larger negative = slower playback
- `speed == MINIMUM_REPLAY_SPEED`: Paused

---

## Determinism Requirements

For replays to work correctly, the game must be **fully deterministic**:

1. **Fixed-point math only** — No floating-point (see [Appendix D](appendix_d_fixedpoint.md))
2. **Synchronized RNG** — Same `initial_random_seed` produces same sequence
3. **Same physics** — Map checksum verifies identical physics
4. **Same tick rate** — Always 30 ticks/second
5. **Same code paths** — Platform-independent behavior

Any deviation causes **desync** — the replay diverges from the original.

---

## Bandwidth Analysis

For network transmission, action flags are extremely compact:

```
Per player per tick: 4 bytes (32-bit flags)
4 players × 4 bytes × 30 ticks/sec = 480 bytes/sec

Compare to modern games: 10-100 KB/sec per player
Marathon: ~97% more efficient!
```

This efficiency is why Marathon supported 8-player games over 1990s modems.

---

## Parsing Example (Pseudocode)

```python
def parse_film(file):
    # read header
    header = read_struct(file, recording_header)
    print(f"Players: {header.num_players}")
    print(f"Level: {header.level_number}")
    print(f"Map checksum: {header.map_checksum:08X}")

    # decode action flags
    tick = 0
    while True:
        for player in range(header.num_players):
            count = read_short(file)
            if count == END_OF_RECORDING_INDICATOR:
                return  # done
            flags = read_long(file)

            for _ in range(count):
                print(f"Tick {tick}, Player {player}: {flags:08X}")
                decode_flags(flags)
                tick += 1
```

---

## See Also

- [Chapter 7: Game Loop](07_game_loop.md) - How action flags drive simulation
- [Chapter 9: Networking](09_networking.md) - Network action queue system
- [Appendix G: Physics File Format](appendix_g_physics_file.md) - Physics modifications
- [Appendix J: Modding Cookbook](appendix_j_cookbook.md) - Practical examples
