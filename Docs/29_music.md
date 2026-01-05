# Chapter 29: Music/Soundtrack System

## Background Music, Track Looping, and Fades

> **For Porting:** Music logic uses Mac Sound Manager. Replace with your audio library (e.g., miniaudio, SDL_mixer). The song structure and state machine are portable concepts.

---

## 29.1 What Problem Are We Solving?

Marathon needs background music that:

- **Loops appropriately** - Intro → chorus (loop) → outro
- **Responds to events** - Fade out on level end
- **Supports multiple tracks** - Different music per level
- **Manages resources** - Streaming from disk

---

## 29.2 Music State Machine

```
┌─────────────────────────────────────────────────────────────────┐
│                    MUSIC STATE MACHINE                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   _no_song_playing                                              │
│          │                                                      │
│          │ queue_song(index)                                    │
│          ▼                                                      │
│   _delaying_for_loop ──────────────────────┐                    │
│          │                                  │                   │
│          │ delay expires                    │                   │
│          ▼                                  │                   │
│   _playing_introduction                     │                   │
│          │                                  │                   │
│          │ intro complete                   │                   │
│          ▼                                  │                   │
│   _playing_chorus ◄────────────────────────┐│                   │
│          │                      │          ││                   │
│          │                      │ loop     ││                   │
│          │ chorus count done    └──────────┘│                   │
│          ▼                                  │                   │
│   _playing_trailer                          │                   │
│          │                                  │                   │
│          │ trailer complete                 │                   │
│          ▼                                  │                   │
│   _song_completed flag ─────────────────────┘                   │
│          │                     (if _song_automatically_loops)   │
│          │                                                      │
│          ▼ (if no loop)                                         │
│   _no_song_playing                                              │
│                                                                 │
│   ┌────────────────────────────────────────────────────────┐    │
│   │ fade_out_music() can transition to:                    │    │
│   │   _music_fading ──► gradual volume decrease            │    │
│   │                 ──► _no_song_playing when done         │    │
│   └────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 29.3 Music Data Structure

```c
struct music_data {
    boolean initialized;           // Handler ready
    short flags;                   // _song_completed, _song_paused
    short state;                   // Current playback state
    short phase;                   // Timing counter
    short fade_duration;           // Total fade time
    short play_count;              // Chorus repetitions remaining
    short song_index;              // Current song
    short next_song_index;         // Queued song (or NONE)
    short song_file_refnum;        // File handle to Music file
    short fade_interval_duration;  // Ticks between volume steps
    short fade_interval_ticks;     // Counter for volume steps
    long ticks_at_last_update;     // For delta timing
    char *sound_buffer;            // Playback buffer
    long sound_buffer_size;        // Buffer size
    SndChannelPtr channel;         // Audio channel (Mac-specific)
    FilePlayCompletionUPP completion_proc;  // Callback
};
```

---

## 29.4 Song Definition

```c
struct song_definition {
    short flags;                     // _song_automatically_loops
    long sound_start;                // File offset to song data
    struct sound_snippet introduction;  // Intro section
    struct sound_snippet chorus;     // Main loop section
    short chorus_count;              // Times to play (-1 = random)
    struct sound_snippet trailer;    // Outro section
    long restart_delay;              // Ticks before looping
};

struct sound_snippet {
    long start_offset;   // Byte offset in file
    long end_offset;     // End byte offset
};
```

### Song Structure Visualization

```
Song File Layout:
┌─────────────────────────────────────────────────────────────────┐
│ │◄── Introduction ──►│◄───── Chorus ─────►│◄── Trailer ──► │    │
│ ├────────────────────┼────────────────────┼────────────────┤    │
│ │    intro.start     │   chorus.start     │  trailer.start │    │
│ │         ↓          │        ↓           │       ↓        │    │
│ │    intro.end       │   chorus.end       │  trailer.end   │    │
└─────────────────────────────────────────────────────────────────┘

Playback Order:
  Introduction → Chorus (×N) → Trailer → [restart_delay] → Loop
                    ▲                                        │
                    └────────────────────────────────────────┘
                         (if _song_automatically_loops)
```

---

## 29.5 Music States

```c
enum /* music states */ {
    _no_song_playing,
    _playing_introduction,
    _playing_chorus,
    _playing_trailer,
    _delaying_for_loop,
    _music_fading
};

enum /* music flags */ {
    _no_flags         = 0x0000,
    _song_completed   = 0x0001,
    _song_paused      = 0x0002
};

enum /* song flags */ {
    _no_song_flags          = 0x0000,
    _song_automatically_loops = 0x0001
};
```

---

## 29.6 Music API

```c
// Initialize music system with song file
boolean initialize_music_handler(FileDesc *song_file);

// Queue a song to play (fades out current if playing)
void queue_song(short song_index);

// Fade out over duration (in ticks)
void fade_out_music(short duration);

// Stop immediately
void stop_music(void);

// Pause/resume
void pause_music(boolean pause);

// Check if music is playing
boolean music_playing(void);

// Called each tick to update state machine
void music_idle_proc(void);

// Release audio channel (for other sound needs)
void free_music_channel(void);
```

---

## 29.7 Fade System

```c
#define BUILD_STEREO_VOLUME(l, r) ((((long)(r))<<16)|(l))

void fade_out_music(short duration) {
    music_state->fade_duration = duration;
    music_state->phase = duration;
    music_state->state = _music_fading;
    music_state->fade_interval_duration = 5;  // Steps every 5 ticks
    music_state->fade_interval_ticks = 5;
    music_state->song_index = NONE;
}

// In music_idle_proc(), _music_fading state:
// new_volume = (0x100 * phase) / fade_duration
// Ranges from 256 (full) to 0 (silent)
```

### Fade Visualization

```
Volume
  256 ┤▓▓▓▓▓▓▓▓▓▓▓
      │          ▓▓▓▓
      │              ▓▓▓▓
  128 ┤                  ▓▓▓▓
      │                      ▓▓▓▓
      │                          ▓▓▓▓
    0 ┼───────────────────────────────────►
      0    fade_duration/2     fade_duration
                 Time (ticks)
```

---

## 29.8 Music Playback Flow

```
queue_song(song_index):
    │
    ├─► Is music currently playing?
    │     │
    │     ├─► Yes: Start fade, set next_song_index
    │     │
    │     └─► No: Start immediately
    │
    ▼
music_idle_proc() (called each tick):
    │
    ├─► _no_song_playing:
    │     └─► Check for queued song, start if present
    │
    ├─► _playing_introduction:
    │     └─► When done, switch to _playing_chorus
    │
    ├─► _playing_chorus:
    │     └─► Decrement play_count, loop or advance
    │
    ├─► _playing_trailer:
    │     └─► When done, check for auto-loop
    │
    ├─► _delaying_for_loop:
    │     └─► Count down restart_delay
    │
    └─► _music_fading:
          └─► Reduce volume, stop when silent
```

---

## 29.9 Buffer Management

```c
#define kDefaultSoundBufferSize (500*KILO)  // 500KB buffer

// Music streams from disk into this buffer
// Double-buffering allows continuous playback
// while loading next chunk
```

---

## 29.10 music_idle_proc() Implementation

The core music update function runs every tick and manages all state transitions (music.c:195):

```c
void music_idle_proc(void)
{
    if (music_state && music_state->initialized &&
        music_state->state != _no_song_playing)
    {
        short ticks_elapsed = TickCount() - music_state->ticks_at_last_update;

        switch (music_state->state)
        {
            case _delaying_for_loop:
                // Count down restart delay before starting playback
                if ((music_state->phase -= ticks_elapsed) <= 0)
                {
                    // Allocate streaming buffer and start playback
                    music_state->sound_buffer = malloc(kDefaultSoundBufferSize);
                    if (music_state->sound_buffer)
                    {
                        // Start async file playback with completion callback
                        error = SndStartFilePlay(
                            music_state->channel,
                            music_state->song_file_refnum,
                            0,  // resource ID
                            music_state->sound_buffer_size,
                            music_state->sound_buffer,
                            NULL,  // audio selection
                            music_state->completion_proc,  // callback
                            TRUE);  // async

                        if (!error)
                            music_state->state = _playing_introduction;
                        else
                            music_state->state = _no_song_playing;
                    }
                }
                break;

            case _music_fading:
                if (ticks_elapsed > 0)
                {
                    // Check if fade complete or song ended
                    if ((music_state->phase -= ticks_elapsed) <= 0 ||
                        (music_state->flags & _song_completed))
                    {
                        stop_music();
                        music_state->state = _no_song_playing;

                        // Start queued song if any
                        if (music_state->song_index != NONE)
                            queue_song(music_state->song_index);
                    }
                    else
                    {
                        // Gradual volume reduction
                        if (--music_state->fade_interval_ticks <= 0)
                        {
                            music_state->fade_interval_ticks =
                                music_state->fade_interval_duration;

                            // Linear interpolation: volume = 256 * phase / duration
                            short new_volume =
                                (0x100 * music_state->phase) / music_state->fade_duration;

                            // Apply stereo volume via Sound Manager
                            SndCommand command;
                            command.cmd = volumeCmd;
                            command.param1 = 0;
                            command.param2 = BUILD_STEREO_VOLUME(new_volume, new_volume);
                            SndDoImmediate(music_state->channel, &command);
                        }
                    }
                }
                break;

            default:  // _playing_introduction, _playing_chorus, _playing_trailer
                // Wait for completion callback to set _song_completed flag
                if (music_state->flags & _song_completed)
                {
                    struct song_definition *song =
                        get_song_definition(music_state->song_index);

                    if (song->flags & _song_automatically_loops)
                    {
                        // Restart with delay
                        music_state->state = _delaying_for_loop;
                        music_state->phase = song->restart_delay;
                    }
                    else
                    {
                        music_state->state = _no_song_playing;
                    }
                    music_state->flags &= ~_song_completed;
                }
                break;
        }
        music_state->ticks_at_last_update = TickCount();
    }
}
```

### Completion Callback

```c
// Called by Sound Manager when section finishes (music.c:~350)
static pascal void file_play_completion_routine(SndChannelPtr channel)
{
    struct music_data *state = (struct music_data *)channel->userInfo;
    state->flags |= _song_completed;
}
```

### Timing Constants

```c
#define MACINTOSH_TICKS_PER_SECOND 60
#define kDefaultSoundBufferSize (500*KILO)  // 500KB streaming buffer

// Typical fade duration: 10 seconds
// fade_out_music(10 * MACINTOSH_TICKS_PER_SECOND);  // 600 ticks

// Fade interval: update volume every 5 ticks (~12 updates/second)
#define DEFAULT_FADE_INTERVAL 5
```

---

## 29.11 See Also

- **[Chapter 13: Sound System](13_sound.md)** - Sound channel management and 3D audio
- **[Chapter 28: Computer Terminals](28_terminals.md)** - `#TRACK` command triggers music
- **[Chapter 7: Game Loop](07_game_loop.md)** - `music_idle_proc()` called from main loop

---

## 29.12 Summary

Marathon's music system provides:

- **Structured songs** (intro/chorus/outro)
- **Automatic looping** with delay between loops
- **Smooth fading** for transitions
- **Queue system** for seamless track changes
- **Streaming playback** from disk

### Key Source Files

| File | Purpose |
|------|---------|
| `music.c` | Music playback logic |
| `music.h` | Music structures |
| `song_definitions.h` | Song data |

---

*Next: [Chapter 30: Error Handling & Progress](30_errors.md) - Error states and loading displays*
