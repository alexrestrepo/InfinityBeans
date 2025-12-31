# Chapter 28: Computer Terminal System

## Story Content, Checkpoints, and Level Transitions

> **For Porting:** Terminal logic in `computer_interface.c` is portable. Replace Mac text rendering with your font system. The preprocessed terminal format is cross-platform.

---

## 28.1 What Problem Are We Solving?

Marathon tells its story through in-game computer terminals that:

- **Deliver narrative** - Plot, backstory, character dialog
- **Provide objectives** - Mission briefings with checkpoints
- **Enable teleportation** - Level-to-level and within-level travel
- **Display multimedia** - Images, sounds, movies

---

## 28.2 Terminal Scripting Language

Terminals are authored using a markup language:

### Group Commands

| Command | Purpose | Parameter |
|---------|---------|-----------|
| `#LOGON XXXX` | Login screen | Shape ID for graphic |
| `#LOGOFF` | Logout screen | None |
| `#UNFINISHED` | Show if mission incomplete | None |
| `#SUCCESS` | Show if mission complete | None |
| `#FAILURE` | Show if mission failed | None |
| `#INFORMATION` | General text info | None |
| `#CHECKPOINT XX` | Show map checkpoint | Goal ID |
| `#PICT XXXX` | Display image | PICT resource ID |
| `#SOUND XXXX` | Play sound effect | Sound ID |
| `#MOVIE XXXX` | Play QuickTime movie | Movie ID |
| `#TRACK XXXX` | Play music track | Track ID |
| `#INTERLEVEL TELEPORT XXX` | Go to another level | Level number |
| `#INTRALEVEL TELEPORT XXX` | Teleport within level | Polygon index |
| `#STATIC XX` | Display static effect | Duration ticks |
| `#TAG XX` | Activate tagged objects | Tag number |
| `#END` | End current group | None |

### Text Formatting

| Code | Effect |
|------|--------|
| `$B` | Bold ON |
| `$b` | Bold OFF |
| `$I` | Italic ON |
| `$i` | Italic OFF |
| `$U` | Underline ON |
| `$u` | Underline OFF |
| `$$` | Literal `$` character |

---

## 28.3 Example Terminal Script

```
#LOGON 1234
Welcome to Terminal 47.
#END

#UNFINISHED
$BObjective:$b Find the primary reactor.

The pfhor have overrun this section. Proceed with
$Iextreme$i caution.
#CHECKPOINT 3
#END

#SUCCESS
$BWell done.$b The reactor is secured.

Proceed to the extraction point.
#INTERLEVEL TELEPORT 5
#END
```

---

## 28.4 Preprocessed Terminal Format

Terminal scripts are compiled into efficient binary format:

### Header Structure

```c
struct static_preprocessed_terminal_data {
    short total_length;       // Total bytes of terminal data
    short flags;              // _text_is_encoded_flag (0x0001)
    short lines_per_page;     // Lines visible per screen
    short grouping_count;     // Number of terminal_groupings
    short font_changes_count; // Number of text_face_data entries
};
```

### Binary Layout

```
┌─────────────────────────────────────────────────────────────────┐
│  static_preprocessed_terminal_data (10 bytes)                    │
├─────────────────────────────────────────────────────────────────┤
│  terminal_groupings[grouping_count] (12 bytes each)              │
│    ├─ Group 0: type, permutation, start_index, length           │
│    ├─ Group 1: ...                                               │
│    └─ Group N: ...                                               │
├─────────────────────────────────────────────────────────────────┤
│  text_face_data[font_changes_count] (6 bytes each)               │
│    ├─ Face 0: index, face, color                                 │
│    └─ Face N: ...                                                │
├─────────────────────────────────────────────────────────────────┤
│  char text[] (raw text with formatting codes stripped)           │
└─────────────────────────────────────────────────────────────────┘
```

---

## 28.5 Terminal Grouping Structure

```c
struct terminal_groupings {
    short flags;               // _draw_object_on_right, _center_object
    short type;                // Group type enum
    short permutation;         // Type-specific parameter
    short start_index;         // Offset into text array
    short length;              // Text length for this group
    short maximum_line_count;  // Lines in this group
};
```

### Group Types

```c
enum /* group types */ {
    _logon_group,               // 0 - Login screen
    _unfinished_group,          // 1 - Mission incomplete
    _success_group,             // 2 - Mission complete
    _failure_group,             // 3 - Mission failed
    _information_group,         // 4 - General info
    _end_group,                 // 5 - End marker
    _interlevel_teleport_group, // 6 - Level transition
    _intralevel_teleport_group, // 7 - In-level teleport
    _checkpoint_group,          // 8 - Map checkpoint
    _sound_group,               // 9 - Play sound
    _movie_group,               // 10 - Play movie
    _track_group,               // 11 - Play music
    _pict_group,                // 12 - Display image
    _logoff_group,              // 13 - Logout screen
    _camera_group,              // 14 - Camera view
    _static_group,              // 15 - TV static
    _tag_group                  // 16 - Activate tags
};
```

---

## 28.6 Text Face Data

```c
struct text_face_data {
    short index;   // Character position where style changes
    short face;    // Style flags
    short color;   // Text color index
};

enum /* face flags */ {
    _plain_text     = 0x00,
    _bold_text      = 0x01,
    _italic_text    = 0x02,
    _underline_text = 0x04
};
```

### Style Processing Example

```
Source: "This is $Bbold$b text"

After preprocessing:
  text = "This is bold text"

  text_face_data[0] = { index: 0,  face: _plain_text, color: 0 }
  text_face_data[1] = { index: 8,  face: _bold_text,  color: 0 }
  text_face_data[2] = { index: 12, face: _plain_text, color: 0 }
```

---

## 28.7 Terminal State Machine

```
┌─────────────────────────────────────────────────────────────────┐
│              TERMINAL STATE MACHINE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Player activates    enter_computer_    Terminal state          │
│   control panel  ──►  interface()    ──► machine starts          │
│                                                                  │
│   _reading_terminal ──► Groups processed sequentially            │
│         │                                                        │
│         ▼                                                        │
│   ┌──────────┐   ┌──────────┐   ┌──────────┐                    │
│   │  LOGON   │──►│ CONTENT  │──►│  LOGOFF  │                    │
│   │  Screen  │   │ (groups) │   │ /Teleport│                    │
│   └──────────┘   └──────────┘   └──────────┘                    │
│                                                                  │
│   _no_terminal_state ──► Player exits terminal mode              │
└─────────────────────────────────────────────────────────────────┘
```

---

## 28.8 Player Terminal Data

```c
struct player_terminal_data {
    short flags;                   // _terminal_is_dirty
    short phase;                   // Animation/timing phase
    short state;                   // _reading_terminal or _no_terminal_state
    short current_group;           // Which group being displayed
    short level_completion_state;  // For choosing success/failure
    short current_line;            // Scroll position
    short maximum_line;            // Total lines in current group
    short terminal_id;             // Terminal being accessed
    long last_action_flag;         // For debouncing input
};
```

---

## 28.9 Terminal Navigation

| Key | Action Flag | Effect |
|-----|-------------|--------|
| Up / Page Up | `_terminal_page_up` | Scroll up |
| Down / Page Down | `_terminal_page_down` | Scroll down |
| Tab / Enter / Space | `_terminal_next_state` | Next group |
| Escape | `_any_abort_key_mask` | Exit terminal |

---

## 28.10 Terminal Display

```
┌─────────────────────────────────────────────────────────────────┐
│                         BORDER (18px)                            │
├─────────────────────────────────────────────────────────────────┤
│ ┌─────────────────────────────────────────────────────────────┐ │
│ │                                                             │ │
│ │   Marathon Terminal Network                                 │ │
│ │   ─────────────────────────                                 │ │
│ │                                                             │ │
│ │   $BObjective:$b Secure the reactor                         │ │
│ │                                                             │ │
│ │   The pfhor have breached containment.                      │ │
│ │   Proceed to sublevel 3.                                    │ │
│ │                                           ┌───────────────┐ │ │
│ │                                           │  CHECKPOINT   │ │ │
│ │                                           │     MAP       │ │ │
│ │                                           │    (goal)     │ │ │
│ │                                           └───────────────┘ │ │
│ │                                                             │ │
│ └─────────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│                    [Press SPACE to continue]                     │
└─────────────────────────────────────────────────────────────────┘
```

### Display Constants

```c
#define BORDER_HEIGHT 18
#define BORDER_INSET 9
#define LABEL_INSET 3
#define LOG_DURATION_BEFORE_TIMEOUT (2*TICKS_PER_SECOND)
#define MAXIMUM_FACE_CHANGES_PER_TEXT_GROUPING 128
```

---

## 28.11 Level Completion Detection

```c
// Completion flag determines which groups to show:
// 0 = _unfinished_group
// 1 = _success_group
// 2 = _failure_group

void enter_computer_interface(
    short player_index,
    short text_number,
    short completion_flag)
{
    // completion_flag determines narrative branch
}
```

### Completion Flow

```
Player activates terminal ──► Check mission goals
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
              Goals incomplete    Goals complete    Goals failed
                    │                 │                 │
                    ▼                 ▼                 ▼
              #UNFINISHED         #SUCCESS          #FAILURE
              groups shown        groups shown      groups shown
```

---

## 28.12 Terminal API

```c
// Initialize terminal system
void initialize_terminal_manager(void);

// Initialize player's terminal state
void initialize_player_terminal_info(short player_index);

// Enter terminal mode
void enter_computer_interface(short player_index, short text_number,
                              short completion_flag);

// Render terminal display
void _render_computer_interface(struct view_terminal_data *data);

// Update player in terminal mode
void update_player_for_terminal_mode(short player_index);

// Process terminal input
void update_player_keys_for_terminal(short player_index, long action_flags);

// Check if player is in terminal
boolean player_in_terminal_mode(short player_index);

// Exit terminal
void abort_terminal_mode(short player_index);

// Mark display as needing redraw
void dirty_terminal_view(short player_index);
```

---

## 28.13 Terminal Parsing Implementation

The actual parsing code in `computer_interface.c` demonstrates how terminal data is accessed:

### Getting Terminal Data (computer_interface.c:~400)

```c
// Extract preprocessed terminal from map data
static struct static_preprocessed_terminal_data *get_indexed_terminal_data(
    short terminal_id)
{
    struct static_preprocessed_terminal_data *terminal = NULL;

    if (map_terminal_data) {
        // Walk through terminal entries to find requested ID
        byte *data = map_terminal_data;
        short count = 0;

        while (count <= terminal_id && data < map_terminal_data + map_terminal_data_length) {
            terminal = (struct static_preprocessed_terminal_data *)data;
            if (count == terminal_id) break;
            data += terminal->total_length;
            count++;
        }
    }
    return terminal;
}
```

### Extracting Groups from Terminal Data

```c
// Get array of groupings from preprocessed terminal
struct terminal_groupings *get_terminal_groupings(
    struct static_preprocessed_terminal_data *terminal)
{
    // Groupings start immediately after header
    return (struct terminal_groupings *)(((byte *)terminal) +
           sizeof(struct static_preprocessed_terminal_data));
}

// Get text face changes array
struct text_face_data *get_terminal_text_faces(
    struct static_preprocessed_terminal_data *terminal)
{
    // Face data follows groupings
    return (struct text_face_data *)(((byte *)terminal) +
           sizeof(struct static_preprocessed_terminal_data) +
           terminal->grouping_count * sizeof(struct terminal_groupings));
}

// Get raw text buffer
char *get_terminal_text(
    struct static_preprocessed_terminal_data *terminal)
{
    // Text follows face data
    return (char *)(((byte *)terminal) +
           sizeof(struct static_preprocessed_terminal_data) +
           terminal->grouping_count * sizeof(struct terminal_groupings) +
           terminal->font_changes_count * sizeof(struct text_face_data));
}
```

### Group Processing Loop

```c
// Process terminal groups based on completion state (computer_interface.c:~520)
static short find_group_type(
    struct static_preprocessed_terminal_data *terminal,
    short group_type,
    short completion_flag)
{
    struct terminal_groupings *groups = get_terminal_groupings(terminal);

    for (short i = 0; i < terminal->grouping_count; i++) {
        if (groups[i].type == group_type) {
            // For conditional groups, check completion state
            switch (group_type) {
                case _unfinished_group:
                    if (completion_flag == 0) return i;
                    break;
                case _success_group:
                    if (completion_flag == 1) return i;
                    break;
                case _failure_group:
                    if (completion_flag == 2) return i;
                    break;
                default:
                    return i;  // Non-conditional group
            }
        }
    }
    return NONE;
}
```

### Text Decoding (if encoded)

```c
// Terminal text may be XOR-encoded for copy protection
#define TERMINAL_DECODE_KEY 0xFE  // XOR key

static void decode_terminal_text(char *text, short length) {
    for (short i = 0; i < length; i++) {
        text[i] ^= TERMINAL_DECODE_KEY;
    }
}

// Check and decode in enter_computer_interface()
if (terminal->flags & _text_is_encoded_flag) {
    char *text = get_terminal_text(terminal);
    decode_terminal_text(text, terminal->total_length - header_and_groupings_size);
    terminal->flags &= ~_text_is_encoded_flag;  // Mark as decoded
}
```

---

## 28.14 See Also

- **[Chapter 15: Control Panels](15_control_panels.md)** - How terminals are activated via switches
- **[Chapter 10: File Formats](10_file_formats.md)** - WAD tag storage for terminal data
- **[Chapter 7: Game Loop](07_game_loop.md)** - Terminal mode integration with game state

---

## 28.15 Summary

Marathon's terminal system provides:

- **Markup language** for authoring story content
- **Conditional groups** based on mission state
- **Text formatting** with bold, italic, underline
- **Multimedia support** (images, sounds, movies)
- **Teleportation** commands for level flow
- **Checkpoint maps** showing objectives

### Key Source Files

| File | Purpose |
|------|---------|
| `computer_interface.c` | Terminal logic |
| `computer_interface.h` | Terminal structures |
| `terminal_definitions.h` | Terminal data (if separate) |

---

*Next: [Chapter 29: Music/Soundtrack System](29_music.md) - Background music and audio tracks*
