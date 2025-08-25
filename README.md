# Mordfield Definition Language (MDL) Specification
The **Mordfield Definition Language (MDL)** is a JSON-based scripting system used to define
campaign missions, events, and behaviors in *Mordfield*.
This document describes the structure, supported fields, and special behaviors of MDL.
---
## 1. Overview
MDL replaces direct GDScript mission scripting with a structured, data-driven format.
It allows designers to describe missions declaratively in JSON while the engine parses and executes
behaviors.
MDL files are parsed at mission load and control:
- Starting resources
- Unit restrictions
- Persistent and temporary modifiers
- Quest objectives
- Triggers and scripted sequences
- Failure and success conditions
---
## 2. Top-Level Structure
Each MDL file is a JSON object with the following main fields:
```jsonc
{
  "experienceModifier": 1.0, // Global XP multiplier for mission rewards.
  "resourcePacks": { },      // Starting resources available to the player.
  "flags": [ ],              // Mission-specific integer flags (variables used for gating logic and counters).
  "permanentUnitMods": [ ],  // Permanent adjustments to unit costs/stats for the mission.
  "startingUnitMods": [ ],   // Temporary mods or strategies applied to initial units when mission begins.
  "disabledUnits": [ ],      // Units disallowed from being built during the mission.
  "enabledUnits": [ ],       // Units explicitly allowed during the mission (overrides disables).
  "failureModes": [ ],       // Built-in mission failure conditions (e.g., no outposts).
  "quests": [ ],             // Objectives shown to the player; referenced by index in triggers/actions.
  "triggers": [ ],           // Event-driven logic blocks; can be anonymous (auto-activated) or named (instantiated later).
  "init": [ ]                // Ordered actions run once immediately after the mission loads.
}
```

---
## 3. Flags
Flags are mission-scoped variables stored as integers.
They can be incremented, decremented, checked in conditions, or compared against thresholds.
```json
{ "name": "turnsRemaining", "value": 15 }
```
Comparison operators supported:
- `equal`
- `lessThan`
- `greaterThan`
Future variants may include `lessThanOrEqual` and `greaterThanOrEqual`.
---
## 4. Unit Modifiers
### Permanent Unit Mods
Apply globally and persist across the mission.
```json
{ "type": "UNIT_COST", "player": "PLAYER", "unitTypes": ["OUTPOST"], "PRODUCTION": 10 }
```
### Starting Unit Mods
Convenience modifiers applied at mission start.
```json
{ "type": "TURRET_STRATEGY", "player": "PLAYER", "radius": 5 }
```
Notes:
- `UNIT_STRATEGY` and `TURRET_STRATEGY` in starting mods are broad-stroke directives.
- Fine-grained control should be implemented in triggers instead.
---
## 5. Quests
Quests define objectives shown to the player.
```json
{ "title": "Build a Scrapyard.", "icon": "SCRAPYARD" }
```
- `title`: Concise instruction.
- `icon`: Unit or object associated with the quest.
---
## 6. Triggers
Triggers define **event-driven logic** inside your custom maps. A trigger listens for a condition (e.g., a unit dying, a building being constructed, a tile being destroyed), and when that condition is met, it executes a list of **actions**.

---

### Declaring a Trigger

Triggers are declared in the top-level `triggers` array of the JSON file.

Each trigger has these common fields:

| Field              | Type                              | Description                                                                                                                                                |
| ------------------ | --------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `name`             | `String` *(optional)*             | A label for the trigger. If provided, it can be instantiated later using `CREATE_TRIGGER`. If omitted, the trigger is automatically active from the start. |
| `params`           | `Array[String]` *(optional)*      | Names of parameters that can be passed in when instantiating with `CREATE_TRIGGER`.                                                                        |
| `type`             | `String`                          | The event type that will fire the trigger (see [Trigger Types](#trigger-types)).                                                                           |
| `player`           | `String` *(optional)*             | Filters by player: `"PLAYER"`, `"ENEMY_PLAYER"`, or `"NEUTRAL_PLAYER"`.                                                                                    |
| `unitType`         | `String` *(optional)*             | Restrict the trigger to a specific unit type (e.g., `"SCRAPYARD"`, `"GRUEL_HAUS"`).                                                                        |
| `unit`             | `Int / Param` *(optional)*        | Restrict the trigger to a specific unit instance. Supports parameter substitution.                                                                         |
| `units`            | `Array[Int] / Param` *(optional)* | Restrict to any of several units.                                                                                                                          |
| `whenAllDead`      | `Array[Int] / Param` *(optional)* | Trigger fires only after all listed units are dead.                                                                                                        |
| `position`         | `Vector2i / Param` *(optional)*   | Location on the map that the trigger watches.                                                                                                              |
| `radius`           | `Int` *(optional)*                | Area of effect radius around `position`.                                                                                                                   |
| `visionIsRequired` | `Bool` *(optional)*               | If `true`, trigger only activates when the tile is visible to the watching player.                                                                         |
| `flags`            | `Array[Conditions]` *(optional)*  | Conditions on scenario flags that must pass before the trigger executes.                                                                                   |
| `actions`          | `Array[Actions]`                  | List of actions to perform when triggered.                                                                                                                 |

---

### Instantiating a Trigger

Triggers can be created dynamically during gameplay with the `CREATE_TRIGGER` action:

```jsonc
["CREATE_TRIGGER", "triggerName", { "paramValues": [ ... ] }]
```

* `"triggerName"` must match the `name` field of a declared trigger.
* `"paramValues"` must supply values for each `params` entry defined in the trigger declaration.

💡 This allows you to **re-use trigger definitions** with different positions, units, or contexts.

---

### Example – Simple Auto-Start Trigger

This trigger completes a quest as soon as the player builds a Scrapyard:

```jsonc
{
  "type": "UNIT_COMPLETED",
  "player": "PLAYER",
  "unitType": "SCRAPYARD",
  "actions": [
    ["COMPLETE_QUEST", 0]
  ]
}
```

---

### Example – Declared & Instantiated Trigger

1. **Declare a reusable trigger:**

```jsonc
{
  "name": "destroyedWheatTrigger",
  "params": ["unitPosition"],
  "type": "TILE_DESTROYED",
  "position": "$(unitPosition)",
  "actions": [
    ["DELAY", 2.0],
    ["DIALOG", "JEB", ["They destroyed our Wheat, Toots..."]],
    ["KILL", "PLAYER"]
  ]
}
```

2. **Instantiate it later:**

```jsonc
["CREATE_TRIGGER", "destroyedWheatTrigger", { "paramValues": ["$(TRIGGER:unit:position)"] }]
```

Here, the `unitPosition` parameter is supplied dynamically from the unit that caused the event.

---

### Trigger Types

The `"type"` field accepts these values (from the parser):

* **UNIT\_COMPLETED** – Fires when a unit/building finishes construction.
* **UNIT\_DEATH** – Fires when a unit dies.
* **TILE\_DESTROYED** – Fires when a tile is destroyed.
* **OCCUPY\_TILE** – Fires when a player’s unit enters/controls a tile.
* **Other engine-supported values** (future expansion may include more).

---

### Flag Conditions

Triggers can check scenario flags before running actions. Example:

```jsonc
"flags": [
  { "name": "turnsRemaining", "equal": 0 }
]
```

Supported comparisons:

* `lessThan`
* `equal`
* `greaterThan`

If the condition fails, the trigger stays alive and waits.

---

### Actions inside Triggers

Actions are executed sequentially when a trigger fires. Some examples:

* `["DIALOG", "JEB", ["Line of dialog"]]`
* `["SPAWN_UNIT", { "name": "enemyLeviathan", "player": "ENEMY_PLAYER", "type": "LEVIATHAN", "position": "$(MAP:wheatTile00)" }]`
* `["COMPLETE_QUEST", 1]`
* `["INC_FLAG", "destroyedWheatCounter", 1]`
* `["CREATE_TRIGGER", "leviathanKilled", { "paramValues": ["$(enemyLeviathan)"] }]`

See the [Actions Reference](actions.md) for the full list.

---

### Example workflow:

1. Player builds a Gruel Haus → fires a `UNIT_COMPLETED` trigger.
2. That trigger spawns an enemy Scout Drone.
3. The trigger also **creates a new trigger** (`buildGruelHaus`) that watches the Wheat tile for an upcoming attack.
4. The chain continues with new triggers and quests.
---
## 7. Actions

### Overview

Actions are the commands executed by a scenario when a trigger fires (or during `init`).

Each action is written as a JSON array:

```jsonc
["OPCODE", param1, param2, {...}]
```

* **`OPCODE`** (string): the action type.
* **`param1..n`**: positional parameters (numbers, strings, arrays, or objects).
* Many parameters can also use **selectors/variables** (like `"$(flagName)"`, `"$(TRIGGER:unit)"`, `"$(MAP:startTile)"`).

Selectors/variables like `$(...)` can be used anywhere a parameter accepts that type (e.g., `Vector2i`, unitId, arrays, flags).

> Notation:
>
> * `Vector2i` = tile coordinate
> * `unitId[]` = array of unit IDs (or a variable that resolves to one)
> * `player` = `"PLAYER" | "ENEMY_PLAYER" | "NEUTRAL_PLAYER"`
> * Conditions support: `lessThan | equal | greaterThan`

| Opcode                 | Signature                                                       | Brief                                       | Key Parameters / Notes                                                                                                                                 |                                                                                                       |                                 |
| ---------------------- | --------------------------------------------------------------- | ------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------- | ------------------------------- |
| `COMPLETE_QUEST`       | `["COMPLETE_QUEST", questIndex:int]`                            | Marks a quest complete.                     | `questIndex` is the index in top-level `quests`.                                                                                                       |                                                                                                       |                                 |
| `ADD_QUEST`            | `["ADD_QUEST", questIndex:int]`                                 | Adds a quest to the tracker.                | Shows quest in UI; does not auto-complete.                                                                                                             |                                                                                                       |                                 |
| `CREATE_TRIGGER`       | `["CREATE_TRIGGER", name:str, {paramValues: any[]}]`            | Instantiates a named trigger.               | `name` must match a trigger with `name`; `paramValues` align with declared `params` order.                                                             |                                                                                                       |                                 |
| `DELAY`                | `["DELAY", seconds:float]`                                      | Waits before next action.                   | Non-blocking; subsequent actions continue after delay.                                                                                                 |                                                                                                       |                                 |
| `DIALOG`               | `["DIALOG", speaker:str, lines:str[]]`                          | Shows campaign dialog.                      | `speaker`: \`"JEB"                                                                                                                                     | "TOOTS"                                                                                               | "MEATBAG"\` (mapped portraits). |
| `SPEAK`         | `["SPEAK", unit:unitId\|var, text:str]`                                                               | Displays a speech bubble above a unit.                                                           | `unit`: must resolve to a valid unit ID (or a variable containing one). `text`: string to display.                                                                                    |
| `KILL`                 | `["KILL", player:str]`                                          | Eliminates a player.                        | Ends side immediately (`setIsDead(true)`).                                                                                                             |                                                                                                       |                                 |
| `FLAG`                 | `["FLAG", flagName:str, cond:obj, subActions:Action[]]`         | Conditional block on a flag.                | `cond` supports `lessThan/equal/greaterThan`. If not met, nothing runs.                                                                                |                                                                                                       |                                 |
| `INC_FLAG`             | `["INC_FLAG", flagName:str, amount:int]`                        | Adds delta to a flag.                       | Negative amounts allowed.                                                                                                                              |                                                                                                       |                                 |
| `SPAWN_UNIT`           | `["SPAWN_UNIT", params:obj]`                                    | Spawns one unit.                            | `params`: `name?:str` (stores unitId under that var), `player`, `type`, `position:Vector2i`, `flags?:["OUT_OF_SIGHT"]`.                                |                                                                                                       |                                 | `MOVE_TOWARDS`  | `["MOVE_TOWARDS", destination:Vector2i, unitArray:unitId[]\|var, options?:obj, subActions?:Action[]]` | Orders units to move toward a tile.                                                              | `destination`: target tile. `unitArray`: array of unit IDs or variable containing one. `options.maxDistance?:int`. `subActions`: actions run after movement completes.                |
| `ATTACK_TILE`          | \`\["ATTACK\_TILE", target\:Vector2i, unitArray\:unitId         | var, subActions?\:Action]\`                 | Orders units to attack a tile.                                                                                                                         | Auto-picks melee/ranged; `subActions` run post-attack.                                                |                                 |
| `DESTROY_TILE`         | `["DESTROY_TILE", position:Vector2i]`                           | Destroys a tile immediately.                | Sets tile health to 0.                                                                                                                                 |                                                                                                       |                                 |
| `UNIT_STRATEGY` | `["UNIT_STRATEGY", unit:unitId\|var, cfg:obj]`                                                        | Assigns an AI behavior strategy to a unit.                                                       | `unit`: unit ID or variable. `cfg`: `{ "goal": "PROTECT\|MOVE\|ATTACK_MOVE\|...", "radius": [min:int, max:int], "preferCover"?:bool, "willScatter"?:bool, "destination"?:Vector2i }`. |
| `SPAWN_ATTACK`         | `["SPAWN_ATTACK", params:obj]`                                  | Spawns a group, sets attack-move.           | `params`: `name?:str` (stores unitIds), `composition:obj[]` (per ring/tile-set), `position:Vector2i`, `destination:Vector2i`.                          |                                                                                                       |                                 |
| `RESET_MOVEMENT`       | `["RESET_MOVEMENT", selector]`                                  | Restores movement points.                   | `selector`: `Vector2i` (tile → units on it), \`unitId\[]                                                                                               | var`, or `"PLAYER"                                                                                    | "ENEMY\_PLAYER"\`.              |
| `DISABLE_UNIT_CANCEL`  | `["DISABLE_UNIT_CANCEL", unitTypes:str[]]`                      | Blocks canceling listed unit constructions. | Applies a UI validator that forbids cancel for those types.                                                                                            |                                                                                                       |                                 |
| `SET_BUILD_UNIT_FLAGS` | `["SET_BUILD_UNIT_FLAGS", unitTypes:str[], requirements:obj[]]` | Gating rules for building units.            | Requirements include flag conditions and/or tile constraints: `tileType`, `radius`, `evolutionIndex` array (e.g., `[0]` undamaged).                    |                                                                                                       |                                 |
| `UI_WASD_ICON`         | `["UI_WASD_ICON", isEnabled:bool]`                              | Toggle WASD hint.                           | `true` to show; `false` to hide.                                                                                                                       |                                                                                                       |                                 |
| `MOVE_CAMERA`          | `["MOVE_CAMERA", position:Vector2i]`                            | Pans/centers camera to tile.                | Awaits camera move completion before proceeding.                                                                                                       |                                                                                                       |                                 |
| `GET_UNITS`            | `["GET_UNITS", varName:str, selector:obj]`                      | Stores unit IDs into a variable.            | Selectors: `{ "player": player }` for all units of a player; `{ "tile": Vector2i }` for units on a tile. Stores `int[]` in `globalVariables[varName]`. |                                                                                                       |                                 |
| `GET_TILES`            | `["GET_TILES", varName:str, params:obj]`                        | Collects tiles in radius.                   | `params`: `position:Vector2i`, `radius:int`, `excludeCenter?:bool`, `visionIsRequired?:bool`, `player?:player`. Stores `Vector2i[]`.                   |                                                                                                       |                                 |
| `PICK_RANDOM`   | `["PICK_RANDOM", varName:str, source:any\|arrayVar]`                                                  | Chooses a random element from an array (or a value if not an array) and stores it in a variable. | `varName`: name of the variable created in `globalVariables`. `source`: an array variable, or any other value (if not an array, the value is just stored directly).                   |
| `MAX_DISTANCE`         | `["MAX_DISTANCE", varName:str, params:obj]`                     | Selects farthest position.                  | `params`: `sourcePosition:Vector2i`, `positions:Vector2i[]`. Stores one `Vector2i` in local values.                                                    |                                                                                                       |                                 |
| `ENABLE_UI_MOD`        | `["ENABLE_UI_MOD", "TURN_COUNTDOWN", options:obj]`              | Adds the turn countdown widget.             | `options`: `setTitle?:str`, `setAmount?:int`, `addAmount?:int`.                                                                                        |                                                                                                       |                                 |
| `UPDATE_UI_MOD`        | `["UPDATE_UI_MOD", "TURN_COUNTDOWN", options:obj]`              | Updates countdown widget.                   | Same fields as above.                                                                                                                                  |                                                                                                       |                                 |
| `DISABLE_UI_MOD`       | `["DISABLE_UI_MOD", "TURN_COUNTDOWN"]`                          | Removes countdown widget.                   | Safe if not present (no-op logged).                                                                                                                    |                                                                                                       |                                 |
| `NEW_WAVE_ALERT`       | `["NEW_WAVE_ALERT"]`                                            | Shows incoming wave banner.                 | Pure UI effect.                                                                                                                                        |                                                                                                       |                                 |
| `SET_STARS`            | `["SET_STARS", starCount:int]`                                  | Sets mission stars.                         | Immediate UI/mission state update.                                                                                                                     |                                                                                                       |                                 |
| `ADD_RESOURCES`        | `["ADD_RESOURCES", player:str, resources:obj, options?:obj]`    | Grants resources/AP.                        | `resources` keys: `POWER,OIL,COBALT,HELIUM,SCIENCE,SCRAP,PRODUCTION,AP`; `options.showToast?:bool`.                                                    |                                                                                                       |                                 |
| `NOOP`                 | `["NOOP", ...optional]`                                         | Placeholder / spacer.                       | Accepts unused extra params without effect.                                                                                                            |                                                                                                       |                                 |

**Selectors and dynamic values:**

* **Flags / locals / globals:** `"$(flagName)"`, `"$(someLocalVar)"`, `"$(someGlobalVar)"`
* **Trigger context:** `"$(TRIGGER:unit)"`, `"$(TRIGGER:totalDamage)"`, `"$(TRIGGER:unit:position)"`
* **Units captured by name:** when `SPAWN_UNIT` or `SPAWN_ATTACK` uses `"name"`, it writes IDs into that variable, usable later as `["$(nameVar)"]`.
* **Map markers:** `"$(MAP:markerName)"` populated from `triggerPoints` at scenario init.

---

### Quest Actions

#### `COMPLETE_QUEST`

Marks a quest complete.

**Format**:

```jsonc
["COMPLETE_QUEST", questIndex]
```

* `questIndex` (int): index from the `quests` array.

---

#### `ADD_QUEST`

Adds a quest to the quest tracker so the player can see it.

**Format**:

```jsonc
["ADD_QUEST", questIndex]
```

* `questIndex` (int): index from the `quests` array.

---

### Trigger Actions

#### `CREATE_TRIGGER`

Instantiates a trigger previously declared by name.

**Format**:

```jsonc
["CREATE_TRIGGER", "triggerName", { "paramValues": [ ... ] }]
```

* `triggerName` (string): must match a trigger in the `triggers` array with a `name`.
* `paramValues` (array): ordered values passed into the trigger’s `params`.

---

#### `ON_TICK`

Runs sub-actions once per game tick.

**Format**:

```jsonc
["ON_TICK", [ ...subActions... ]]
```

* `subActions` (array of actions): executed every tick.

---

### Timing & Flow Control

#### `DELAY`

Waits before continuing.

**Format**:

```jsonc
["DELAY", seconds]
```

* `seconds` (float): time to wait.

---

#### `FLAG`

Conditionally executes sub-actions if a flag meets the condition.

**Format**:

```jsonc
["FLAG", "flagName", {condition}, [ ...subActions... ]]
```

* `flagName` (string): name of the flag.
* `{condition}` (object): supports `lessThan`, `equal`, or `greaterThan`.
* `subActions`: only run if condition passes.

---

#### `INC_FLAG`

Changes a flag value by a delta.

**Format**:

```jsonc
["INC_FLAG", "flagName", amount]
```

* `flagName` (string)
* `amount` (int): positive or negative.

---

#### `NOOP`

Does nothing. Useful as a placeholder.

```jsonc
["NOOP"]
```

---

### Camera & UI

#### `MOVE_CAMERA`

Moves the camera to a tile.

**Format**:

```jsonc
["MOVE_CAMERA", position]
```

* `position` (Vector2i or selector): target tile position.

---

#### `UI_WASD_ICON`

Toggles the WASD movement hint.

**Format**:

```jsonc
["UI_WASD_ICON", isEnabled]
```

* `isEnabled` (bool): `true` shows, `false` hides.

---

#### `ENABLE_UI_MOD`

Adds a UI widget (currently only `"TURN_COUNTDOWN"` supported).

**Format**:

```jsonc
["ENABLE_UI_MOD", "TURN_COUNTDOWN", {options}]
```

* `options` may include:

  * `setTitle` (string)
  * `setAmount` (int)
  * `addAmount` (int)

---

#### `UPDATE_UI_MOD`

Updates an existing UI mod.

**Format**:

```jsonc
["UPDATE_UI_MOD", "TURN_COUNTDOWN", {options}]
```

Same `options` as `ENABLE_UI_MOD`.

---

#### `DISABLE_UI_MOD`

Removes an active UI mod.

**Format**:

```jsonc
["DISABLE_UI_MOD", "TURN_COUNTDOWN"]
```

---

#### `NEW_WAVE_ALERT`

Displays the “Incoming Wave” banner.

```jsonc
["NEW_WAVE_ALERT"]
```

---

### Dialog & Flavor

#### `DIALOG`

Shows dialogue in the campaign panel.

**Format**:

```jsonc
["DIALOG", "speaker", [lines]]
```

* `speaker` (string): `"JEB"`, `"TOOTS"`, or `"MEATBAG"`.
* `lines` (array of strings): dialogue text.

---

#### `SPEAK`

Displays a speech bubble from a unit.

**Format**:

```jsonc
["SPEAK", unit, text]
```

* `unit` (unitId or variable)
* `text` (string)

---

### Units

#### `SPAWN_UNIT`

Creates a new unit.

**Format**:

```jsonc
["SPAWN_UNIT", {
  "name": "varName",
  "player": "PLAYER|ENEMY_PLAYER|NEUTRAL_PLAYER",
  "type": "UNIT_TYPE",
  "position": position,
  "flags": ["OUT_OF_SIGHT"]
}]
```

* `name` (string): if given, saves the spawned unit’s ID into a variable.
* `player` (string): owning player.
* `type` (string): unit type.
* `position` (Vector2i or selector).
* `flags`: special placement options.

---

#### `MOVE_TOWARDS`

Orders units to move toward a tile.

**Format**:

```jsonc
["MOVE_TOWARDS", destination, unitArray, {options}, [subActions]]
```

* `destination` (Vector2i or selector).
* `unitArray` (array of unit IDs / variable).
* `options`:

  * `maxDistance` (int): maximum allowed move distance.
* `subActions`: run after movement completes.

---

#### `ATTACK_TILE`

Orders units to attack a tile.

**Format**:

```jsonc
["ATTACK_TILE", targetTile, unitArray, [subActions]]
```

* `targetTile` (Vector2i or selector).
* `unitArray` (array of unit IDs / variable).
* `subActions`: run after attack finishes.

---

#### `DESTROY_TILE`

Destroys a tile.

**Format**:

```jsonc
["DESTROY_TILE", position]
```

* `position` (Vector2i or selector).

---

#### `UNIT_STRATEGY`

Assigns an AI behavior to a unit.

**Format**:

```jsonc
["UNIT_STRATEGY", unit, {
  "goal": "PROTECT|MOVE|ATTACK_MOVE|...",
  "radius": [min, max],
  "preferCover": true,
  "willScatter": false,
  "destination": position
}]
```

* `unit` (unitId or variable).
* Options configure AI behavior.

---

#### `SPAWN_ATTACK`

Spawns multiple units and gives them an attack-move strategy.

**Format**:

```jsonc
["SPAWN_ATTACK", {
  "name": "varName",
  "composition": [
    { "LASERBOT": 3, "AT_LAUNCHER": 1 },
    { "LEVIATHAN": 1 }
  ],
  "position": spawnTile,
  "destination": targetTile
}]
```

* `name` (string): saves unit IDs into this variable.
* `composition` (array): groups of unit types + counts.
* `position` (Vector2i).
* `destination` (Vector2i).

---

#### `RESET_MOVEMENT`

Restores unit movement points.

**Format**:

```jsonc
["RESET_MOVEMENT", selector]
```

* `selector` can be:

  * unit position (`Vector2i`)
  * unit array (variable)
  * `"PLAYER"` or `"ENEMY_PLAYER"`

---

#### `DISABLE_UNIT_CANCEL`

Blocks canceling production for listed unit types.

**Format**:

```jsonc
["DISABLE_UNIT_CANCEL", ["UNIT_TYPE", ...]]
```

---

#### `SET_BUILD_UNIT_FLAGS`

Restricts construction until conditions are met.

**Format**:

```jsonc
["SET_BUILD_UNIT_FLAGS", ["UNIT_TYPE", ...], [requirements]]
```

* `requirements` can include:

  * `{"flag": "flagName", "lessThan|equal|greaterThan": value}`
  * `{"tileType": "WHEAT", "radius": 2, "evolutionIndex": 0}`

---

### Resource & Scenario State

#### `ADD_RESOURCES`

Adds resources to a player.

**Format**:

```jsonc
["ADD_RESOURCES", "PLAYER|ENEMY_PLAYER", {resources}, {options}]
```

* `resources`: `{ "POWER": 30, "SCRAP": 10 }`
* `options`: `{ "showToast": true|false }`

---

#### `SET_STARS`

Sets star rating.

**Format**:

```jsonc
["SET_STARS", starCount]
```

---

### Variables & Map Data

#### `GET_TILES`

Collects tiles around a position into a variable.

**Format**:

```jsonc
["GET_TILES", "varName", {
  "position": position,
  "radius": int,
  "excludeCenter": bool,
  "visionIsRequired": bool,
  "player": "PLAYER|ENEMY_PLAYER"
}]
```

---

#### `PICK_RANDOM`

Chooses a random element from an array and stores it.

**Format**:

```jsonc
["PICK_RANDOM", "varName", sourceArray]
```

* `sourceArray`: must resolve to an array variable.

---

#### `MAX_DISTANCE`

Selects the farthest tile from a given point.

**Format**:

```jsonc
["MAX_DISTANCE", "varName", {
  "sourcePosition": pos,
  "positions": [pos1, pos2, ...]
}]
```

---

#### `GET_UNITS`

Stores units into a variable.

**Format**:

```jsonc
["GET_UNITS", "varName", selector]
```

* `varName` (string): created variable.
* `selector` (object):

  * `{ "player": "PLAYER|ENEMY_PLAYER" }`
  * `{ "tile": position }`

---

### Player Control

#### `KILL`

Eliminates a player.

**Format**:

```jsonc
["KILL", "PLAYER|ENEMY_PLAYER|NEUTRAL_PLAYER"]
```

---
## 8. Init Sequence
Executed once at mission start.
Sets up resources, quests, camera, and initial triggers.
Example:
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
## 9. Map References
MDL supports referencing **named tiles** from the map file.
These appear as:
```json
"$(MAP:startTile)"
```
The map editor allows designers to name tiles for scripting use.
---
## 10. Stars and Mission End
- **Stars**:
- 0 = incomplete
- 1 = minimal success
- 2 = good performance
- 3 = perfect performance
- **Kill Actions**:
- `"KILL", "PLAYER"` – Ends mission in defeat.
- `"KILL", "ENEMY_PLAYER"` – Ends mission in victory.
---
## 11. Notes & Gotchas
- **Turret Strategy**: Does not support inner/outer radius since turrets cannot move.
- Inner radius: always engage targets.
- Outer radius: how far they will chase once engaged.
- **Resetting Movement**: Allows a unit to move again during the same turn.
- **Destroy Tile**: Evolves a tile’s damage state.
- **NOOP**: Useful for temporarily disabling actions but cannot wrap triggers.
---
## 12. Future Considerations
Planned or possible improvements:
- Extended flag comparisons (`greaterThanOrEqual`, `lessThanOrEqual`).
- More granular unit selectors for `GET_UNITS`.
- Boolean combinations (`OR`) in build constraints.
- Comment support beyond `NOOP`.
---
## 13. Example
A simplified example mission:
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
---
## 14. Summary
MDL provides a structured, data-driven approach to scripting campaign missions.
It empowers designers to define missions declaratively without coding in GDScript.
The system is extensible, human-readable, and supports complex behaviors through triggers and
actions.
