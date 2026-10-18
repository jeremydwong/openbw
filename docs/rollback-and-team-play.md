# Design: rollback netcode and team play for OpenBW

Status: draft for discussion · Author: Jeremy Wong · 2026-10-18

This document proposes two pieces of work on the `jeremydwong/openbw` fork:

1. **Rollback netcode** to replace the stall-on-late-input lockstep in `sync.h`, modelled on
   what ShieldBattery shipped for StarCraft: Remastered in October 2026.
2. **A team-play game mode** with three features:
   - a simple top-right panel to send minerals/gas to a teammate;
   - optional control of allied units;
   - taking over an ally's units when they leave.

The two pieces are separate but meet in one place: a player leaving is an irreversible event
that rollback must treat specially, and the takeover feature changes what happens at that
moment. Both are aimed at the fork (and Butterfly) first. Upstreaming to OpenBW is a later
question (see [Upstream](#upstream)).

---

## 1. Where we are today

### 1.1 OpenBW's lockstep (`sync.h`)

- **Transport.** `sync_server_asio_socket` sends uint16 length-prefixed messages over TCP or a
  local stream (`sync_server_asio_socket.h:194-252`), fully meshed (`sync_server_asio_tcp.h`).
  The transport interface is `sync_server_noop` (`sync.h:115-161`).
- **Scheduling.** Every message is scheduled at `client->frame + latency`, with `latency = 2`
  (`sync.h:223-281`). Frames are stored as `uint8_t` and wrap mod 256.
- **Stalling.** Each frame, `syncer_t::sync()` blocks in `server.run_until(all_clients_in_sync)`
  until every peer is within `latency` frames (`sync.h:949-996`). One late packet from anyone
  freezes everyone. That is the behaviour we want to get rid of.
- **Desync detection.** Every 32 frames, an FNV hash covers action counters, the RNG, resources
  and hp/shields/position of visible units (`update_insync_hash`, `sync.h:625-650`). A mismatch
  kills the client. Nothing is recovered.
- **Limits.**
  - At most 2 clients with a uid (`sync.h:392`, deliberate, commit 61955ca).
  - Player leave (action 87) is synthesized in `kill_client` and executed *immediately at local
    time*, not at a scheduled frame (`sync.h:463-484`). That is only safe with 2 peers.
  - Peers never agree on map, game type or alliances; only the seed and slots are exchanged.
- **Not used in this repo.** No frontend in this repo includes `sync.h`; `ui/` is a replay
  viewer. Whatever game client uses OpenBW netplay (Butterfly's WASM build) is the real
  consumer, and its transport (WebSocket/WebRTC) will not be asio TCP.

### 1.2 What we can reuse

- **The sim is deterministic and self-contained.** It uses only integer math, and the RNG
  (`lcg_rand_state`) lives in `state` (`bwgame.h:237-239`, `13197`). Data files and the map
  (`global_state`, `game_state`) are immutable during play.
- **Snapshots already work.**
  - `copy_state(const state&)` (`bwgame.h:19757`) deep-copies the state through `state_copier`,
    remapping every pointer and rebuilding every intrusive list.
  - `action_state` (selection, control groups) has its own `copy_state` (`actions.h:24-37`).
  - `ui/gfxtest.cpp:81-120` already restores a snapshot and re-simulates forward to seek
    backwards in replays. That is rollback-and-resimulate in all but name.
- **Actions are compact, player-tagged and replay-shaped.** `read_action(owner, r)`
  (`actions.h:1348`) is the single entry point, and IDs 211-219 and 221-255 are unused (220 is
  the sync escape byte).

### 1.3 What ShieldBattery did (verified)

ShieldBattery merged "Implement rollback netcode" (PR #1657) on 2026-10-06. It is built on
their QUIC relay transport, "netcode v2" (PR #1365). Their design doc is `docs/rollback.md` in
their repo. Whether it is enabled in production is not confirmed: it is gated behind
`SB_RP2_ROLLBACK`. The points that matter for us:

| Topic | ShieldBattery's choice |
|---|---|
| Prediction | A missing remote turn is simulated as an **empty turn**. The client rolls back if the real turn differs. It predicts at most 8 frames ahead, then stalls. |
| Delay vs rollback | Per-player "rollback balance" target of 0-4 (default 2). Steady lateness beyond it becomes that player's *own* input delay, so `D + R ≥ own RTT`. One player's bad connection never costs the others delay. |
| Snapshots | BW allocates nothing during play, so a snapshot is a memcpy of fixed memory ranges on helper threads, kept every 3 frames in a ring. |
| Cost | One step is about 0.25-0.45 ms. A 2-frame rollback costs under 1 ms; a frame at Fastest is 42 ms. |
| Side effects | Sounds, text and UI writes are deduplicated during re-simulation. Camera and local UI selection stay outside the snapshot. |
| Fences | A player leaving and the game ending happen only on *confirmed* frames, and no rollback crosses them. Replays record only confirmed input. |
| Desync | A pointer-free 64-bit state hash every 8 confirmed steps. Relays evict the minority; with no majority, the game is voided. |
| Visuals | Corrections show as snaps: invisible at 2-3 frames, noticeable from 4. Smoothing was tried and looked worse. |
| Testing | A replay-driven harness checks that rolled-back play matches straight playback frame for frame. There are also soak and benchmark runners. |

We should copy this design rather than invent one. The main difference for OpenBW is the
snapshot: our state is a C++ object graph, not fixed memory ranges (see §2.3).

---

## 2. Rollback netcode

### 2.1 Goals and non-goals

**Goals**
- No global stall when one peer's input arrives a few frames late.
- Each player's input delay depends only on their own connection.
- Support 2-8 players. This removes the 2-client cap and is required by the team mode.
- Detect desyncs reliably, and record replays from confirmed input only.

**Non-goals (for now)**
- Playing against real StarCraft clients. This is OpenBW-to-OpenBW only.
- Visual smoothing of corrections.
- Spectator/observer streams.

### 2.2 Model

Keep the input model of lockstep: each player sends a **turn** per frame, possibly empty, that
contains that player's actions for frame *f*. Then:

- **Confirmed frame `C`** is the highest frame for which every live player's turn is known.
- **Current frame `F`** is the frame the client is simulating. `F - C ≤ max_prediction`
  (default 8); beyond that, stall exactly as lockstep does today.
- **Prediction:** for each missing remote turn in `(C, F]`, assume it is empty.
- **Rollback:** when a remote turn arrives for frame `f ≤ F`, check whether it is empty.
  - If it is empty, the prediction was right: just advance `C`.
  - If it has actions, restore the snapshot at or before `f`, then re-simulate to `F` with the
    now-known turns.
- **Local input delay `D`:** local actions are stamped for frame `F + D`. `D` starts at 1-2 and
  is adapted per player from measured lateness, following ShieldBattery's rule that lateness
  beyond the rollback target becomes your own delay.

Turns must be sent every frame, even empty ones, so peers can advance `C`. An empty turn is a
few bytes; we can send several frames' turns per packet for redundancy over UDP-like
transports.

### 2.3 Snapshots: the hard part for OpenBW

ShieldBattery memcpys raw BW memory. OpenBW's `state` is made of allocator-backed containers
(`object_container`, `intrusive_list`, `a_list`, `std::unordered_map` in the copier).
`state_copier` allocates, remaps pointers through hash maps and needs a *fresh* destination.
That is fine every 238 frames for replay seeking, but probably too slow and allocation-heavy
for every 1-3 frames.

Plan, in order of escalation:

1. **Measure first.** Benchmark `copy_state` and one `next_frame` step on a late-game 4v4
   replay with Remastered limits (`OPENBW_REMASTERED_LIMITS`), in native and WASM builds. If a
   copy costs well under 1 ms, snapshot every 2-3 frames as ShieldBattery does and move on.
2. **Copy into a preallocated ring.** Change `state_copier` to copy into an existing `state` whose
   containers are already sized, reusing nodes instead of allocating. Keep a ring of
   `max_prediction / interval + 1` states. This removes the allocation cost, though pointer
   remapping remains.
3. **Index-based state (only if 1-2 are not enough).** Store links as indices, not pointers, so a
   snapshot is a memcpy of the containers. This is a big refactor of `bwgame.h` (22k lines) and
   should be avoided unless measurements force it.

Everything that is simulation state must be in the snapshot:
- `state`
- `action_state` (selection and control groups are part of the sim: actions act on the
  *simulated* selection)
- the action success/fail counters used by the hash.

Things that must stay outside:
- the transport
- the turn buffers
- the local camera and UI selection
- the sound queue
- the replay writer

### 2.4 Side effects during re-simulation

These hooks fire inside the sim and would repeat on every re-simulation:
- `state_functions::play_sound`, `on_unit_deselect`, `on_unit_destroy`, `on_kill_unit`,
  `on_player_eliminated` and `on_victory_state` (`bwgame.h:334-341`)
- `action_functions::on_action` (APM, `actions.h:1336`)
- `sync_functions::on_custom_action`
- replay saving in `process_messages`.

Approach:
- Add a `resimulating` flag to `state_functions`. While it is set:
  - Suppress sounds, except dedupe by `(frame, sound id, unit index)` so that a sound which
    only exists in the corrected timeline still plays once.
  - Defer `on_player_eliminated` and `on_victory_state` until the frame is confirmed. They are
    fences (§2.5).
  - Skip APM counting and replay writes.
- Replays are written from the confirmed turn stream only, never from simulated frames.

### 2.5 Fences: irreversible events

These events apply only once every earlier turn is confirmed:
- a player leaving or dropping (action 87)
- unit takeover (§3.3)
- game end

Implementation: `leave_game` / `kill_client` no longer execute action 87 at local time. They
schedule it as a turn for a specific frame. The relay (or, peer-to-peer, the agreed
membership) makes a dropped player's last turn final. Simulation of that frame waits until
`C` reaches it. This fixes the 2-peer-only assumption in the current `kill_client`.

### 2.6 Desync detection

- Replace the insync hash with a full-state hash over confirmed frames, every 8 confirmed
  steps. Hash *all* units, not only visible ones, plus bullets and the RNG, using indices
  rather than pointers.
- With a relay: majority vote, evict the minority, void the game if there is no majority (as
  ShieldBattery does). Peer-to-peer with 2 players: end the game with a desync error and save
  the replay plus both hashes for debugging.

### 2.7 Transport

Rollback assumes turns can arrive late or out of order and should not be blocked behind a lost
packet. TCP's head-of-line blocking undercuts that, so:

- Keep the `sync_server_noop` interface as the abstraction, but split it into a reliable
  control channel (join, slots, start, leave) and an unreliable turn channel with redundancy:
  each packet carries the last *k* unacknowledged turns.
- Native: UDP. Browser (Butterfly): WebRTC data channels in unordered/unreliable mode, or
  WebTransport, through a small relay. **A relay is recommended either way**:
  - it gives one place for session clocks and pacing;
  - it decides on drops and fences;
  - it handles majority desync votes;
  - it avoids NAT traversal problems.
- Frame numbers become `uint32_t` throughout. The `uint8_t` wraparound in
  `scheduled_actions` goes away.

### 2.8 Session start

The start handshake must agree on, and hash, everything that affects the sim:
- map hash
- game type
- slot → player and race mapping
- forces and alliances
- the team-mode settings from §3
- seed
- OpenBW build/version and the `OPENBW_REMASTERED_LIMITS` mode.

Mismatches refuse to start. Today only the seed and slots are exchanged.

### 2.9 Testing

- **Replay harness (first deliverable).** Play a replay through the rollback engine with
  synthetic per-player delay, jitter and loss. Assert that every confirmed frame's hash equals
  plain playback. This needs no networking and catches snapshot and side-effect bugs early.
- **Snapshot fuzz.** At random frames, snapshot, step N, restore, step N again, and compare
  hashes.
- **Soak.** Several headless clients over a lossy local relay for long games.
- **Benchmarks.** Copy cost, step cost and worst-case resimulation per frame, native vs WASM.

---

## 3. Team-play mode

A game setting, `team_play`, agreed at session start. It enables the three features below for
players who are mutually allied on the same force. All three are new OpenBW-only actions in a
reserved range, so they are deterministic, rollback-safe and recorded in OpenBW replays. Those
replays will not play in real StarCraft, which is acceptable for this mode.

New action, mirroring the existing `ext_cheat` (210) pattern:
**`211 ext_team`**, payload `[uint8 subtype][...]`. It is validated inside `read_action`, so
an invalid request is a deterministic no-op on every client.

Team-mode state lives in `state_base_copyable`, so it is automatically snapshotted:
`shared_control[12]` (a bitmask of players allowed to control each player's units) and the
team-mode flag.

### 3.1 Sending resources to a teammate

**Action:** `ext_team / transfer`, payload `[uint8 to_player][uint32 minerals][uint32 gas]`.

The action is accepted only if all of these hold:
- team mode is on;
- the two players are mutually allied, and on the same force;
- the receiving player has not been eliminated;
- the amounts are ≤ the sender's current minerals/gas at execution time, so a stale UI click
  cannot overdraw.

If accepted, it debits the sender and credits the receiver. It does *not* count as "gathered"
for score.

**UI: top-right panel**, next to the resource counters:

```
 ⛏ 1250   ⛽ 400   👥 58/100
 ┌ Send to ───────────────┐
 │ Alice   [+50][+100][+500] ⛏  [+50][+100] ⛽ │
 │ Bob     [+50][+100][+500] ⛏  [+50][+100] ⛽ │
 └────────────────────────┘
```

- One row per living ally, with fixed increments; shift-click sends ×5. A short "sent 100 ⛏ to
  Bob" line appears for both players, through a team chat/announcement hook rather than real
  chat.
- OpenBW's `ui/` has no HUD or text rendering (`draw_callback`, `ui.h:1436`, is the only overlay
  hook). For Butterfly, build the panel in **HTML/JS on top of the WASM canvas**, calling an
  exported `queue_team_transfer(to, minerals, gas)` function. That is far cheaper than adding
  font rendering to `ui/`. A native `ui/` version can come later.

### 3.2 Optionally controlling allied units

**Setting:** each player toggles "allow allies to control my units". This is like BW's
"allied control" in UMS. It is sent as `ext_team / set_shared_control [uint8 mask]`, stored in
`shared_control[owner]` and validated against the alliance and force. Default: off. The lobby
can force it on.

**Engine change:** add one helper,
`bool player_can_control(int player, int unit_owner)`, which is true when
`player == unit_owner || (team_play && shared_control[unit_owner] has player)`. Use it to
replace the ~30 `u->owner != owner` checks:
- `unit_can_receive_order` (`bwgame.h:17750`), `unit_can_use_tech`, `unit_can_research` and
  `unit_can_upgrade`;
- train, build, cancel_*, control groups, siege, cloak, stim, morph, unload and train_fighter
  in `actions.h` (lines 336-951).

Selection already allows any visible unit (`actions.h:268-331`), so mixed selections work as
soon as the command checks pass.

**Who pays and who owns.** The unit's owner pays, supplies and owns anything produced. If I
train a marine from my ally's barracks, my ally's minerals and supply are used and the marine
is theirs. This matches SC2 shared control and keeps economies separate, which is what the
transfer panel is for. Today's billing is inconsistent:
- research charges `u->owner` (`actions.h:677`);
- upgrades check level by `owner` but debit `u->owner` (`actions.h:702`).

These should be normalised to `u->owner` under team mode, while 1.16 behaviour stays unchanged
when it is off.

**UI:** allied units can be box-selected and commanded, and their selection circles use the
ally colour. Control groups may contain allied units, since `unit_id`s are already global.

### 3.3 Taking over a leaver's units

Today, `player_left`/`player_dropped` → `remove_player` gives **every unit to neutral player
11** (`bwgame.h:13057-13087`). The units stay on the map, inert and blocking.

There are two options:

| | A. Transfer ownership | B. Keep the player, grant control (**recommended**) |
|---|---|---|
| Mechanism | `give_unit_to(u, heir)` for each unit | The leaver stays a live player whose controller is "left". `shared_control[leaver]` is set to all remaining allies. |
| Upgrades/tech | Lost. Units get the heir's upgrades; `give_unit_to` only grants unit-specific ones. | Kept. |
| In-progress builds/research | Cancelled by `give_unit_to` | Kept |
| Supply / resources | The heir's supply cap can overflow; the leaver's bank is stranded | The leaver's bank and supply stay usable, and allies can transfer from it (§3.1) |
| Code | Small change, but loses state | Reuses §3.2 almost entirely |

**Recommendation: B.** On a leave or drop in team mode:
- if the player has at least one living ally, skip `make_unit_neutral`;
- set the controller to `user_left`, keep their `victory_state` pending, and set
  `shared_control[leaver]` to all remaining mutually-allied players;
- allow `ext_team / transfer` *from* a player who has left, when the sender is an ally
  controlling them.

The team is defeated as a whole when all of its players, including inherited ones, have no
units left. With no allies left, fall back to today's neutralisation.

Rollback: the leave is a fence (§2.5), so the control grant happens on a confirmed frame and is
never rolled back.

---

## 4. Phasing

| Phase | Deliverable | Depends on |
|---|---|---|
| 0 | Benchmarks for `copy_state` and step cost (native + WASM, Remastered limits). Merge `copier-sizes`. | — |
| 1 | Full-state hash + replay rollback harness with synthetic delay (no networking) | 0 |
| 2 | Side-effect suppression/dedupe, `resimulating` flag, confirmed-only replay writing | 1 |
| 3 | Snapshot ring (copy-into-existing if phase 0 says so) | 0, 1 |
| 4 | New turn protocol: uint32 frames, empty turns, N players, fenced leave, start handshake | 2, 3 |
| 5 | Transport: unreliable turn channel + relay; WebRTC/WebTransport for Butterfly | 4 |
| 6 | Adaptive per-player delay / rollback balance, pacing | 5 |
| T1 | `ext_team` action + transfer + HTML panel | independent of rollback (works on lockstep too) |
| T2 | `player_can_control`, shared-control toggle, billing normalisation | T1 |
| T3 | Leaver takeover (option B) | T2, and 4 for fencing under rollback |

The team features (T1-T3) don't depend on rollback and can ship first on the current netcode.
T3 under rollback needs phase 4's fenced leave.

## 5. Open questions

1. **Who is the netcode consumer?** Does Butterfly use `sync.h` today, or its own WASM-side
   netcode? This decides whether phase 4-5 changes `sync.h` or Butterfly's transport.
2. **Relay hosting.** Is a small relay (one process per region) acceptable, or must
   peer-to-peer work?
3. **Butterfly's "take over from a replay" start.** Rollback needs an agreed initial snapshot.
   Is it enough to start from replay frame *N* (everyone re-simulates the replay to *N*), or do
   we ship a serialised state?
4. **Transfer limits.** Should there be a cooldown, a minimum game time or a cap, to stop
   instant funnelling of all income to one player? Default proposal: no limits.
5. **Shared-control default.** Per-player opt-in (proposed), or lobby-wide always-on for team
   mode?

## Upstream

Upstreaming should be decided separately from the fork. OpenBW's author capped sync at 2 clients
on purpose, and these changes touch core paths. Suggested candidates once the work is proven
on the fork:
- the full-state hash and the replay rollback harness;
- the copy-into-existing snapshot;
- `player_can_control` with 1.16 behaviour unchanged by default.

The team-mode actions (211) and the relay are likely to stay fork-only.

## References

- ShieldBattery PR #1657, "Implement rollback netcode":
  https://github.com/ShieldBattery/ShieldBattery/pull/1657. Design notes are in `docs/rollback.md`
  in that repository.
- ShieldBattery PR #1365, "Integrate rally-point2":
  https://github.com/ShieldBattery/ShieldBattery/pull/1365
- GGPO: https://en.wikipedia.org/wiki/GGPO
