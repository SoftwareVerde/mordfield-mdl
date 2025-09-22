# Mordfield Definition Language (MDL) Specification

---

## 1. Overview

The Mordfield Definition Language (MDL) is a JSON-based scripting system used to define campaign missions, events, and behaviors in Mordfield.

MDL files are parsed at mission load and control:

* Starting resources
* Unit restrictions
* Persistent and temporary modifiers
* Quest objectives
* Triggers and scripted sequences
* Failure and success conditions

---

## 2. Top-Level Structure

Each MDL file is a JSON object with the following main fields:

```jsonc
{
  "experienceModifier": 1.0, // Global XP multiplier for mission rewards.
  "resourcePacks": { },      // Starting resources available to the player.
  "flags": [ ],              // Mission-specific integer flags.
  "permanentUnitMods": [ ],  // Permanent adjustments to units.
  "startingUnitMods": [ ],   // Temporary mods applied at mission start.
  "disabledUnits": [ ],      // Units disallowed during the mission.
  "enabledUnits": [ ],       // Units explicitly allowed.
  "failureModes": [ ],       // Mission failure conditions.
  "quests": [ ],             // Objectives shown to the player.
  "triggers": [ ],           // Event-driven logic blocks.
  "init": [ ]                // Ordered actions run once after mission loads.
}
```

---

## 3. Flags

Flags are mission-scoped integer variables. They can be incremented, decremented, or checked in conditions.

Supported comparisons: `equal`, `lessThan`, `greaterThan`.

---

## 4. Unit Modifiers

### Permanent Unit Mods

Apply globally and persist across the mission.

```json
{ "type": "UNIT_COST", "player": "PLAYER", "unitTypes": ["OUTPOST"], "PRODUCTION": 10 }
```

### Starting Unit Mods

Applied at mission start.

```json
{ "type": "TURRET_STRATEGY", "player": "PLAYER", "radius": 5 }
```

---

## 5. Quests

Quests define objectives shown to the player.

```json
{ "title": "Build a Scrapyard.", "icon": "SCRAPYARD" }
```

---

## 6. Triggers

Triggers are event-driven logic. Each trigger watches for a condition (`type`) and executes `actions`.

See [Trigger Types](#trigger-types).

---

## 7. Actions

This section lists every supported opcode (action), its **signature** (parameter order and types), and important notes. Unless specified, actions run sequentially and await any animations/async work they start when the engine does so internally.

### Conventions & Types

* **Types**: `int`, `float`, `bool`, `string`, `Vector2i`, `UnitId`, `UnitId[]`, `UnitType`, `PerkType`, `TileType`, `ResourceType`, `Player` (`"PLAYER" | "ENEMY_PLAYER" | "NEUTRAL_PLAYER"`).
* **Selectors/Variables**: Any parameter may accept a selector like `"$(flagName)"`, `"$(varName)"`, `"$(TRIGGER:unit)"`, or a map marker `"$(MAP:name)"` if the underlying type matches.
* **Param Objects** are JSON objects with the listed fields; unrecognized fields are ignored.
* **Storage**: When a signature mentions `varName`, the result is stored under that name. Unless noted, storage uses **globalVariables**.

---

### Quest & Flow

| Opcode           | Signature                            | Notes                                                                                      |
| ---------------- | ------------------------------------ | ------------------------------------------------------------------------------------------ |
| `ADD_QUEST`      | `["ADD_QUEST", questIndex:int]`      | Adds a quest from top-level `quests` by index to the tracker.                              |
| `COMPLETE_QUEST` | `["COMPLETE_QUEST", questIndex:int]` | Marks the quest complete.                                                                  |
| `FAIL_QUEST`     | `["FAIL_QUEST", questIndex:int]`     | Marks the quest failed.                                                                    |
| `DELAY`          | `["DELAY", seconds:float]`           | Non-blocking wait before next action in the same list.                                     |
| `ON_TICK`        | `["ON_TICK", subActions:Action[]]`   | Runs `subActions` every game tick. Persisted across saves with a snapshot of local values. |
| `NOOP`           | `["NOOP", ...optional:any]`          | Does nothing. Useful for commenting/disabling steps.                                       |

---

### Trigger Control

| Opcode           | Signature                                                   | Notes                                                                                                       |
| ---------------- | ----------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `CREATE_TRIGGER` | `["CREATE_TRIGGER", name:string, { "paramValues": any[] }]` | Instantiates a named trigger declared in `triggers`. `paramValues` align with the trigger’s `params` order. |

---

### Camera & UI

| Opcode  | Signature | Notes |
| ---- | ------ | -------- |
| `MOVE_CAMERA`    | `["MOVE_CAMERA", position:Vector2i]`                                                              | Pans to tile and waits for camera to finish.                                                                                                                                 |                                                                                                                                                                                             |
| `UI_WASD_ICON`   | `["UI_WASD_ICON", isEnabled:bool]`                                                                | Shows/hides WASD hint.                                                                                                                                                       |                                                                                                                                                                                             |
| `ENABLE_UI_MOD`  | `["ENABLE_UI_MOD", "TURN_COUNTDOWN", opts?:{ setTitle?:string, setAmount?:int, addAmount?:int }]` | Adds a turn countdown widget if not present.                                                                                                                                 |                                                                                                                                                                                             |
| `UPDATE_UI_MOD`  | `["UPDATE_UI_MOD", "TURN_COUNTDOWN", opts?:{ setTitle?:string, setAmount?:int, addAmount?:int }]` | Updates the countdown widget.                                                                                                                                                |                                                                                                                                                                                             |
| `DISABLE_UI_MOD` | `["DISABLE_UI_MOD", "TURN_COUNTDOWN"]`                                                            | Removes the widget if present.                                                                                                                                               |                                                                                                                                                                                             |
| `NEW_WAVE_ALERT` | `["NEW_WAVE_ALERT"]`                                                                              | Displays an incoming-wave banner.                                                                                                                                            |                                                                                                                                                                                             |
| `SET_ZOOM`       | `["SET_ZOOM", zoom:float]`                                                                        | Sets camera zoom.                                                                                                                                                            |                                                                                                                                                                                             |
| `CINEMATIC`      | `["CINEMATIC", (unitArray:UnitId, position:Vector2i), params:{ unitText?:string, duration?:float, topText?:string, topColor?:string, bottomText?:string, bottomColor?:string }, concurrent?:Action ]` | Renders a unit-or-tile cinematic. If first arg resolves to units, the first valid unit is used; otherwise a tile at `position` is used. `concurrent` actions run when the cinematic starts. |
| `TOAST`          | `["TOAST", { text:string, position?:Vector2i, icon?:string }]`                                    | Shows a toast. If `position` is set, clicking recenters camera near it. `icon` accepts unit/resource names (and `AP`/`VICTORY`).                                             |                                                                                                                                                                                             |

---

### Variables, Flags & Math

| Opcode | Signature   | Notes |
| ------ | --------- | ------- |
| `FLAG`         | `["FLAG", flagName:string, cond:{ lessThan?:int, equal?:int, greaterThan?:int }, sub:Action[]]` | Runs `sub` only if the condition on the flag passes.                         |                                                                                                              |
| `INC_FLAG`     | `["INC_FLAG", flagName:string, amount:int]`                                                     | Adds (or subtracts) from a flag.                                             |                                                                                                              |
| `SET_FLAG`     | `["SET_FLAG", flagName:string, value:int]`                                                      | Sets a flag directly.                                                        |                                                                                                              |
| `COUNT`        | `["COUNT", varName:string, ints:int]`                                                                       | Stores `size(ints)` into `globalVariables[varName]`.                                                         |
| `PICK_RANDOM`  | `["PICK_RANDOM", varName:string, source:any]`                                                                       | If `source` is an array, picks one element; otherwise stores the value as-is. Stored in **globalVariables**. |
| `MAX_DISTANCE` | `["MAX_DISTANCE", varName:string, { sourcePosition:Vector2i, positions:Vector2i[] }]`           | Picks the farthest position and stores it in **local values** (not globals). |                                                                                                              |
| `GET_RESOURCE` | `["GET_RESOURCE", resource:ResourceType, player:Player, varName:string]`                                   | Reads a player resource into `globalVariables[varName]`.                                                     |

---

### Units: Spawning, Ownership, Movement & Combat

| Opcode | Signature   | Notes |
| ----- | --------- | ------ |
| `SPAWN_UNIT`     | `["SPAWN_UNIT", params:{ name?:string, player:Player, type:UnitType, position:Vector2i, flags:["OUT_OF_SIGHT"], forFree?:bool=true, force?:bool=true, warpIn?:bool=false, production?:int=0, unitFactory?:UnitId=-1, quantity?:int=1..7 }]` | Spawns one or more units. If `name` is set, stores spawned unit IDs in `units[name]`. `OUT_OF_SIGHT` tries to place out of enemy vision. If `unitFactory` is set, production/queue rules apply. `warpIn` plays an arrival effect. |                                                                                                |                                                                                           |
| `SPAWN_ATTACK`   | `["SPAWN_ATTACK", { name?:string, composition: { UnitType:int }[], position:Vector2i, destination:Vector2i }]` | Spawns a composition of units near `position` and orders them to `ATTACK_MOVE` toward `destination`. Stores spawned IDs in `units[name]` if set.                                           |                                                                                                                                                                                                                                   |                                                                                                |                                                                                           |
| `REMOVE`         | `["REMOVE", unitArray\:UnitId]`                                                                                                                                                                                     | Removes units from the map (not a kill).                                                                                                                                                                                          |                                                                                                |                                                                                           |
| `KILL`           | `["KILL", target:Player\|UnitId]`                                                                                                                                                                                                                            | If `target` resolves to a player string, kills that side. Otherwise kills the specified units. |                                                                                           |
| `SET_PLAYER`     | `["SET_PLAYER", unitArray:UnitId, player:Player]`                                                                                                                                                                     | Transfers ownership of units to another player.                                                                                                                                                                                   |                                                                                                |                                                                                           |
| `MOVE_TOWARDS`   | `["MOVE_TOWARDS", destination:Vector2i, unitArray:UnitId, options?:{ maxDistance?:int }, subActions?:Action ]`                                                                                                                               | Orders units to path toward a tile. Runs `subActions` after arrival.                                                                                                                                                              |                                                                                                |                                                                                           |
| `ATTACK_TILE`    | `["ATTACK_TILE", target:Vector2i, unitArray:UnitId, subActions?:Action ]`                                                                                                                                                               | Units attack a tile; engine chooses melee/ranged. `subActions` run after the attack animation.                                                                                                                                    |                                                                                                |                                                                                           |
| `RESET_MOVEMENT` | `["RESET_MOVEMENT", selector:Vector2i\|UnitId]`                                                                                     | Restores movement points for all units on a tile, listed units, or all units of a player. |
| `SET_HEALTH`     | `["SET_HEALTH", unitArray:UnitId, params:{ percent?:float\| amount?:int, delta?:int }]`                                                                                                                                                                                                    | Sets health by absolute `amount` or by `percent` of max, then applies `delta` if provided.     |                                                                                           |
| `UNIT_STRATEGY`  | `["UNIT_STRATEGY", unitArray:UnitId, cfg:{ goal:Goal, radius?:[int,int], preferCover?:bool, willScatter?:bool, destination?:Vector2i }]`                                                                             | Attaches an AI strategy engine to the units. Goals include `PROTECT`, `MOVE`, `ATTACK_MOVE`, `WAIT`.                                                                                                                              |                                                                                                |                                                                                           |
| `ENABLE_UNITS`   | `["ENABLE_UNITS", unitTypes:("ALL_UNIT_TYPES"\| UnitType\[])]`                                                                                                                                                                            | Enables listed unit types in the build UI.                                                                                                                                                                                        |                                                                                                |                                                                                           |
| `DISABLE_UNITS`  | `["DISABLE_UNITS", unitTypes:("ALL_UNIT_TYPES"\| UnitType)]`                                                                                                                                                                            | Disables listed unit types in the build UI.                                                                                                                                                                                       |                                                                                                |                                                                                           |
| `SPEAK`          | `["SPEAK", unit:UnitId, text:string]`                                                                                                                                                                       | Unit speech bubble.                                                                                                                                                                                                               |                                                                                                |                                                                                           |
| `SET_UNIT_NAME`  | `["SET_UNIT_NAME", name:string, unitArray:UnitId]`                                                                                                                                                                                     | Sets the display name on the **first** valid unit in the array.                                                                                                                                                                   |                                                                                                |                                                                                           |

---

### Tiles, Map & Vision

| Opcode | Signature | Notes |
| ----- | --------- | ------ |
| `DESTROY_TILE`  | `["DESTROY_TILE", position:Vector2i]`                                                                                                                                  | Sets tile health to 0 (advances damage state). |                                                                                                                                                          |
| `GET_TILES`     | `["GET_TILES", varName:string, { position:Vector2i, radius:int, excludeCenter?:bool=false, visionIsRequired?:bool=false, player?:Player, canFitUnits?:UnitId, canFitUnitTypes?:UnitType }]`           | Gathers tiles into `globalVariables[varName]` (as `Vector2i[]`). If `canFit*` is set, filters by placement rules (and optionally by `player` occupancy). |
| `GRANT_VISION`  | `["GRANT_VISION", position:Vector2i, radius:int]`                                                                                                                      | Reveals tiles in a radius for the player.      |                                                                                                                                                          |
| `REVOKE_VISION` | `["REVOKE_VISION", position:Vector2i, radius:int]`                                                                                                                     | Removes vision in a radius (restores fog).     |                                                                                                                                                          |

---

### Selection & Query

| Opcode      | Signature                                                                                                                                                      | Notes                                                                                                                                                                                                                                                                            |
| ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `GET_UNITS` | `["GET_UNITS", varName:string, selector: { player?:Player, tile?:Vector2i, radius?:int, units?:bool, buildings?:bool, turrets?:bool, unitTypes?:UnitType[] }]` | Stores a `UnitId[]` into `globalVariables[varName]`. Filtering defaults: if none of `units/buildings/turrets` is provided, all are included; otherwise only `true` ones are included (turrets are a subset of buildings). If `tile` is set, optional `radius` expands selection. |

---

### Construction Rules & UI Validations

| Opcode                 | Signature                                                                                                                            | Notes                                                    |                                                            |                                                                                                                                                         |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------- | ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `DISABLE_UNIT_CANCEL`  | `["DISABLE_UNIT_CANCEL", unitTypes:UnitType[]]`                                                                                      | Forbids canceling construction of the listed unit types. |                                                            |                                                                                                                                                         |
| `SET_BUILD_UNIT_FLAGS` | \`\["SET\_BUILD\_UNIT\_FLAGS", unitTypes\:UnitType, requirements: ( { flag\:string, lessThan?\:int, equal?\:int, greaterThan?\:int } | { tileType:(TileType                                     | TileType\[]), radius\:int, evolutionIndex?\:int } )\[] ]\` | Adds a UI validator. All listed requirements must pass for the given `unitTypes` when building *from a unit*. Failing sets an explanatory error string. |

---

### Resources, Rewards & Mission State

| Opcode          | Signature                                                                                                                                                                               | Notes                                                                                  |
| --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| `ADD_RESOURCES` | `["ADD_RESOURCES", player:Player, resources:{ POWER?:int, OIL?:int, COBALT?:int, HELIUM?:int, SCIENCE?:int, SCRAP?:int, PRODUCTION?:int, AP?:int }, options?:{ showToast?:bool=true }]` | Grants resources/AP and (optionally) shows toasts. `AP` affects max action points.     |
| `SET_STARS`     | `["SET_STARS", stars:int]`                                                                                                                                                              | Sets mission star count immediately.                                                   |
| `KILL`           | `["KILL", target:Player\|UnitId]`                                                                                                                                                                                                                            | If `target` resolves to a player string, kills that side. Otherwise kills the specified units. Used to end missions: killing `"PLAYER"` = defeat; killing `"ENEMY_PLAYER"` = victory. |
| `SPAWN_SCRAP`   | `["SPAWN_SCRAP", position:Vector2i, scrap:{ player?:Player, SCRAP?:int, POWER?:int, OIL?:int, COBALT?:int, HELIUM?:int, SCIENCE?:int, ACTION_POINT?:int }]`                             | Spawns scrap piles on the map at a tile. Optional `player` sets ownership.             |

---

### Auras & Effects

| Opcode        | Signature                                               | Notes  |                                                                 |
| ------------- | ------------------------------------------------------- | ------ | --------------------------------------------------------------- |
| `EMIT_AURA`   | `["EMIT_AURA", auraName:string, unitArray:UnitId]` | Applies an aura to each unit and tracks it in the save context. |
| `CANCEL_AURA` | `["CANCEL_AURA", auraName:string, unitArray:UnitId]` | Removes an aura from each unit (and updates saved context).     |

---

#### Notes on Storage Scopes

* **`globalVariables`**: Written by most query actions (`GET_UNITS`, `GET_TILES`, `PICK_RANDOM`, `GET_RESOURCE`, etc.). Accessible via `$(varName)`.
* **Local `values` (stack)**: Written by `MAX_DISTANCE` and used for temporary/local substitutions; also extended by `CREATE_TRIGGER`'s parameter injection and by trigger callback context (`TRIGGER:*`).

> Tip: If you need a value produced by `MAX_DISTANCE` to persist across triggers, write it back into `globalVariables` with `PICK_RANDOM` (storing the value itself) or by assigning it via a subsequent action that accepts a variable.

## 8. Trigger Types

```
NONE, OCCUPY_TILE, UNIT_DEATH, ALL_UNITS_DEAD, ALL_BUILDINGS_DEAD, ALL_CITIES_DEAD, RECEIVED_DAMAGE, ATTACK,
UNIT_COMPLETED, TILE_REVEALED, AP_BOOST, TILE_DESTROYED, PLAYER_DEATH, PERK_UNLOCKED, RESOURCE_COLLECTED, SCRAP_COLLECTED,
UNIT_QUEUED
```

---

## 9. Enums & Constants

* **Unit Types:** Full list of `Unit.Type` values (e.g., CITY, OUTPOST alias, LASERBOT, etc.).
* **Strategy Goals:** `MOVE, ATTACK_MOVE, PROTECT, WAIT, NONE`.
* **Tile Types:** `MOUNTAIN, MOUND, FOREST, COBALT, DESERT, WHEAT, WATER, CLIFF, FIELD, SWAMP, NONE, SNOW`.
* **Perk Types:** Full list (e.g., SCRAPYARD\_UNIT\_BONUS\_HEALTH, REVERSE\_ENGINEERING, etc.).
* **Resource Types:** `ELECTRICITY, OIL, COBALT, PRODUCTION, SCIENCE, WATER, HELIUM, SCRAP, NONE`.
* **Scrap Types:** `SCRAP, POWER, OIL, COBALT, HELIUM, SCIENCE, ACTION_POINT`.
* **Portraits:** `JEB`, `TOOTS`, `MEATBAG`.
* **UI Mods:** Currently `TURN_COUNTDOWN`.
* **Auras:** Armor Boost, Extra Attack, Heal, Emergency Production, …, Warping.
* **Special Constants:** `ALL_UNIT_TYPES`, `ALL_PERK_TYPES`.

---

## 10. Init Sequence

Executed once at mission start. Example:

```json
"init": [
  ["DISABLE_UNIT_CANCEL", ["OUTPOST"]],
  ["SET_BUILD_UNIT_FLAGS", ["OUTPOST"], [
    {"flag": "placeOutpostEnabled", "equal": 1},
    {"tileType": "WHEAT", "radius": 2, "evolutionIndex": 0}
  ]],
  ["ADD_RESOURCES", "PLAYER", {"POWER": 30}, {"showToast": false}]
]
```

---

## 11. Map References

Use `$(MAP:markerName)` to reference named tiles.

---

## 12. Stars and Mission End

* Stars: 0–3 rating.
* `KILL` actions end mission.

---

## 13. Notes & Gotchas

* Aliases: `OUTPOST`→`CITY`, `ICBOOM`→`ICMBOOM`.
* Evolution indices: 0 = undamaged.
* `RESET_MOVEMENT` allows re-use in same turn.
* `NOOP` useful for disabling actions.

---

## 14. Example

A simple MDL mission:

```json
{
  "experienceModifier": 1.0,
  "resourcePacks": { "POWER": 6, "SCRAP": 21 },
  "flags": [{ "name": "turnsRemaining", "value": 15 }],
  "quests": [{ "title": "Build a Scrapyard.", "icon": "SCRAPYARD" }],
  "triggers": [
    {
      "type": "UNIT_COMPLETED",
      "player": "PLAYER",
      "unitType": "SCRAPYARD",
      "actions": [["COMPLETE_QUEST", 0]]
    }
  ],
  "init": [
    ["ADD_RESOURCES", "PLAYER", {"POWER": 30}]
  ]
}
```
