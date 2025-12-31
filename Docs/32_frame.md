# Chapter 32: Life of a Frame

## Complete Frame Lifecycle Walkthrough

> **For Porting:** This chapter traces execution from input to pixels. Understanding this flow is essential for integrating Marathon into any windowing system.
>
> **Terminology:** `world_pixels`, `screen`, `destination`, and `framebuffer` all refer to the render target buffer. See **[Appendix A: Glossary → Graphics Buffer Terminology](appendix_a_glossary.md#graphics-buffer-terminology)** for details.

---

## 32.1 Overview

Every frame in Marathon follows this path:

```
┌─────────────────────────────────────────────────────────────────┐
│                    MARATHON FRAME LIFECYCLE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌──────────────────────┐                                       │
│   │  1. INPUT GATHERING  │  Read keyboard/mouse, build action    │
│   │     (at interrupt)   │  flags, queue for game loop           │
│   └──────────┬───────────┘                                       │
│              │                                                   │
│              ▼                                                   │
│   ┌──────────────────────┐                                       │
│   │  2. WORLD UPDATE     │  Physics, monsters, projectiles       │
│   │     update_world()   │  (may run 0-N times per frame)        │
│   └──────────┬───────────┘                                       │
│              │                                                   │
│              ▼                                                   │
│   ┌──────────────────────┐                                       │
│   │  3. RENDER SCREEN    │  Build view, render 3D world,         │
│   │     render_screen()  │  draw HUD, apply fades                │
│   └──────────┬───────────┘                                       │
│              │                                                   │
│              ▼                                                   │
│   ┌──────────────────────┐                                       │
│   │  4. DISPLAY          │  Copy framebuffer to screen           │
│   │     (blit/flip)      │                                       │
│   └──────────────────────┘                                       │
│                                                                  │
│   Target: 30 game ticks/second, unlimited render FPS             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 32.2 Phase 1: Input Gathering

Input is gathered via interrupt handler (VBL on Mac):

### Source: `vbl.c` / keyboard controller

```c
// Called at interrupt time (~60Hz on Mac)
void input_controller_interrupt(void) {
    // Read raw input state
    GetKeys(key_map);
    GetMouse(&mouse_position);

    // Convert to action flags
    long action_flags = 0;
    if (key_map[key_forward])  action_flags |= _moving_forward;
    if (key_map[key_back])     action_flags |= _moving_backward;
    if (key_map[key_left])     action_flags |= _turning_left;
    if (key_map[key_right])    action_flags |= _turning_right;
    if (key_map[key_fire])     action_flags |= _left_trigger_state;
    // ... more flags

    // Include mouse delta for turning
    action_flags |= (mouse_delta_x << _turn_delta_shift);
    action_flags |= (mouse_delta_y << _pitch_delta_shift);

    // Queue for game loop
    queue_action_flags(local_player_index, action_flags);

    heartbeat_count++;  // Track time for game loop
}
```

### Action Flags

```c
enum /* action_flags */ {
    _absolute_yaw_mode           = 0x0001,
    _turning_left                = 0x0002,
    _turning_right               = 0x0004,
    _sidestep_dont_turn          = 0x0008,
    _looking_left                = 0x0010,
    _looking_right               = 0x0020,
    _moving_forward              = 0x0040,
    _moving_backward             = 0x0080,
    _sidestepping_left           = 0x0100,
    _sidestepping_right          = 0x0200,
    _looking_up                  = 0x0400,
    _looking_down                = 0x0800,
    _action_trigger_state        = 0x1000,  // Use/activate
    _left_trigger_state          = 0x2000,  // Primary fire
    _right_trigger_state         = 0x4000,  // Secondary fire
    _toggle_map                  = 0x8000,
    // ... more flags
};
```

---

## 32.3 Phase 2: World Update

### Main Game Loop (interface.c:536-545)

```c
// In game's idle processing
if (game_state.state == _game_in_progress) {
    if (get_keyboard_controller_status() && (ticks_elapsed = update_world())) {
        render_screen(ticks_elapsed);
    }
    else if (no_frame_rate_limit) {
        render_screen(0);  // Render even without game tick
    }
}
```

### update_world() (marathon2.c:73-149)

This is the core game tick. It may execute **0 or more times per frame** depending on queued input:

```c
short update_world(void) {
    short lowest_time, highest_time;
    short time_elapsed;

    // Find how many ticks we can advance (limited by slowest player in network)
    highest_time = SHORT_MIN, lowest_time = SHORT_MAX;
    for (player_index = 0; player_index < dynamic_world->player_count; ++player_index) {
        short queue_size;

        if (game_is_networked) {
            queue_size = MIN(get_action_queue_size(player_index),
                            NetGetNetTime() - dynamic_world->tick_count);
        } else {
            queue_size = MIN(get_action_queue_size(player_index),
                            get_heartbeat_count() - dynamic_world->tick_count);
        }

        if (queue_size > highest_time) highest_time = queue_size;
        if (queue_size < lowest_time) lowest_time = queue_size;
    }

    time_elapsed = lowest_time;

    // Execute N game ticks
    for (i = 0; i < time_elapsed; ++i) {
        // === THE UPDATE ORDER (CRITICAL!) ===
        update_lights();           // 1. Animate lights
        update_medias();           // 2. Update liquid heights
        update_platforms();        // 3. Move platforms
        update_control_panels();   // 4. Check switches (before players!)
        update_players();          // 5. Process player input & physics
        move_projectiles();        // 6. Move bullets/rockets
        move_monsters();           // 7. AI and monster movement
        update_effects();          // 8. Animate visual effects
        recreate_objects();        // 9. Rebuild object lists
        handle_random_sound_image(); // 10. Ambient sounds
        animate_scenery();         // 11. Animated decorations
        update_net_game();         // 12. Network state sync

        // Check for level transition
        if (check_level_change()) {
            time_elapsed = 0;
            break;
        }
        if (game_is_over()) break;

        dynamic_world->tick_count += 1;
        dynamic_world->game_information.game_time_remaining -= 1;
    }

    // Post-tick updates
    if (time_elapsed) {
        update_interface(time_elapsed);  // HUD dirty flags
        update_fades();                   // Screen effects
    }

    check_recording_replaying();

    return time_elapsed;
}
```

### Update Order Visualization

```
┌─────────────────────────────────────────────────────────────────┐
│                    UPDATE ORDER (PER TICK)                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   1. update_lights()                                             │
│      └─► Animate light sources (flicker, pulse, fade)           │
│          This affects media heights too!                         │
│                                                                  │
│   2. update_medias()                                             │
│      └─► Calculate new liquid heights from light intensities    │
│          Update current/flow texture scrolling                  │
│                                                                  │
│   3. update_platforms()                                          │
│      └─► Move elevators, doors, crushing ceilings               │
│          Damage monsters/players caught in crushers             │
│                                                                  │
│   4. update_control_panels()  ← BEFORE update_players!          │
│      └─► Process switch states                                  │
│          Must run before players to avoid race conditions       │
│                                                                  │
│   5. update_players()                                            │
│      └─► For each player:                                       │
│            • Dequeue action flags                               │
│            • Apply movement physics                             │
│            • Handle weapon firing                               │
│            • Process item pickups                               │
│            • Update view bobbing                                │
│                                                                  │
│   6. move_projectiles()                                          │
│      └─► For each projectile:                                   │
│            • Apply velocity                                     │
│            • Check collisions                                   │
│            • Spawn effects on impact                            │
│            • Deal damage on hit                                 │
│                                                                  │
│   7. move_monsters()                                             │
│      └─► For each monster:                                      │
│            • Run AI state machine                               │
│            • Calculate pathfinding                              │
│            • Apply movement                                     │
│            • Attack if in range                                 │
│                                                                  │
│   8. update_effects()                                            │
│      └─► Animate explosions, blood, sparks                      │
│          Remove completed effects                               │
│                                                                  │
│   9. recreate_objects()                                          │
│      └─► Rebuild sorted object lists for rendering              │
│                                                                  │
│  10. handle_random_sound_image()                                 │
│      └─► Play ambient sounds (wind, water, etc.)                │
│                                                                  │
│  11. animate_scenery()                                           │
│      └─► Update animated decorations (max 20)                   │
│                                                                  │
│  12. update_net_game()                                           │
│      └─► Sync state for multiplayer                             │
│          Check win/lose conditions                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 32.4 Phase 3: Render Screen

### render_screen() (screen.c:668-810)

```c
void render_screen(short ticks_elapsed) {
    // 1. Setup view parameters from current player
    world_view->ticks_elapsed = ticks_elapsed;
    world_view->tick_count = dynamic_world->tick_count;
    world_view->yaw = current_player->facing;
    world_view->pitch = current_player->elevation;
    world_view->maximum_depth_intensity = current_player->weapon_intensity;

    // 2. Handle special vision modes
    world_view->shading_mode = current_player->infravision_duration ?
        _shading_infravision : _shading_normal;

    // 3. Handle extravision (fisheye) effect
    if (current_player->extravision_duration) {
        if (world_view->field_of_view != EXTRAVISION_FIELD_OF_VIEW) {
            world_view->field_of_view = EXTRAVISION_FIELD_OF_VIEW;
            initialize_view_data(world_view);
        }
    }

    // 4. Handle overhead map
    if (PLAYER_HAS_MAP_OPEN(current_player)) {
        if (!world_view->overhead_map_active) set_overhead_map_status(TRUE);
    }

    // 5. Handle terminal mode
    if (player_in_terminal_mode(current_player_index)) {
        if (!world_view->terminal_mode_active) set_terminal_status(TRUE);
    }

    // 6. Setup framebuffer
    world_pixels_structure->width = world_view->screen_width;
    world_pixels_structure->height = world_view->screen_height;
    world_pixels_structure->bytes_per_row = (*pixels)->rowBytes & 0x3fff;
    world_pixels_structure->row_addresses[0] = myGetPixBaseAddr(world_pixels);
    precalculate_bitmap_row_addresses(world_pixels_structure);

    // 7. Set camera position
    world_view->origin = current_player->camera_location;
    world_view->origin_polygon_index = current_player->camera_polygon_index;

    // 8. RENDER THE 3D VIEW!
    render_view(world_view, world_pixels_structure);

    // 9. Blit to screen (Mac-specific)
    draw_world_pixels_to_screen();
}
```

### render_view() (render.c:497-547)

```c
void render_view(struct view_data *view, struct bitmap_definition *destination) {
    // 1. Update view transformation matrices
    update_view_data(view);

    // 2. Clear render flags
    memset(render_flags, 0, sizeof(word) * RENDER_FLAGS_BUFFER_SIZE);

    // 3. Choose render mode
    if (view->terminal_mode_active) {
        // Render computer terminal
        render_computer_interface(view);
    }
    else {
        // BUILD THE RENDER TREE
        // This is the portal visibility algorithm!
        build_render_tree(view);

        if (view->overhead_map_active) {
            // Render 2D overhead map
            render_overhead_map(view);
        }
        else {
            // === MAIN 3D RENDERING PATH ===

            // Sort by depth for correct draw order
            sort_render_tree(view);

            // Collect visible objects (monsters, items, players)
            build_render_object_list(view);

            // Draw polygons back-to-front with texture mapping
            render_tree(view, destination);

            // Draw player's weapon sprite on top
            render_viewer_sprite_layer(view, destination);
        }
    }
}
```

---

## 32.5 The Render Tree

The render tree is Marathon's visibility system:

```
┌─────────────────────────────────────────────────────────────────┐
│                    RENDER TREE BUILDING                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   build_render_tree(view):                                       │
│                                                                  │
│   1. Start at viewer's polygon                                   │
│      ┌─────────────────────────────────────────┐                │
│      │  Polygon Queue: [viewer_polygon]        │                │
│      └─────────────────────────────────────────┘                │
│                                                                  │
│   2. For each polygon in queue:                                  │
│      • Check each line (edge)                                   │
│      • If line is transparent/portal AND visible in view cone:  │
│        - Add neighboring polygon to queue                       │
│        - Create render node with clipping info                  │
│                                                                  │
│   3. Result: Tree of visible polygons with clip windows         │
│                                                                  │
│   Example scene:                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                                                          │   │
│   │   ┌───────┐           ┌───────┐         ┌───────┐       │   │
│   │   │ Poly5 │           │ Poly3 │         │ Poly4 │       │   │
│   │   │ (far) │───portal──│       │─portal──│       │       │   │
│   │   └───────┘           └───────┘         └───────┘       │   │
│   │                            │                             │   │
│   │                         portal                           │   │
│   │                            │                             │   │
│   │                       ┌───────┐                          │   │
│   │                       │ Poly1 │                          │   │
│   │                       │(start)│ ← Viewer here            │   │
│   │                       └───────┘                          │   │
│   │                                                          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   Resulting render tree:                                         │
│                                                                  │
│                         Poly1                                    │
│                        /     \                                   │
│                    Poly3     Poly4                               │
│                      │                                           │
│                    Poly5                                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 32.6 Texture Mapping Pipeline

For each visible surface:

```
┌─────────────────────────────────────────────────────────────────┐
│                    TEXTURE MAPPING PIPELINE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   1. CLIP TO VIEW FRUSTUM                                        │
│      • Left/right cone edges                                    │
│      • Top/bottom scan limits                                   │
│      • Near plane                                               │
│                                                                  │
│   2. PROJECT TO SCREEN                                           │
│      screen_x = (world_x * world_to_screen_x) / world_z         │
│      screen_y = (world_y * world_to_screen_y) / world_z + dtanpitch │
│                                                                  │
│   3. CHOOSE TEXTURE MAPPER                                       │
│      • Walls: texture_horizontal_polygon (perspective-correct)  │
│      • Floors/Ceilings: texture_vertical_polygon (affine)       │
│                                                                  │
│   4. FOR EACH COLUMN (walls) OR ROW (floors):                    │
│      │                                                          │
│      ├─► Calculate texture coordinates                          │
│      │                                                          │
│      ├─► Calculate lighting (from light index → intensity)      │
│      │                                                          │
│      ├─► Look up shading table for this light level             │
│      │                                                          │
│      └─► Write pixels: dest[x] = shading_table[texture[u,v]]    │
│                                                                  │
│   5. APPLY TRANSFER MODE                                         │
│      • _normal: Direct texture lookup                           │
│      • _tint: Multiply by tint color                            │
│      • _static: Random noise                                    │
│      • _landscape: Spherical mapping                            │
│      • _xfer_fold_in/out: Teleport effect                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 32.7 Phase 4: Display

### Blitting to Screen (Mac-specific)

```c
// screen.c - After render_view completes:
void draw_world_pixels_to_screen(void) {
    // Copy offscreen buffer to visible screen
    GDHandle old_device;
    CGrafPtr old_port;

    myGetGWorld(&old_port, &old_device);
    mySetGWorld((CGrafPtr)screen_window, world_device);

    // CopyBits from offscreen to screen
    myHLock((Handle)world_pixels->portPixMap);
    myHLock((Handle)(*world_device)->gdPMap);

    CopyBits(&world_pixels->portBits,
             &screen_window->portBits,
             &world_pixels->portRect,
             &screen_window->portRect,
             srcCopy, NULL);

    myHUnlock((Handle)world_pixels->portPixMap);
    myHUnlock((Handle)(*world_device)->gdPMap);

    mySetGWorld(old_port, old_device);
}
```

### For Modern Porting

Replace the Mac-specific blit with your windowing system:

```c
// Example with Fenster/Full Beans:
void display_frame(struct fenster *f, uint32_t *framebuffer) {
    // Marathon renders to framebuffer
    // Just point Fenster's buffer at it
    memcpy(f->buf, framebuffer, width * height * sizeof(uint32_t));
    // Or render directly into f->buf
}
```

---

## 32.8 Timing Diagram

```
Time ──────────────────────────────────────────────────────────────►
      │         │         │         │         │         │
      ▼         ▼         ▼         ▼         ▼         ▼
   VBL Int   VBL Int   VBL Int   VBL Int   VBL Int   VBL Int
   (~60Hz)

   Queue     Queue     Queue     Queue     Queue     Queue
   Input     Input     Input     Input     Input     Input

      └────┬────┘         └────┬────┘         └────┬────┘
           │                   │                   │
           ▼                   ▼                   ▼
      Game Tick           Game Tick           Game Tick
      (30Hz target)       (30Hz target)       (30Hz target)

           │                   │                   │
           ▼                   ▼                   ▼
      Render Frame        Render Frame        Render Frame
      (unlimited)         (unlimited)         (unlimited)

Key insight:
• Input gathered at interrupt rate (~60Hz)
• Game logic runs at 30 ticks/second
• Rendering can happen faster than game ticks
• Multiple inputs may accumulate per game tick
```

---

## 32.9 Frame Timing Code

```c
// From interface.c - Main loop excerpt:
void idle_game_state(void) {
    // Track time since last idle
    short ticks_elapsed;
    long machine_tick_count = TickCount();  // Mac OS timing

    // Has enough time passed?
    if (machine_tick_count - game_state.last_ticks_on_idle >=
        game_state.user_pause_count) {

        // Update and render
        if (game_state.state == _game_in_progress) {
            if (get_keyboard_controller_status()) {
                ticks_elapsed = update_world();  // May be 0!

                if (ticks_elapsed || no_frame_rate_limit) {
                    render_screen(ticks_elapsed);
                }
            }
        }

        game_state.last_ticks_on_idle = machine_tick_count;
    }
}
```

---

## 32.10 Complete Call Graph

```
main()
  └─► main_event_loop()
        └─► idle_game_state()
              │
              ├─► update_world()                    [GAME LOGIC]
              │     ├─► update_lights()
              │     ├─► update_medias()
              │     ├─► update_platforms()
              │     ├─► update_control_panels()
              │     ├─► update_players()
              │     │     ├─► process_action_flags()
              │     │     ├─► update_player_physics()
              │     │     └─► update_player_weapons()
              │     ├─► move_projectiles()
              │     ├─► move_monsters()
              │     │     ├─► execute_monster_ai()
              │     │     └─► move_monster()
              │     ├─► update_effects()
              │     ├─► recreate_objects()
              │     ├─► handle_random_sound_image()
              │     ├─► animate_scenery()
              │     ├─► update_net_game()
              │     ├─► update_interface()
              │     └─► update_fades()
              │
              └─► render_screen()                   [RENDERING]
                    ├─► setup_view_data()
                    └─► render_view()
                          ├─► update_view_data()
                          ├─► build_render_tree()
                          ├─► sort_render_tree()
                          ├─► build_render_object_list()
                          ├─► render_tree()
                          │     ├─► render_node()
                          │     │     ├─► texture_horizontal_polygon()
                          │     │     └─► texture_vertical_polygon()
                          │     └─► render_object()
                          │           └─► texture_rectangle()
                          └─► render_viewer_sprite_layer()
                                └─► draw_weapon()
```

---

## 32.11 Key Variables

| Variable | Location | Purpose |
|----------|----------|---------|
| `dynamic_world->tick_count` | map.c | Global game tick counter |
| `heartbeat_count` | vbl.c | Input interrupt counter |
| `current_player` | player.c | Pointer to local player |
| `world_view` | screen.c | Current view parameters |
| `render_flags` | render.c | Per-polygon render state |

---

## 32.12 Timing Budget

Typical frame time breakdown on target hardware (1995 Macintosh):

| Phase | Typical Time | Percentage | Notes |
|-------|--------------|------------|-------|
| Input | 0.5-1 ms | ~3% | VBL-driven, very fast |
| Update (per tick) | 5-15 ms | ~30% | Monster AI is expensive |
| Render | 15-25 ms | ~60% | Texture mapping dominates |
| Display/Blit | 1-3 ms | ~7% | Memory copy to screen |
| **Total** | **22-44 ms** | **100%** | **23-45 FPS** |

On modern hardware (2020+):

| Phase | Typical Time | Notes |
|-------|--------------|-------|
| Input | < 0.1 ms | Trivial |
| Update | 0.5-2 ms | Even complex AI is fast |
| Render | 1-5 ms | Software rendering still works |
| **Total** | **2-8 ms** | **120-500 FPS** (capped to 30 ticks) |

---

## 32.13 Phase Comparison

| Aspect | Input | Update | Render |
|--------|-------|--------|--------|
| **Rate** | 60 Hz (VBL) | 30 Hz (fixed) | Variable |
| **Trigger** | Interrupt | Main loop | After update |
| **Duration** | ~1 ms | 5-15 ms | 15-25 ms |
| **State** | Reads hardware | Modifies world | Reads world |
| **Deterministic** | Yes | Yes | N/A |
| **Network sync** | Queued | Critical | Not synced |

### Key Functions by Phase

| Phase | Primary Function | Location | Purpose |
|-------|-----------------|----------|---------|
| Input | `parse_keymap()` | vbl_macintosh.c | Read keyboard/mouse |
| Input | `process_action_flags()` | vbl.c | Queue for player |
| Update | `update_world()` | marathon2.c:73 | Advance simulation |
| Update | `update_players()` | player.c | Apply player input |
| Update | `move_monsters()` | monsters.c | AI and pathfinding |
| Render | `render_screen()` | screen.c | Setup and dispatch |
| Render | `render_view()` | render.c:497 | 3D world rendering |
| Render | `build_render_tree()` | render.c:702 | Portal visibility |

---

## 32.14 Summary

Marathon's frame lifecycle:

1. **Input** gathered at interrupt time, queued as action flags
2. **Update** runs 0-N times depending on available input (30 ticks/sec target)
3. **Render** builds visibility tree, texture maps surfaces, draws HUD
4. **Display** copies framebuffer to screen

### Critical Order Dependencies

- `update_control_panels()` MUST run before `update_players()`
- `recreate_objects()` MUST run after all movement updates
- `update_interface()` and `update_fades()` run after all ticks complete
- Rendering happens AFTER world state is finalized

### Key Source Files

| File | Purpose |
|------|---------|
| `marathon2.c` | update_world() - game tick |
| `interface.c` | Main game loop |
| `render.c` | 3D rendering |
| `screen.c` | render_screen(), display |
| `vbl.c` | Input interrupt handler |

---

*Next: [Appendix A: Glossary](appendix_a_glossary.md) - Definitions of key terms*
