# Chapter 28: Computer Terminal System

## Story Content, Checkpoints, and Level Transitions

> **Source files**: `computer_interface.c`, `computer_interface.h`
> **Related chapters**: [Chapter 15: Control Panels](15_control_panels.md), [Chapter 10: File Formats](10_file_formats.md)

> **For Porting:** Terminal logic in `computer_interface.c` is mostly portable (2364 lines). Replace Mac text rendering (QuickDraw) with your font system. The preprocessed terminal format is cross-platform.

---

## 28.1 What Problem Are We Solving?

Marathon tells its story through in-game computer terminals that:

- **Deliver narrative** - Plot, backstory, character dialog
- **Provide objectives** - Mission briefings with checkpoints
- **Enable teleportation** - Level-to-level and within-level travel
- **Display multimedia** - Images, sounds, movies

---

## 28.2 Terminal Scripting Language (`computer_interface.h:6-56`)

The header documents the terminal markup language:

```c
/*
    New paradigm:
    Groups each start with one of the following groups:
     #UNFINISHED, #SUCCESS, #FAILURE

    First is shown the
    #LOGON XXXXX

    Then there are any number of groups with:
    #INFORMATION, #CHECKPOINT, #SOUND, #MOVIE, #TRACK

    And a final:
    #INTERLEVEL TELEPORT, #INTRALEVEL TELEPORT

    Each group ends with:
    #END

    Groupings:
    #logon XXXX- login message (XXXX is shape for login screen)
    #unfinished- unfinished message
    #success- success message
    #failure- failure message
    #information- information
    #briefing XX- briefing, then load XX
    #checkpoint XX- Checkpoint xx (associated with goal)
    #sound XXXX- play sound XXXX
    #movie XXXX- play movie XXXX (from Movie file)
    #track XXXX- play soundtrack XXXX (from Music file)
    #interlevel teleport XXX- go to level XXX
    #intralevel teleport XXX- go to polygon XXX
    #pict XXXX- display the pict resource XXXX

    Special embedded keys:
    $B- Bold on
    $b- bold off
    $I- Italic on
    $i- italic off
    $U- underline on
    $u- underline off
    $- anything else is passed through unchanged
*/
```

---

## 28.3 Display Constants (`computer_interface.c:38-42`)

```c
#define LABEL_INSET 3
#define LOG_DURATION_BEFORE_TIMEOUT (2*TICKS_PER_SECOND)
#define BORDER_HEIGHT 18
#define BORDER_INSET 9
#define FUDGE_FACTOR 1
```

---

## 28.4 Terminal States (`computer_interface.c:44-52`)

```c
enum {
    _reading_terminal,
    _no_terminal_state,
    NUMBER_OF_TERMINAL_STATES
};

enum {
    _terminal_is_dirty= 0x01
};
```

### Dirty Flag Macros (`computer_interface.c:78-79`)

```c
#define TERMINAL_IS_DIRTY(term) ((term)->flags & _terminal_is_dirty)
#define SET_TERMINAL_IS_DIRTY(term, v) ((v)? ((term)->flags |= _terminal_is_dirty) : ((term)->flags &= ~_terminal_is_dirty))
```

---

## 28.5 Terminal Input Actions (`computer_interface.c:54-61`)

```c
enum {
    _any_abort_key_mask= _action_trigger_state,
    _terminal_up_arrow= _moving_forward,
    _terminal_down_arrow= _moving_backward,
    _terminal_page_down= _turning_right,
    _terminal_page_up= _turning_left,
    _terminal_next_state= _left_trigger_state
};
```

### Key Mappings (`computer_interface.c:171-183`)

```c
static struct terminal_key terminal_keys[]= {
    {0x7e, 0, 0, _terminal_page_up},  // arrow up
    {0x7d, 0, 0, _terminal_page_down},// arrow down
    {0x74, 0, 0, _terminal_page_up},   // page up
    {0x79, 0, 0, _terminal_page_down}, // page down
    {0x30, 0, 0, _terminal_next_state}, // tab
    {0x4c, 0, 0, _terminal_next_state}, // enter
    {0x24, 0, 0, _terminal_next_state}, // return
    {0x31, 0, 0, _terminal_next_state}, // space
    {0x3a, 0, 0, _terminal_next_state}, // command
    {0x35, 0, 0, _any_abort_key_mask}  // escape
};
#define NUMBER_OF_TERMINAL_KEYS (sizeof(terminal_keys)/sizeof(struct terminal_key))
```

---

## 28.6 Group Type Enumeration (`computer_interface.c:88-108`)

```c
enum {
    _logon_group,
    _unfinished_group,
    _success_group,
    _failure_group,
    _information_group,
    _end_group,
    _interlevel_teleport_group, // permutation is level to go to
    _intralevel_teleport_group, // permutation is polygon to go to.
    _checkpoint_group, // permutation is the goal to show
    _sound_group, // permutation is the sound id to play
    _movie_group, // permutation is the movie id to play
    _track_group, // permutation is the track to play
    _pict_group, // permutation is the pict to display
    _logoff_group,
    _camera_group, //  permutation is the object index
    _static_group, // permutation is the duration of static.
    _tag_group, // permutation is the tag to activate

    NUMBER_OF_GROUP_TYPES
};
```

---

## 28.7 Text Face Flags (`computer_interface.c:110-116`)

```c
enum // flags to indicate text styles for paragraphs
{
    _plain_text      = 0x00,
    _bold_text       = 0x01,
    _italic_text     = 0x02,
    _underline_text  = 0x04
};
```

---

## 28.8 Terminal Grouping Flags (`computer_interface.c:118-121`)

```c
enum { /* terminal grouping flags */
    _draw_object_on_right= 0x01,  // for drawing checkpoints, picts, movies.
    _center_object= 0x02
};
```

---

## 28.9 Preprocessed Terminal Header (`computer_interface.h:59-65`)

```c
struct static_preprocessed_terminal_data {
    short total_length;
    short flags;
    short lines_per_page; /* Added for internationalization/sync problems */
    short grouping_count;
    short font_changes_count;
};
```

### Encoding Flag (`computer_interface.c:84-86`)

```c
enum {
    _text_is_encoded_flag= 0x0001
};
```

---

## 28.10 Terminal Groupings Structure (`computer_interface.c:123-130`)

```c
struct terminal_groupings {
    short flags; /* varies.. */
    short type; /* _information_text, _checkpoint_text, _briefing_text, _movie, _sound_bite, _soundtrack */
    short permutation; /* checkpoint id for chkpt, level id for _briefing, movie id for movie, sound id for sound, soundtrack id for soundtrack */
    short start_index;
    short length;
    short maximum_line_count;
};
```

---

## 28.11 Text Face Data Structure (`computer_interface.c:132-136`)

```c
struct text_face_data {
    short index;
    short face;
    short color;
};
```

---

## 28.12 Player Terminal Data (`computer_interface.c:138-149`)

```c
struct player_terminal_data
{
    short flags;
    short phase;
    short state;
    short current_group;
    short level_completion_state;
    short current_line;
    short maximum_line;
    short terminal_id;
    long last_action_flag;
};
```

---

## 28.13 View Terminal Data (`computer_interface.h:67-70`)

```c
struct view_terminal_data {
    short top, left, bottom, right;
    short vertical_offset;
};
```

---

## 28.14 Global Terminal Data (`computer_interface.c:164-165`, `computer_interface.h:72-73`)

```c
byte *map_terminal_data;
long map_terminal_data_length;
```

---

## 28.15 Binary Layout

```
┌─────────────────────────────────────────────────────────────────┐
│  static_preprocessed_terminal_data (10 bytes)                    │
├─────────────────────────────────────────────────────────────────┤
│  terminal_groupings[grouping_count] (12 bytes each)              │
│    ├─ Group 0: flags, type, permutation, start_index, length    │
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

## 28.16 Terminal API (`computer_interface.h:76-93`)

```c
void initialize_terminal_manager(void);
void initialize_player_terminal_info(short player_index);
void enter_computer_interface(short player_index, short text_number, short completion_flag);
void _render_computer_interface(struct view_terminal_data *data);
void update_player_for_terminal_mode(short player_index);
void update_player_keys_for_terminal(short player_index, long action_flags);
long build_terminal_action_flags(char *keymap);
void dirty_terminal_view(short player_index);
void abort_terminal_mode(short player_index);

boolean player_in_terminal_mode(short player_index);

void *get_terminal_data_for_save_game(void);
long calculate_terminal_data_length(void);

/* This returns the text.. */
void *get_terminal_information_array(void);
long calculate_terminal_information_length(void);
```

---

## 28.17 Enter Computer Interface (`computer_interface.c:292-326`)

```c
void enter_computer_interface(
    short player_index,
    short text_number,
    short completion_flag)
{
    struct player_terminal_data *terminal= get_player_terminal_data(player_index);
    struct player_data *player= get_player_data(player_index);
    struct static_preprocessed_terminal_data *terminal_text= get_indexed_terminal_data(text_number);

    if(dynamic_world->player_count==1)
    {
        short lines_per_page;

        /* Reset the lines per page to the actual value for whatever fucked up font that they have */
        lines_per_page= calculate_lines_per_page();
        if(lines_per_page != terminal_text->lines_per_page)
        {
            terminal_text->lines_per_page= lines_per_page;
        }
    }

    /* Tell everyone that this player is in the computer interface.. */
    terminal->state= _reading_terminal;
    terminal->phase= NONE;
    terminal->current_group= NONE;
    terminal->level_completion_state= completion_flag;
    terminal->current_line= 0;
    terminal->maximum_line= 1; // any click or keypress will get us out.
    terminal->terminal_id= text_number;
    terminal->last_action_flag= -1l; /* Eat the first key */

    /* And select the first one. */
    next_terminal_group(player_index, terminal_text);
}
```

---

## 28.18 Get Indexed Terminal Data (`computer_interface.c:1093-1112`)

```c
static struct static_preprocessed_terminal_data *get_indexed_terminal_data(
    short id)
{
    struct static_preprocessed_terminal_data *data;
    long offset= 0l;
    short index= id;

    data= (struct static_preprocessed_terminal_data *) (map_terminal_data);
    while(index>0) {
        vassert(offset<map_terminal_data_length, csprintf(temporary, "Unable to get data for terminal: %d", id));
        offset+= data->total_length;
        data= (struct static_preprocessed_terminal_data *) (map_terminal_data+offset);
        index--;
    }

    /* Note that this will only decode the text once. */
    decode_text(data);

    return data;
}
```

---

## 28.19 Data Accessor Functions (`computer_interface.c:2311-2351`)

### Get Indexed Grouping (`computer_interface.c:2311-2323`)

```c
static struct terminal_groupings *get_indexed_grouping(
    struct static_preprocessed_terminal_data *data,
    short index)
{
    byte *start;

    assert(index>=0 && index<data->grouping_count);
    start= (byte *) data;
    start += sizeof(struct static_preprocessed_terminal_data) +
        index*sizeof(struct terminal_groupings);

    return (struct terminal_groupings *) start;
}
```

### Get Indexed Font Changes (`computer_interface.c:2325-2337`)

```c
static struct text_face_data *get_indexed_font_changes(
    struct static_preprocessed_terminal_data *data,
    short index)
{
    byte *start;

    assert(index>=0 && index<data->font_changes_count);
    start= (byte *) data;
    start += sizeof(struct static_preprocessed_terminal_data) +
        data->grouping_count*sizeof(struct terminal_groupings)+
        index*sizeof(struct text_face_data);

    return (struct text_face_data *) start;
}
```

### Get Text Base (`computer_interface.c:2340-2351`)

```c
static char *get_text_base(
    struct static_preprocessed_terminal_data *data)
{
    byte *start;

    start= (byte *) data;
    start += sizeof(struct static_preprocessed_terminal_data) +
        data->grouping_count*sizeof(struct terminal_groupings)+
        data->font_changes_count*sizeof(struct text_face_data);

    return (char *) start;
}
```

---

## 28.20 Text Encoding/Decoding (`computer_interface.c:1119-1166`)

Terminal text can be XOR-encoded for copy protection:

### Decode Text (`computer_interface.c:1119-1129`)

```c
static void decode_text(
    struct static_preprocessed_terminal_data *terminal_text)
{
    if(terminal_text->flags & _text_is_encoded_flag)
    {
        encode_text(terminal_text);

        terminal_text->flags &= ~_text_is_encoded_flag;
    }
}
```

### Encode Text (`computer_interface.c:1135-1166`)

```c
static void encode_text(
    struct static_preprocessed_terminal_data *terminal_text)
{
    char *text_base= get_text_base(terminal_text);
    short index;
    long length;
    long *long_offset;
    char *byte_offset;

    length= terminal_text->total_length-
        (sizeof(struct static_preprocessed_terminal_data) +
        terminal_text->grouping_count*sizeof(struct terminal_groupings)+
        terminal_text->font_changes_count*sizeof(struct text_face_data));

    long_offset= (long *) text_base;
    for(index= 0; index<length/sizeof(long); ++index)
    {
        (*long_offset) ^= 0xfeed;
        long_offset++;
    }

    /* And get the last bytes */
    byte_offset= (char *) long_offset;
    for(index= 0; index<length%sizeof(long); ++index)
    {
        (*byte_offset) ^= 0xfe;
        byte_offset++;
    }

    terminal_text->flags |= _text_is_encoded_flag;
}
```

---

## 28.21 Level Completion State (`computer_interface.c:1376-1410`)

```c
static void next_terminal_group(
    short player_index,
    struct static_preprocessed_terminal_data *terminal_text)
{
    struct player_terminal_data *terminal_data= get_player_terminal_data(player_index);

    if(terminal_data->current_group==NONE)
    {
        switch(terminal_data->level_completion_state)
        {
            case _level_unfinished:
                terminal_data->current_group= find_group_type(terminal_text, _unfinished_group);
                break;

            case _level_finished:
                terminal_data->current_group= find_group_type(terminal_text, _success_group);
                if(terminal_data->current_group==terminal_text->grouping_count)
                {
                    /* Fallback. */
                    terminal_data->current_group= find_group_type(terminal_text, _unfinished_group);
                }
                break;

            case _level_failed:
                terminal_data->current_group= find_group_type(terminal_text, _failure_group);
                if(terminal_data->current_group==terminal_text->grouping_count)
                {
                    /* Fallback. */
                    terminal_data->current_group= find_group_type(terminal_text, _unfinished_group);
                }
                break;
        }
        /* ... */
    }
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

## 28.22 Teleportation Functions (`computer_interface.c:932-951`)

### Interlevel Teleport (`computer_interface.c:932-941`)

```c
static void teleport_to_level(
    short level_number)
{
    /* It doesn't matter which player we get. */
    struct player_data *player= get_player_data(0);

    assert(level_number != 0);
    player->teleporting_destination= -level_number;
    player->delay_before_teleport= TICKS_PER_SECOND/2; // delay before we teleport.
}
```

### Intralevel Teleport (`computer_interface.c:943-951`)

```c
static void teleport_to_polygon(
    short player_index,
    short polygon_index)
{
    struct player_data *player= get_player_data(player_index);

    player->teleporting_destination= polygon_index;
    assert(!player->delay_before_teleport);
}
```

---

## 28.23 Terminal Display

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

---

## 28.24 Maximum Face Changes (`computer_interface.c:82`)

```c
/* Maximum face changes per text grouping.. */
#define MAXIMUM_FACE_CHANGES_PER_TEXT_GROUPING (128)
```

---

## 28.25 Summary

Marathon's terminal system provides:

- **17 group types** for varied content (`computer_interface.c:88-108`)
- **Markup language** documented in header (`computer_interface.h:6-56`)
- **Conditional groups** based on mission state (`computer_interface.c:1376-1410`)
- **Text formatting** with bold, italic, underline (`computer_interface.c:110-116`)
- **XOR text encoding** for copy protection (`computer_interface.c:1135-1166`)
- **Teleportation** commands for level flow (`computer_interface.c:932-951`)

### Key Constants

| Constant | Value | Source |
|----------|-------|--------|
| `BORDER_HEIGHT` | 18 | `computer_interface.c:40` |
| `BORDER_INSET` | 9 | `computer_interface.c:41` |
| `LOG_DURATION_BEFORE_TIMEOUT` | 60 ticks | `computer_interface.c:39` |
| `MAXIMUM_FACE_CHANGES_PER_TEXT_GROUPING` | 128 | `computer_interface.c:82` |
| `NUMBER_OF_GROUP_TYPES` | 17 | `computer_interface.c:107` |
| XOR key (long) | 0xfeed | `computer_interface.c:1153` |
| XOR key (byte) | 0xfe | `computer_interface.c:1161` |

### Key Source Files

| File | Purpose |
|------|---------|
| `computer_interface.c` | Terminal logic (2364 lines) |
| `computer_interface.h` | Structures and prototypes (105 lines) |

---

## 28.26 See Also

- [Chapter 15: Control Panels](15_control_panels.md) — How terminals are activated via switches
- [Chapter 10: File Formats](10_file_formats.md) — WAD tag storage for terminal data
- [Chapter 7: Game Loop](07_game_loop.md) — Terminal mode integration with game state

---

*Next: [Chapter 29: Music/Soundtrack System](29_music.md) - Background music and audio tracks*
