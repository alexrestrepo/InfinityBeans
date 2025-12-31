# Chapter 30: Error Handling & Progress

## Game Errors, System Alerts, and Loading Displays

> **For Porting:** Error handling in `game_errors.c` is fully portable (14 lines of core logic). Progress bars in `progress.c` use Mac dialogs - replace with your UI system.

---

## 30.1 What Problem Are We Solving?

Games need robust error handling and user feedback:

- **Error propagation** - Pass errors up the call stack
- **User notification** - Display meaningful messages
- **Progress feedback** - Show loading status during map transfers
- **Recovery** - Clean up gracefully when things go wrong

---

## 30.2 Error Type System

Marathon categorizes errors into two types:

```c
enum /* error types */ {
    systemError,    // OS-level errors (file I/O, memory)
    gameError,      // Game logic errors
    NUMBER_OF_TYPES // 2
};

enum /* game errors */ {
    errNone               = 0,  // No error
    errMapFileNotSet      = 1,  // No map file loaded
    errIndexOutOfRange    = 2,  // Array bounds violation
    errTooManyOpenFiles   = 3,  // File handle exhaustion
    errUnknownWadVersion  = 4,  // Incompatible WAD format
    errWadIndexOutOfRange = 5,  // Invalid WAD entry
    errServerDied         = 6,  // Network host disconnected
    errUnsyncOnLevelChange = 7, // Multiplayer desync
    NUMBER_OF_GAME_ERRORS // 8
};
```

---

## 30.3 Error State Machine

```
┌─────────────────────────────────────────────────────────────────┐
│                   ERROR STATE MACHINE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────┐                                                │
│   │  NO ERROR   │◄───────────────────┐                          │
│   │  (default)  │                    │                          │
│   └──────┬──────┘                    │                          │
│          │                           │                          │
│          │ set_game_error()          │ clear_game_error()       │
│          ▼                           │                          │
│   ┌─────────────┐                    │                          │
│   │ ERROR SET   │────────────────────┘                          │
│   │             │                                                │
│   │ last_type   │                                                │
│   │ last_error  │                                                │
│   └──────┬──────┘                                                │
│          │                                                       │
│          │ error_pending() → TRUE                                │
│          │ get_game_error() → returns error code                 │
│          │                                                       │
│          ▼                                                       │
│   ┌─────────────┐                                                │
│   │  HANDLED    │  Caller displays alert, logs, or recovers     │
│   │             │                                                │
│   └─────────────┘                                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 30.4 Error API Implementation

```c
// Global error state (2 static variables)
static short last_type = systemError;
static short last_error = 0;

void set_game_error(short type, short error_code) {
    assert(type >= 0 && type < NUMBER_OF_TYPES);
    last_type = type;
    last_error = error_code;
#ifdef DEBUG
    if (type == gameError)
        assert(error_code >= 0 && error_code < NUMBER_OF_GAME_ERRORS);
#endif
}

short get_game_error(short *type) {
    if (type) {
        *type = last_type;
    }
    return last_error;
}

boolean error_pending(void) {
    return (last_error != 0);
}

void clear_game_error(void) {
    last_error = 0;
    last_type = 0;
}
```

### Error Flow Example

```
load_map():
    │
    ├─► File doesn't exist?
    │       set_game_error(systemError, errno)
    │       return FALSE
    │
    ├─► WAD version wrong?
    │       set_game_error(gameError, errUnknownWadVersion)
    │       return FALSE
    │
    └─► Success
            return TRUE

Caller:
    if (!load_map(filename)) {
        short type;
        short code = get_game_error(&type);

        if (type == systemError)
            display_system_alert(code);
        else
            display_game_alert(code);

        clear_game_error();
    }
```

---

## 30.5 Error Categories

| Type | Source | Typical Response |
|------|--------|------------------|
| **systemError** | OS calls (file, memory) | Display system message |
| **gameError** | Game logic | Display game-specific message |

### System Errors

System errors use OS error codes (Mac OSErr):

```
Common Mac OS Errors:
  fnfErr  (-43)  File not found
  ioErr   (-36)  I/O error
  memFullErr (-108)  Out of memory
  dirNFErr (-120) Directory not found
```

### Game Errors

| Error | Cause | Recovery |
|-------|-------|----------|
| `errMapFileNotSet` | Started game without loading map | Return to menu |
| `errIndexOutOfRange` | Corrupt or invalid data | Abort operation |
| `errTooManyOpenFiles` | File handle leak | Close unused files |
| `errUnknownWadVersion` | Old/incompatible map file | Show compatibility error |
| `errWadIndexOutOfRange` | Map references invalid entry | Skip or abort |
| `errServerDied` | Network host disconnected | Return to menu |
| `errUnsyncOnLevelChange` | Multiplayer desync | End network game |

---

## 30.6 Progress Dialog System

Marathon shows progress during lengthy operations (map loading, network transfers).

### Progress Messages

```c
enum {
    strPROGRESS_MESSAGES = 143,  // String resource ID
    _distribute_map_single = 0,  // "Distributing map..."
    _distribute_map_multiple,    // "Distributing maps..."
    _receiving_map,              // "Receiving map..."
    _awaiting_map,               // "Awaiting map..."
    _distribute_physics_single,  // "Distributing physics..."
    _distribute_physics_multiple,// "Distributing physics..."
    _receiving_physics           // "Receiving physics..."
};
```

### Progress Data Structure

```c
struct progress_data {
    DialogPtr dialog;             // Mac dialog window
    GrafPtr old_port;             // Saved graphics port
    UserItemUPP progress_bar_upp; // Progress bar callback
};
```

---

## 30.7 Progress Bar Visualization

```
┌─────────────────────────────────────────────────────────────────┐
│                      PROGRESS DIALOG                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│           ┌─────────────────────────────────────────┐           │
│           │  "Receiving map..."                     │           │
│           └─────────────────────────────────────────┘           │
│                                                                  │
│           ┌─────────────────────────────────────────┐           │
│           │░░░░░░░░░░░░░░░░░░│                      │           │
│           └─────────────────────────────────────────┘           │
│                 ▲                 ▲                               │
│                 │                 │                               │
│            Filled             Unfilled                           │
│          (sent/total)        (remaining)                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

Progress Calculation:
    width = (sent * bar_width) / total

Example:
    sent = 50KB, total = 200KB, bar_width = 200px
    filled_width = (50 * 200) / 200 = 50px
```

---

## 30.8 Progress API

```c
// Open progress dialog with initial message
void open_progress_dialog(short message_id);

// Update the message text
void set_progress_dialog_message(short message_id);

// Update progress bar fill
void draw_progress_bar(long sent, long total);

// Reset progress bar to empty
void reset_progress_bar(void);

// Close and cleanup
void close_progress_dialog(void);
```

### Usage Flow

```
open_progress_dialog(_distribute_map_single);
    │
    ├─► For each chunk sent:
    │       draw_progress_bar(bytes_sent, total_bytes);
    │
    ├─► Message change needed?
    │       set_progress_dialog_message(_receiving_physics);
    │       reset_progress_bar();
    │
    └─► Done
        close_progress_dialog();
```

---

## 30.9 Network Transfer Progress

Progress dialogs are primarily used for multiplayer map distribution:

```
┌─────────────────────────────────────────────────────────────────┐
│              NETWORK MAP DISTRIBUTION                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   HOST                              CLIENT                       │
│   ────                              ──────                       │
│                                                                  │
│   open_progress_dialog              open_progress_dialog         │
│   (_distribute_map_single)          (_awaiting_map)              │
│          │                                  │                    │
│          ▼                                  ▼                    │
│   For each chunk:                   For each chunk received:     │
│     send_chunk()                      receive_chunk()            │
│     draw_progress_bar()               draw_progress_bar()        │
│          │                                  │                    │
│          ▼                                  ▼                    │
│   close_progress_dialog             close_progress_dialog        │
│                                                                  │
│   Total transfer time: ~5-30 seconds depending on map size      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 30.10 Debug Assertions

Marathon uses assertions extensively for development-time error detection:

```c
// From cseries.h
#ifdef DEBUG
    #define assert(expression) \
        if (!(expression)) \
            _assertion_failure(#expression, __FILE__, __LINE__)

    #define vassert(expression, message) \
        if (!(expression)) \
            _assertion_failure(message, __FILE__, __LINE__)

    #define halt() \
        _assertion_failure("halt", __FILE__, __LINE__)

    void _assertion_failure(char *information, char *file, int line);
#else
    #define assert(expression) ((void) 0)
    #define vassert(expression, message) ((void) 0)
    #define halt() ((void) 0)
#endif
```

### Assertion vs Error

| Feature | assert() | set_game_error() |
|---------|----------|------------------|
| **When** | Development bugs | Runtime conditions |
| **Action** | Halt immediately | Set flag for handling |
| **Release build** | Compiled out | Always active |
| **Recovery** | No (program stops) | Yes (caller handles) |

---

## 30.11 Porting Considerations

### Error System (Portable)

The error system is **entirely portable** - just 14 lines of core logic:

```c
// No changes needed for:
static short last_type, last_error;
void set_game_error(short, short);
short get_game_error(short*);
boolean error_pending(void);
void clear_game_error(void);
```

### Progress Dialog (Replace)

The progress dialog uses Mac-specific APIs:

| Mac API | Replacement |
|---------|-------------|
| `GetNewDialog()` | Create window/UI element |
| `DialogPtr` | Your UI handle type |
| `GetDItem`/`SetDItem` | Access UI elements |
| `DrawDialog` | Render UI |
| `SetCursor(watchCursor)` | Set busy cursor |

**Simple Replacement Option:**

```c
// Console-based progress for porting
void open_progress_dialog(short message_id) {
    printf("%s\n", progress_messages[message_id]);
}

void draw_progress_bar(long sent, long total) {
    int percent = (sent * 100) / total;
    printf("\rProgress: %d%%", percent);
    fflush(stdout);
}

void close_progress_dialog(void) {
    printf("\nDone.\n");
}
```

---

## 30.12 Summary

Marathon's error system provides:

- **Two-tier errors** (system vs game)
- **Simple state machine** (set, check, clear)
- **Progress feedback** for long operations
- **Debug assertions** for development
- **Clean separation** (portable errors, Mac-specific UI)

### Key Source Files

| File | Purpose |
|------|---------|
| `game_errors.c` | Error state management (portable) |
| `game_errors.h` | Error types and codes |
| `progress.c` | Progress dialog (Mac-specific) |
| `progress.h` | Progress API |

---

*Next: [Chapter 31: Resource Forks Guide](31_resource_forks.md) - Mac-specific file format handling*
