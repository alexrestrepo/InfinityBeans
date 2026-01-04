# Appendix M: Debug Build Features

## Development-Time Debugging and Diagnostics

> **Source files**: `cseries.lib/cseries.h`, `cseries.lib/macintosh_utilities.c`, `marathon2/weapons.c`, `marathon2/network.c`

This appendix documents all debug features available when Marathon is compiled with `#define DEBUG`. These features are compiled out in release builds (`#define FINAL`).

---

## M.1 Build Configuration

### Configuration Headers

**Debug mode** (`cseries.lib/beta.h:1`):
```c
#define DEBUG
#define BETA
// #define FINAL
```

**Release mode** (`cseries.lib/final.h:5`):
```c
// #define DEBUG
// #define BETA
#define FINAL
```

### Conditional Compilation

All debug features are wrapped in `#ifdef DEBUG` blocks:
```c
#ifdef DEBUG
    // Debug code active
#else
    // Empty macros (no overhead in release)
#endif
```

---

## M.2 Assertion System

### Assertion Macros (`cseries.h:61-79`)

| Macro | Type | Purpose |
|-------|------|---------|
| `halt()` | Fatal | Unconditional halt with stack trace |
| `vhalt(diag)` | Fatal | Halt with custom diagnostic message |
| `assert(expr)` | Fatal | Halt if expression is false |
| `vassert(expr,diag)` | Fatal | Assert with custom message |
| `warn(expr)` | Non-fatal | Warning if expression is false (continues) |
| `vwarn(expr,diag)` | Non-fatal | Warning with custom message |
| `pause()` | Non-fatal | Breakpoint without message |
| `vpause(diag)` | Non-fatal | Breakpoint with diagnostic |

### Implementation

```c
// cseries.h:61-79
#ifdef DEBUG
    #define halt() _assertion_failure((char *)NULL, __FILE__, __LINE__, TRUE)
    #define vhalt(diag) _assertion_failure(diag, __FILE__, __LINE__, TRUE)
    #define assert(expr) if (!(expr)) _assertion_failure(#expr, __FILE__, __LINE__, TRUE)
    #define vassert(expr,diag) if (!(expr)) _assertion_failure(diag, __FILE__, __LINE__, TRUE)
    #define warn(expr) if (!(expr)) _assertion_failure(#expr, __FILE__, __LINE__, FALSE)
    #define vwarn(expr,diag) if (!(expr)) _assertion_failure(diag, __FILE__, __LINE__, FALSE)
    #define pause() _assertion_failure((char *)NULL, __FILE__, __LINE__, FALSE)
    #define vpause(diag) _assertion_failure(diag, __FILE__, __LINE__, FALSE)
#else
    #define halt()
    #define vhalt(diag)
    #define assert(expr)
    #define vassert(expr,diag)
    #define warn(expr)
    #define vwarn(expr,diag)
    #define pause()
    #define vpause(diag)
#endif
```

### Assertion Handler (`macintosh_utilities.c:311-357`)

```c
void _assertion_failure(char *assertion, char *file, int line, boolean fatal)
{
    if (debugger_installed && DEBUG) {
        // Send to Mac debugger (MacsBug) via DebugStr()
        // Format: "[halt|pause] in <file>,#<line>: <message>"
    } else {
        // Show alert dialog with file and line
        // Capture screenshot via FKEY 3
        if (fatal) ExitToShell();
    }
}
```

---

## M.3 Debug Output System

### dprintf() Function (`cseries.h:190`, `macintosh_utilities.c:1010-1044`)

```c
int dprintf(const char *format, ...)
```

Printf-style debug output that can be toggled at runtime.

**Behavior**:
- Only outputs when `debug_status == TRUE`
- If debugger installed: sends to DebugStr()
- If no debugger: shows alert dialog
- Message limited to 255 characters

**Control Functions**:
```c
void initialize_debugger(boolean force_debugger_on);  // cseries.h:189
boolean toggle_debug_status(void);                     // cseries.h:187
```

### Usage Examples

```c
// player.c:192 - Player data logging
dprintf("Player %d: health=%d, oxygen=%d", player_index, health, oxygen);

// network.c:1020 - Network statistics
dprintf("Flags processed: %d Time: %d;g", flags_count, net_time);

// wad.c - WAD structure dumps
dprintf("WAD header: version=%d, count=%d", header.version, header.wad_count);
```

---

## M.4 Debug Keyboard Shortcuts

### Available Shortcuts (`shell.c:463-477`)

| Key | Condition | Function |
|-----|-----------|----------|
| `!` | `#ifndef FINAL && #ifdef DEBUG` | `debug_print_weapon_status()` |
| `?` | Always | Toggle FPS display |

### Weapon Status Debug (`weapons.c:912-959`)

Pressing `!` during gameplay outputs:
```
Current weapon: 3
Desired weapon: 3
Weapon 0: type=0 flags=0x0001 trigger[0]={...} trigger[1]={...}
Weapon 1: type=1 flags=0x0003 ...
...
```

**Helper functions**:
- `debug_print_weapon_status()` (line 912)
- `debug_weapon()` (line 926)
- `debug_trigger_data()` (line 951)

---

## M.5 Debug Flags

### DEBUG_NET (`network.c`)

34 conditional blocks for network packet tracing:

```c
#ifdef DEBUG_NET
    dprintf("Ring packet received: type=%d, seq=%ld", packet->type, packet->sequence);
    dprintf("Action flags: player=%d, count=%d", player_index, flag_count);
#endif
```

### DEBUG_REPLAY (`vbl.c:129`)

```c
// #define DEBUG_REPLAY  // Commented out by default
```

When enabled, provides:
- `open_stream_file()` - Record action flags to file
- `debug_stream_of_flags()` - Dump flag stream
- `close_stream_file()` - Close recording

### DEBUG_MODEM (`network_modem_protocol.c:34`)

```c
#define DEBUG_MODEM 1
```

Tracks modem statistics:
```c
struct modem_stats_data {
    long client_packets_sent;
    long server_packets_sent;
    long action_flags_processed;
    long numSmears;
    long stream_packets_necessary;
    long stream_packets_sent;
    long stream_early_acks;
    // ... more fields
};
```

Function: `print_modem_stats()` (line 1220)

### AUTOMAP_DEBUG (`render.c:57`)

```c
// #define AUTOMAP_DEBUG  // Commented out
```

When enabled, clears automap buffers for visibility debugging:
```c
#ifdef AUTOMAP_DEBUG
    memset(automap_lines, 0, ...);
    memset(automap_polygons, 0, ...);
#endif
```

### QUICKDRAW_DEBUG (`render.c:58`)

```c
// #define QUICKDRAW_DEBUG  // Commented out
```

Switches between debug and production graphics headers.

---

## M.6 Bounds Checking

### SET_OBJECT_OWNER Macro (`map.h:284`)

```c
#define SET_OBJECT_OWNER(o,n) { \
    assert((n)>=0&&(n)<=7);  /* Bounds check: owner must be 0-7 */ \
    (o)->flags&= (word)~7; \
    (o)->flags|= (n); \
}
```

Used in 10 locations across:
- player.c
- monsters.c
- items.c
- effects.c
- scenery.c
- projectiles.c

### Valid Owner Types

```c
enum {
    _object_is_normal,      // 0
    _object_is_scenery,     // 1
    _object_is_monster,     // 2
    _object_is_projectile,  // 3
    _object_is_effect,      // 4
    _object_is_item,        // 5
    _object_is_device,      // 6
    _object_is_garbage      // 7
};
```

---

## M.7 Performance Toggles

### Runtime Performance Controls (`interface.h:154`, `vbl.h:22`)

```c
extern boolean no_frame_rate_limit;   // Uncapped frame rate
extern boolean displaying_fps;         // Show FPS counter

void toggle_ludicrous_speed(boolean ludicrous_speed);
```

### Keyboard Controls (`shell.c:532-560`)

| Key | Platform | Effect |
|-----|----------|--------|
| F10 + Shift | PowerPC | Toggle ludicrous speed (networked) |
| F6 | PowerPC | Toggle frame rate limit |

---

## M.8 Debugger Detection

### initialize_debugger() (`macintosh_utilities.c:988-1008`)

```c
void initialize_debugger(boolean force_debugger_on)
{
    // 68K Macs: Check address 0x120 for MacsBug signature
    // PowerPC Macs: Check for 'dbug' resource #128
    debugger_installed = /* detected */;
}
```

Called during initialization:
- `shell.c:267` (main startup)
- `export_definitions.c:53`
- `serial_numbers.c:119`

---

## M.9 Usage Statistics

| Metric | Count |
|--------|-------|
| Total `#ifdef DEBUG` blocks | 157 |
| Total debug macro calls | 472 |
| Files with debug features | 47 |
| dprintf() call sites | 47 |

### Primary Debug File Locations

| File | Purpose |
|------|---------|
| `cseries.lib/cseries.h` | Macro definitions |
| `cseries.lib/macintosh_utilities.c` | Implementation |
| `marathon2/network.c` | Network debugging |
| `marathon2/weapons.c` | Weapon state debugging |
| `marathon2/render.c` | Rendering debugging |
| `marathon2/vbl.c` | Input/timing debugging |

---

## M.10 Porting Considerations

### Modern Replacements

| Mac Feature | Modern Replacement |
|-------------|-------------------|
| `DebugStr()` | `fprintf(stderr, ...)` or debugger breakpoint |
| MacsBug | GDB, LLDB, Visual Studio debugger |
| Alert dialogs | Console output or log files |
| FKEY screenshot | Platform screenshot API |

### Recommended Implementation

```c
// Modern assertion handler
void _assertion_failure(char *assertion, char *file, int line, boolean fatal) {
    fprintf(stderr, "[%s] %s:%d: %s\n",
        fatal ? "FATAL" : "WARN", file, line, assertion ? assertion : "");

    if (fatal) {
        #ifdef _WIN32
            __debugbreak();
        #elif defined(__GNUC__)
            __builtin_trap();
        #else
            abort();
        #endif
    }
}

// Modern dprintf
int dprintf(const char *format, ...) {
    if (!debug_status) return 0;

    va_list args;
    va_start(args, format);
    int result = vfprintf(stderr, format, args);
    va_end(args);
    fprintf(stderr, "\n");
    return result;
}
```

---

## M.11 Summary

Marathon's debug system provides:

- **Assertion macros** for development-time error detection
- **dprintf output** for runtime diagnostics
- **Keyboard shortcuts** for in-game debugging
- **Specialized flags** for network, replay, and rendering
- **Bounds checking** via assert-wrapped macros
- **Performance toggles** for profiling

All debug features compile to no-ops in release builds, ensuring zero runtime overhead in production.

---

*Return to: [Table of Contents](README.md)*
