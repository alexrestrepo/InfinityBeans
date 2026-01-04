# Appendix N: Network Protocol Specification

## Byte-Level Packet Formats for Marathon Multiplayer

> **Source files**: `network.h`, `network.c`, `network_ring.c`, `network_modem.c`, `network_ddp.c`
> **Related chapters**: [Chapter 9: Networking Architecture](09_networking.md)

This appendix documents the byte-level format of Marathon's network packets, enabling implementation of compatible multiplayer clients.

---

## N.1 Protocol Overview

Marathon uses a **ring-based peer-to-peer protocol** where action flags circulate between players. Each player maintains a local action queue and broadcasts their inputs to other players.

**Key characteristics:**
- Deterministic simulation (same inputs = same results)
- Ring topology for packet distribution
- ACK-based reliability with retransmission
- 30 ticks/second synchronization

---

## N.2 Packet Header Structure

All network packets begin with a common header (from `network.h:112-118`):

### NetPacketHeader (8 bytes)

```
Offset  Size  Field           Description
──────────────────────────────────────────────────────────
0x00    2     tag             Packet type identifier
0x02    2     sequence        Sequence number for ordering
0x04    4     reserved        Padding/future use
```

**C Structure:**
```c
struct NetPacketHeader {
    short tag;              // Packet type (see tag values below)
    short sequence;         // Monotonically increasing sequence number
    long reserved;          // Unused, set to 0
};
```

### Packet Tag Values

| Tag Value | Name | Description |
|-----------|------|-------------|
| 0x0001 | `tagRING_PACKET` | Normal game data packet |
| 0x0002 | `tagACKNOWLEDGEMENT` | Acknowledgement of received packet |
| 0x0003 | `tagCHANGE_RING_PACKET` | Ring topology change |
| 0x0100 | `tagNEW_PLAYER` | Player joining game |
| 0x0101 | `tagCANCEL_GAME` | Game cancelled |
| 0x0102 | `tagSTART_GAME` | Game starting |
| 0x0103 | `tagDROP_PLAYER` | Player dropped from game |
| 0x0104 | `tagTOPOLOGY` | Topology distribution |

---

## N.3 Ring Packet Format

The primary game data packet carrying action flags between players.

### NetPacket Structure (from `network.h:120-135`)

```
Offset  Size  Field                   Description
──────────────────────────────────────────────────────────
0x00    8     header                  NetPacketHeader
0x08    2     ring_packet_type        Type of ring packet
0x0A    2     server_player_index     Which player is server
0x0C    4     server_net_time         Server's game time
0x10    N×2   action_flag_count[N]    Count of flags per player
0x10+N×2 M×4  action_flags[M]         Actual action flag data
```

**C Structure:**
```c
struct NetPacket {
    struct NetPacketHeader header;

    short ring_packet_type;         // _normal_ring_packet, _time_ring_packet, etc.
    short server_player_index;      // Index of current server
    long server_net_time;           // Server's tick count

    // Variable-length arrays follow:
    // short action_flag_count[MAXIMUM_NUMBER_OF_NETWORK_PLAYERS];
    // long action_flags[total_flags];  // Concatenated flags from all players
};

// Ring packet types
enum {
    _normal_ring_packet,            // 0: Normal game data
    _time_ring_packet,              // 1: Time synchronization
    _acknowledgement_ring_packet,   // 2: ACK response
    _network_stats_ring_packet      // 3: Statistics/debugging
};
```

### Action Flag Count Array

For N players, `action_flag_count` contains N shorts indicating how many action flags are included for each player:

```
action_flag_count[0] = 3    // Player 0 has 3 ticks of input
action_flag_count[1] = 2    // Player 1 has 2 ticks of input
action_flag_count[2] = 3    // Player 2 has 3 ticks of input
```

### Action Flags Array Layout

Action flags are concatenated in player order:

```
action_flags[] memory layout for above example:
┌────────────────────────────────────────────────────────────────┐
│ P0.tick0 │ P0.tick1 │ P0.tick2 │ P1.tick0 │ P1.tick1 │ P2.tick0 │ P2.tick1 │ P2.tick2 │
└────────────────────────────────────────────────────────────────┘
     0          1          2          3          4          5          6          7
```

---

## N.4 Action Flag Bit Layout

Each action flag is a 32-bit value encoding all player inputs for one tick (from `player.h:20-53`):

### Action Flag Bits (32 bits total)

```
Bit     Name                        Description
──────────────────────────────────────────────────────────────
0       _absolute_yaw               Using absolute yaw mode
1       _turning_left               Turn left key pressed
2       _turning_right              Turn right key pressed
3       _sidestep_dont_turn         Sidestep mode active
4       _looking_left               Look left key pressed
5       _looking_right              Look right key pressed
6       _sidestepping_left          Strafe left key pressed
7       _sidestepping_right         Strafe right key pressed
8       _looking_up                 Look up key pressed
9       _looking_down               Look down key pressed
10      _looking_center             Center look key pressed
11      _moving_forward             Forward key pressed
12      _moving_backward            Backward key pressed
13      _run_dont_walk              Run modifier key pressed
14      _toggle_map                 Toggle automap key pressed
15      _microphone_button          Voice chat active
16      _swim                       Swim up key pressed
17      _action_trigger             Action/use key pressed
18      _cycle_weapons_forward      Next weapon key pressed
19      _cycle_weapons_backward     Previous weapon key pressed
20      _left_trigger_state         Primary fire held
21      _right_trigger_state        Secondary fire held
22-31   (unused)                    Reserved for future use
```

**C Definitions:**
```c
// From player.h:20-53
#define _absolute_yaw_mode           0x0001
#define _turning_left                0x0002
#define _turning_right               0x0004
#define _sidestep_dont_turn          0x0008
#define _looking_left                0x0010
#define _looking_right               0x0020
#define _sidestepping_left           0x0040
#define _sidestepping_right          0x0080
#define _looking_up                  0x0100
#define _looking_down                0x0200
#define _looking_center              0x0400
#define _moving_forward              0x0800
#define _moving_backward             0x1000
#define _run_dont_walk               0x2000
#define _toggle_map                  0x4000
#define _microphone_button           0x8000
#define _swim                        0x00010000
#define _action_trigger              0x00020000
#define _cycle_weapons_forward       0x00040000
#define _cycle_weapons_backward      0x00080000
#define _left_trigger_state          0x00100000
#define _right_trigger_state         0x00200000
```

### Delta Encoding for Analog Values

When using absolute yaw mode (`_absolute_yaw_mode` set), additional data encodes the delta values in the upper bits:

```c
// The action flag encodes deltas when _absolute_yaw_mode is set:
// Bits 22-26: yaw_delta (5 bits, signed)
// Bits 27-31: pitch_delta (5 bits, signed)

// Encode:
action_flags |= ((yaw_delta & 0x1F) << 22);
action_flags |= ((pitch_delta & 0x1F) << 27);

// Decode:
yaw_delta = (action_flags >> 22) & 0x1F;
if (yaw_delta & 0x10) yaw_delta |= ~0x1F;  // Sign extend
pitch_delta = (action_flags >> 27) & 0x1F;
if (pitch_delta & 0x10) pitch_delta |= ~0x1F;  // Sign extend
```

---

## N.5 Game Setup Packets

### NetTopology Structure (432 bytes)

Distributed during game setup to establish player configuration (from `network.h:75-95`):

```
Offset  Size  Field                    Description
──────────────────────────────────────────────────────────
0x00    2     tag                      Always tagTOPOLOGY
0x02    2     player_count             Number of players (1-8)
0x04    2     nextIdentifier           Next available ID
0x06    2     game_data_size           Size of game_data block
0x08    32    game_data                Game configuration
0x28    N×48  players[N]               Player info array (8 max)
```

### NetPlayer Structure (48 bytes per player)

```
Offset  Size  Field                    Description
──────────────────────────────────────────────────────────
0x00    6     dspAddress               Network address (DDP)
0x06    2     ddpSocket                Socket number
0x08    2     identifier               Unique player ID
0x0A    2     flags                    Player flags (host, etc.)
0x0C    32    name                     Player name (Pascal string)
0x2C    2     team                     Team index
0x2E    2     color                    Player color index
```

### NetGameInfo Structure (32 bytes)

Game configuration data within topology:

```
Offset  Size  Field                    Description
──────────────────────────────────────────────────────────
0x00    2     game_type                Every Man for Himself, etc.
0x02    2     game_options             Flags (motion sensor, etc.)
0x04    2     kill_limit               Kill limit for game
0x06    2     time_limit               Time limit (ticks)
0x08    2     difficulty_level         0-4
0x0A    2     map_checksum             CRC of map file
0x0C    2     entry_point              Starting level index
0x0E    18    unused                   Reserved
```

**Game Type Values:**
```c
enum {
    _game_of_kill_monsters,      // 0: Cooperative
    _game_of_cooperative_play,   // 1: Cooperative (same?)
    _game_of_capture_the_flag,   // 2: CTF
    _game_of_king_of_the_hill,   // 3: KOTH
    _game_of_kill_man_with_ball, // 4: Kill the Man with the Ball
    _game_of_defense,            // 5: Defense
    _game_of_rugby,              // 6: Rugby
    _game_of_tag                 // 7: Tag
};
```

**Game Option Flags:**
```c
#define _motion_sensor_does_not_work    0x0001
#define _force_unique_teams             0x0002
#define _burn_items_on_death            0x0004
#define _live_network_stats             0x0008
#define _game_has_kill_limit            0x0010
#define _force_unique_colors            0x0020
#define _suicide_is_penalized           0x0040
#define _overhead_map_is_omniscient     0x0080
#define _overhead_map_shows_items       0x0100
#define _overhead_map_shows_monsters    0x0200
#define _overhead_map_shows_projectiles 0x0400
```

---

## N.6 Ring Protocol Operation

### Normal Ring Flow

```
┌────────────────────────────────────────────────────────────────┐
│                      RING PACKET FLOW                           │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│   Player 0 ──────► Player 1 ──────► Player 2 ──────┐           │
│      ▲                                              │           │
│      └──────────────────────────────────────────────┘           │
│                                                                │
│   Packet contains accumulated action flags from all            │
│   players visited so far in the ring.                          │
│                                                                │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│   Step 1: Player 0 creates packet with their flags             │
│           action_flag_count[0] = their_count                   │
│           Sends to Player 1                                    │
│                                                                │
│   Step 2: Player 1 receives, adds their flags                  │
│           action_flag_count[1] = their_count                   │
│           Appends their action_flags[]                         │
│           Sends to Player 2                                    │
│                                                                │
│   Step 3: Player 2 receives, adds their flags                  │
│           action_flag_count[2] = their_count                   │
│           Packet now complete                                  │
│           Sends back to Player 0                               │
│                                                                │
│   Step 4: Player 0 receives complete packet                    │
│           All players have all inputs                          │
│           Simulation can advance                               │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### ACK/Retry Protocol

```c
#define NET_RETRANSMIT_DELAY 15    // Ticks before retransmit
#define NET_MAXIMUM_RETRIES  10    // Max retries before disconnect

// On send:
send_packet(packet);
expected_ack = packet.sequence;
retransmit_timer = NET_RETRANSMIT_DELAY;

// On tick (if waiting for ACK):
if (--retransmit_timer == 0) {
    if (++retry_count > NET_MAXIMUM_RETRIES) {
        disconnect_player();
    } else {
        resend_packet(last_packet);
        retransmit_timer = NET_RETRANSMIT_DELAY;
    }
}

// On ACK received:
if (ack.sequence == expected_ack) {
    // Packet confirmed, can send next
    waiting_for_ack = FALSE;
}
```

### Acknowledgement Packet

```
Offset  Size  Field           Description
──────────────────────────────────────────────────────────
0x00    8     header          tag = tagACKNOWLEDGEMENT
                              sequence = sequence being ACKed
```

---

## N.7 Time Synchronization

### Time Ring Packet

Used to synchronize game clocks across players:

```c
struct TimeRingPacket {
    struct NetPacketHeader header;  // tag = tagRING_PACKET
    short ring_packet_type;         // _time_ring_packet
    short server_player_index;
    long server_net_time;           // Server's authoritative time
    // No action flags in time packets
};
```

**Synchronization Algorithm:**
```c
#define MAXIMUM_TIME_DIFFERENCE 15  // Maximum allowed drift (ticks)

void synchronize_time(long server_time) {
    long local_time = get_local_net_time();
    long drift = server_time - local_time;

    if (ABS(drift) > MAXIMUM_TIME_DIFFERENCE) {
        // Too far out of sync - resync
        set_local_net_time(server_time);
    } else if (drift > 0) {
        // Behind server - speed up
        advance_local_time();
    } else if (drift < 0) {
        // Ahead of server - slow down
        // Skip tick advancement
    }
}
```

---

## N.8 DDP (Datagram Delivery Protocol) Layer

Marathon uses AppleTalk's DDP for transport (replaced with UDP in modern ports).

### DDP Address Format (6 bytes)

```
Offset  Size  Field      Description
──────────────────────────────────────────────────────────
0x00    2     network    Network number (big-endian)
0x02    1     node       Node ID
0x03    1     unused     Padding
0x04    2     socket     Socket number (big-endian)
```

### DDP Packet Wrapper

```
Offset  Size  Field           Description
──────────────────────────────────────────────────────────
0x00    2     length          Packet length (including header)
0x02    2     destination     Destination socket
0x04    2     source          Source socket
0x06    1     type            DDP type (custom for Marathon)
0x07    N     data            NetPacketHeader + payload
```

---

## N.9 Modem Protocol Variations

For modem play, Marathon uses a slightly different protocol with streaming support.

### Modem Packet Types

```c
enum {
    _modem_data_packet,             // 0: Normal data
    _modem_acknowledgement_packet,  // 1: ACK
    _modem_stream_data_packet,      // 2: Streaming audio/data
    _modem_stream_ack_packet        // 3: Stream ACK
};
```

### Modem Statistics Structure

```c
struct modem_stats_data {
    long client_packets_sent;
    long server_packets_sent;
    long action_flags_processed;
    long numSmears;                 // Lag compensation events
    long stream_packets_necessary;
    long stream_packets_sent;
    long stream_early_acks;
    // ... additional fields
};
```

---

## N.10 Network Statistics

Marathon tracks various network statistics for debugging:

### NetStats Structure

```c
struct NetStats {
    long packets_sent;
    long packets_received;
    long packets_lost;
    long retransmissions;
    long acks_received;
    long acks_sent;
    long bytes_sent;
    long bytes_received;
    long round_trip_time;           // Average RTT in ticks
    long jitter;                    // RTT variance
};
```

---

## N.11 Implementation Notes

### Byte Order

All multi-byte values in Marathon network packets are **big-endian** (network byte order). On little-endian systems (x86):

```c
// Convert to network byte order before sending
packet.sequence = htons(local_sequence);
packet.server_net_time = htonl(local_time);

// Convert from network byte order after receiving
local_sequence = ntohs(packet.sequence);
local_time = ntohl(packet.server_net_time);
```

### Modern Replacement: UDP

For modern ports, replace DDP with UDP:

| DDP Concept | UDP Equivalent |
|-------------|----------------|
| DDP socket | UDP port |
| Node address | IP address |
| Network number | Subnet |
| `DDPWrite()` | `sendto()` |
| `DDPRead()` | `recvfrom()` |

### Determinism Requirements

Network play requires perfect determinism:

1. **Same inputs = Same outputs**: All clients must compute identical results
2. **No floating-point**: Use fixed-point math exclusively
3. **Fixed RNG state**: Random number generator seeded identically
4. **Synchronized timing**: All clients process same tick count

---

## N.12 See Also

- [Chapter 9: Networking Architecture](09_networking.md) — High-level networking overview
- [Chapter 7: Game Loop](07_game_loop.md) — Tick-based update model
- [Appendix D: Fixed-Point Math](appendix_d_fixedpoint.md) — Deterministic arithmetic

---

*Return to: [Table of Contents](README.md)*
