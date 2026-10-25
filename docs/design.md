## Core Loop

Sneak through a level, completing objectives and extracting without being seen. The player is instantly detected by enemies if the player enters their cone of vision. The cone of vision is visible.

Objectives are retrieval based, such as retrieving an item required for completing the mission or saving a hostage. Keys are a means to an objective, not an objective themselves: a key opens locked doors.

Saving a hostage means gathering them first, similar to picking up an item, and then leading them to the exit. **Once gathered, a hostage cannot be released.** They follow the player until the level ends. A following hostage entering an enemy's cone of vision counts as a detection. A captive (not yet gathered) hostage never counts as a detection: the enemy is considered to be in control of them.

Enemies can be defeated by melee actions used against them when standing outside their cone of vision. The melee attack hits only inside the player's facing arc.

All enemies have a cone of vision that extends outwards at a 45 degree half-angle (90 degrees total). Some enemies have a cone of vision that spans a greater distance than others.

Each time the player reaches an objective, defeated enemies respawn at their home positions, so the trip back to the exit stays tense. Objectives are always placed where no enemy's view can reach (the loader warns if a level breaks this).

Levels end at a **level exit**. An exit is either the extraction point or a door that leads to the next level. Both work the same way (see *Exit rule*).

- **Nouns:** player, enemy, item, key, hostage, wall, door, exit
- **Verbs:** move, attack, see, interact, extract

### Game over behavior

On game over, the world freezes in place with the detecting enemy's cone highlighted, so the player can see exactly what caught them. Retry is a key press, accepted only after a 1 second delay (real time) so a held key can't skip the freeze frame. The "press any key" prompt appears when the delay ends. There is no menu or fade, since one mistake is a fail and retries need to be nearly free. `Won` uses the same delay.

### Exit rule

The level is completed when all of these are true on the same frame:

1. The player is touching a `LevelExit` that is not `Locked`.
2. All required objectives are complete.

A gathered hostage is always right behind the player, so there is no separate "hostage must be nearby" check. If the player is spotted on that same frame, `PlayerSpotted` wins.

### Keys and carry-over between levels

- Keys are kept after opening a door, not consumed. One key opens every door that shares its ID.
- Objectives belong to a level and reset each time it starts.
- The player's key `Inventory` carries across levels. Retry restores the keys the player arrived with (see *Game States*).
- Hostages do not carry over. They are delivered when the level ends.

### Out of scope for v1

Sound and noise, distraction items, bodies or alerts from defeated enemies (no bodies, no reactions), multiple hostages (design for one first), pause menu, player health, game over on player death (not on instant detection), enemies that chase.


## Game States

```
Loading (assets, once) -> Menu -> Playing
Playing -> GameOver (spotted) -> Playing (retry)
Playing -> Won (exit reached) -> Playing (next level) or Menu (after last)
```

- **`OnEnter(Playing)`** always does the same thing, whether it's a first start, a retry, or the next level: despawn everything tagged `LevelEntity`, copy `LevelStartInventory` into `Inventory`, rebuild `Objectives` from the level, then spawn the level from `CurrentLevel`.
- **Do not despawn on `OnExit(Playing)`.** The frozen world must stay visible behind the GameOver and Won overlays. The `GameSet` systems are gated on `Playing`, so everything freezes on its own.
- **`OnEnter(Menu)`** also despawns `LevelEntity`, so the last level doesn't linger behind the menu.
- **Inventory snapshot:** `Inventory` is the live set of keys. `LevelStartInventory` is what the player arrived with. Resolution copies `Inventory` into `LevelStartInventory` on `LevelCompleted`. Starting a new game from the Menu clears both.
- **`CurrentLevel`** is incremented on leaving `Won` and reset when returning to the menu.
- **Retry and Won delay:** a `StateDelay` timer resource, set to 1 second and ticked with real time (`Time<Real>`), is inserted on entering `GameOver` and `Won`. Input systems for those states ignore keys until it finishes.
- **Resolution** is the only place that sets `NextState` out of `Playing`. It also stores the spotting enemy in a `SpottedBy` resource so the GameOver screen can highlight that enemy's cone.
- A mid-level restart key may need a dedicated `Restarting` state that bounces back to `Playing`, because setting `Playing` while already in `Playing` behaves differently across Bevy versions. Not needed for v1.


## ECS Model

### Components

**Markers**
- `Player`, `Enemy`, `Hostage`, `Wall`, `Door`
- `LevelExit`: on the extraction point and on any door that leads to the next level. Both share one code path.
- `LevelEntity`: everything spawned for a level (cleanup depends on it)
- `Solid`: blocks movement (walls, locked doors)
- `Occluder`: blocks line of sight (walls, locked doors)
- `Defeated`: inserted on a downed enemy; hidden and filtered out with `Without<Defeated>`
- `Captive` / `Following`: hostage state as two markers, so queries filter. Perception queries `With<Following>`. Gathering swaps `Captive` for `Following`, and that is permanent for the level.

**Movement and facing**
- `MoveIntent(Vec2)`: what the entity wants to do this frame (written by Input and AI, consumed by Movement)
- `DesiredFacing(Vec2)`: where an enemy wants to look
- `Facing(Vec2)`: the actual facing. Enemies turn toward `DesiredFacing`. The player's facing snaps to the last non-zero movement direction and persists while standing still.
- `TurnRate(f32)`: radians per second, for enemies, so cones sweep instead of snapping
- `Speed(f32)`
- `Collider { half_extents }`: size for collision
- `ActionIntent`: attack / interact flags for the frame

**Enemies**
- `VisionCone { range, half_angle }`
- `PatrolRoute { waypoints: Vec<Waypoint>, index, wait_timer }`
- `Waypoint { pos, wait, facing }`
- `Home { pos, route_index }`: where a respawned enemy returns to

**Player**
- `MeleeRange(f32)` and `MeleeArc { half_angle }`
- `PositionTrail(VecDeque<Vec2>)`: recent positions, recorded only after the player has moved a small minimum distance (not per frame)

**Objects**
- `Pickup(ItemKind)` where `ItemKind` is `Key(KeyId)` or `Mission(ItemId)`
- `Locked { key: KeyId }`: on doors; removed on unlock, along with `Solid` and `Occluder`
- `FollowOffset(f32)`: on hostages; how far behind the player (along the trail) they stay

### Resources

- `CurrentLevel`
- `Objectives`: the required list plus a completed set. Rebuilt for each level.
- `Inventory`: key IDs held right now
- `LevelStartInventory`: key IDs the player arrived with
- `SpottedBy(Option<Entity>)`
- `StateDelay`
- `DebugOverlay`

### Events / messages

All defined in `events.rs`, all past tense:

`PlayerSpotted { by }`, `ItemCollected`, `HostageCollected`, `ObjectiveReached { objective }`, `DoorUnlocked`, `EnemyDefeated`, `EnemyRespawned`, `LevelCompleted`

`ObjectiveReached` fires once per objective, when that objective is retrieved (a mission item is picked up or a hostage is gathered). Keys do not count. It is the trigger for enemy respawn.


## System Sets, in Order

Declared once in `schedule.rs` as a `GameSet` enum, chained, and gated with `run_if(in_state(AppState::Playing))`.

1. **Input:** read keys into `MoveIntent` and `ActionIntent` for the player. Update the player's `Facing` (last non-zero direction, snapping).
2. **AI:**
   - Advance patrol waypoints and run wait timers. Write enemy `MoveIntent` and `DesiredFacing`.
   - **Hostage follow:** place each `Following` hostage at `FollowOffset` path-length behind the player along `PositionTrail`. Measure along the trail, not in a straight line, so the hostage follows the same path the player took and never cuts corners. Right after being gathered, the hostage isn't on the trail yet, so it walks toward its trail point at the player's speed until it reaches it, then stays locked on.
   - **Respawn:** on `ObjectiveReached`, remove `Defeated` from every defeated enemy and reset it to `Home`. Respawn is immediate, since objectives are always out of every enemy's view. It reads the event on the next frame, and that one-frame delay is fine.
3. **Movement:** apply `MoveIntent` with wall collision (resolve X, then Y, so entities slide along walls). Turn enemy `Facing` toward `DesiredFacing` at `TurnRate`. Record the player's `PositionTrail`.
4. **Perception:** for each live enemy (`Without<Defeated>`), test the player and any `Following` hostage against its cone (range, half-angle, no `Occluder` in between). The test point is the center of the target. Emit `PlayerSpotted`. The test is a pure function with no Bevy types in its signature, and it is named generally (`in_cone`) because melee reuses it.
5. **Interaction:**
   - **On touch:** pick up keys and mission items (update `Inventory` and `Objectives` directly, emit `ItemCollected`).
   - **On interact press:** unlock a nearby `Locked` door if `Inventory` holds its key (remove `Locked`, `Solid`, `Occluder`; emit `DoorUnlocked`). The interact radius has to be a little larger than the door's collision bounds, since the door is `Solid`. Gather a nearby `Captive` hostage (swap to `Following`, emit `HostageCollected`).
   - **On attack press:** melee takedown. Find live enemies within `MeleeRange`, inside the `MeleeArc` around the player's `Facing`, with no `Occluder` in between (the same `in_cone` function as Perception). Hit the closest one only: insert `Defeated`, emit `EnemyDefeated`.
   - Emit `ObjectiveReached` when an objective is retrieved.
   - **Exit:** if the player touches an unlocked `LevelExit` and the exit rule passes, emit `LevelCompleted`.
6. **Resolution:** the only place that changes `AppState`, in order of precedence:
   1. `PlayerSpotted` -> store `SpottedBy`, go to `GameOver`
   2. `LevelCompleted` -> copy `Inventory` into `LevelStartInventory`, go to `Won`
7. **Presentation:** camera follow, draw cones, sound, HUD.

Notes:
- Perception runs before Interaction, so a player inside a cone is already caught. That means the melee rule "only from outside the cone" needs no separate check. Even if the same frame's attack defeats that enemy, `PlayerSpotted` still wins in Resolution.
- Respawn lives in AI rather than Resolution so that Resolution stays state-only.
- Systems inside the same set are unordered. Order them explicitly if one depends on another.
- Captive hostages never enter Perception. If a captive paces, do it as a Presentation-only animation so it isn't part of the simulation.


## Level Design

**ASCII grid + RON sidecar.** The grid holds geometry and the sidecar holds everything that isn't a position in space. Characters in the grid are IDs that link to sidecar entries. ("Sidecar" just means a companion file that travels with the main file. It describes the file's role, not its format.)

```
####################
#P...#.........#..i#
#....D..1......D...#
#.k.E#......2..#.h.#
####################
```

```ron
(
  carried_keys: [],
  enemies: {
    '1': (range: 6.0, route: [
      (pos: (8, 2), wait: 1.5, facing: (0.0, -1.0)),
      (pos: (11, 2), wait: 0.0, facing: (1.0, 0.0)),
    ]),
    '2': (range: 4.0, route: [
      (pos: (12, 3), wait: 0.0, facing: (-1.0, 0.0)),
    ]),
  },
  items: {
    'k': Key("red"),
    'i': Mission("intel"),
  },
  doors: { 'D': (locked_by: Some("red")) },
  hostages: { 'h': (follow_offset: 1.5) },
  objectives: [Collect("intel"), Rescue('h')],
)
```

Here `E` is the extraction point. The player takes the red key, goes through the hall past two guards, opens the vault door, grabs the intel and the hostage (the objectives, both out of every guard's view), and walks them back to `E`.

A door that leads to the next level is a door with `exit: true`. Doors can be locked or not, and exit or not, in any combination:

```ron
doors: {
  'D': (locked_by: Some("red")),                    // barrier, opens with the red key
  'X': (exit: true),                                // unlocked door that ends the level
  'Y': (exit: true, locked_by: Some("blue")),       // locked exit; the blue key may be carried in
},
```

In the Rust struct, put `#[serde(default)]` on `exit` and `locked_by` so they can be left out of the file. `locked_by` is an `Option`, so RON needs `Some("red")` unless you add `#![enable(implicit_some)]` at the top of the file.

### Loader boundary

The loader produces a plain Rust struct, `LevelDef` (size, wall cells, player start, exits, enemy defs, doors, items, hostages, objectives, carried keys). The spawn system consumes only `LevelDef`. That keeps the file format swappable, so it can start simple and be upgraded (for example to LDtk) without touching gameplay code.

### Conventions

- **Coordinates:** the grid and the sidecar use grid convention: x to the right, y *down*, row 0 at the top. A facing of `(0.0, -1.0)` means "up the screen". The loader converts to Bevy's y-up world coordinates once, and nowhere else.
- **No duplicated truth:** the grid character places the enemy. The first waypoint must equal that grid position, and the loader validates it.
- **Cone half-angle** is a shared constant (45 degrees). Only `range` varies per enemy.
- **Level files:** `assets/levels/01.txt` + `01.ron`, `02.txt` + `02.ron`, and so on. "Last level" means there is no next file (or a small manifest lists them).
- **Key IDs are global.** A red key opens every red door in every later level, so name IDs deliberately.
- **Carried keys:** `carried_keys` declares which keys the level assumes the player arrives with. It also lets a debug level-select start with the right inventory.

### Loader validation

Errors (the level refuses to load):
- Exactly one `P`
- At least one exit: an extraction point or an exit door
- Every row is the same length
- Every grid ID has a sidecar entry, and every sidecar entry appears in the grid
- Every locked door's key is either on the map or listed in `carried_keys`
- Every `Collect` / `Rescue` objective refers to something that exists
- No waypoint is inside a wall
- First waypoint matches the grid position

Warnings (the level still loads):
- **Objective lint:** an objective is visible to an enemy. For each objective, check every enemy's route (all waypoints plus a few sample points along each segment) and warn if any point is within the enemy's `range` with a clear line of sight to the objective. Ignore facing, and treat doors as non-blocking, so the check is conservative. Reuse `in_cone` or a plain range-and-line-of-sight check.
- A key that no door in any level uses. Run this across all levels, because a key can be meant for a later level.


## Plugin Layout

Feature plugins depend on shared definitions, never on each other.

```
src/
  main.rs         // add plugins only
  schedule.rs     // GameSet enum + ordering + Playing gate
  events.rs       // the contract between plugins
  state.rs        // AppState and transitions
  level/          // LevelDef, loader, validation, spawn/despawn
  movement/       // MoveIntent, Speed, Solid, collision, turning
  player/         // input -> intent, melee, interact
  enemy/          // patrol AI, Defeated, respawn
  hostage/        // gather, follow trail
  perception/     // in_cone (pure fn) + detect system
  objectives/     // Inventory, Objectives, doors, exits
  resolution/     // reads events, sets NextState
  ui/             // HUD, menus, game over / won screens, StateDelay
  debug/          // gizmos
```

Menu, GameOver and Won systems are not in `GameSet`. They use `OnEnter` / `OnExit` for one-time setup and `run_if(in_state(...))` on individual systems for per-frame logic.


## Debugging

For this game, debug visualization is part of development, not an extra. Everything goes through Bevy's `Gizmos`:

- **Detection cones:** draw them from the same function the detection system uses, not from the mesh the player sees. Then if the visible cone and the real test ever disagree, there will be two shapes that don't match. That's the bug that would otherwise feel like "the game cheated me."
- **Patrol routes:** lines between waypoints, with the active waypoint highlighted. Gizmos don't draw text, so use color or a small marker for the index.
- **Collision and occlusion:** outlines of every `Solid` and `Occluder`.
- **Player ranges:** the melee arc as a wedge in front of the player, and the interact radius.
- **Hostage:** the player's `PositionTrail` and the hostage's target point.
- **State markers:** a color per enemy state (active, `Defeated`), which makes the respawn logic much easier to debug.
- **Inventory:** the keys currently held, since "why does this door open?" can now be answered by a previous level.

Toggle it with a key (F1) through a `DebugOverlay` resource and a `run_if` on the draw systems. Other tools are worth the small cost: a reload-level key, a god-mode flag (checked in Resolution so it's still the single decision point), and a freeze toggle to inspect a frame.


## Tuning Values

Tune these while playtesting. None of them need to be in the sidecar yet.

- **Melee:** range about one tile, arc half-angle somewhere in 45 to 60 degrees (start at 50)
- **Interact radius:** a little larger than a door's collision bounds
- **Hostage follow offset:** per hostage, in the sidecar (`follow_offset`)
- **Trail sampling:** minimum distance between recorded points, a fraction of a tile
- **Cone test point:** the center of the target, for both the player and a following hostage


## Milestones

1. **Skeleton:** plugins, `AppState`, `GameSet` ordering, debug toggle. *Done when* the app switches between Menu and Playing.
2. **Room and movement:** `MoveIntent` plus collision, camera follow, hardcoded room. *Done when* sliding along walls feels right.
3. **Perception spike:** one static enemy, a visible cone that matches the test, instant detection, freeze frame, delayed instant retry. *Done when* the cone seen is the cone that catches you.
4. **Hostage follow spike (blank room):** gather a captive with interact, and have it trail the player through a doorway and around a corner. *Done when* it behaves in tight spaces. Do this before building the loader around hostages.
5. **Level loader:** ASCII + RON into `LevelDef`, validation, `LevelEntity` cleanup. *Done when* a broken level file gives a clear error message.
6. **Patrols:** waypoints, waits, turn rate. *Done when* a timing puzzle can be built from them.
7. **Exit loop:** extraction point and `LevelExit`, Won, next level, `Inventory` snapshot. *Done when* two levels play start to finish. This is already a complete game.
8. **Keys, doors, items:** interact to unlock, `Objectives` gates the exit, exit doors, `carried_keys`.
9. **Takedowns:** melee arc from outside the cone, `Defeated`.
10. **Hostage objective:** following hostages in Perception, `Rescue` objectives.
11. **Backtracking pressure:** respawn on `ObjectiveReached`, plus the objective lint.
12. **Polish:** rendered cones, sound, menus, more levels.
