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
Triggers define conditions that fire when events occur.
Each trigger has:
- `name` *(optional)*: Identifier for cross-referencing.
- `params`: Named parameters passed to sub-actions.
- `type`: Event type.
- `actions`: Sequence executed when trigger fires.
### Trigger Types
- **NONE**: No condition, used as manual invocation.
- **OCCUPY_TILE**: Fires when a player unit enters a tile. Supports `radius` and `visionIsRequired`.
- **UNIT_DEATH**: Fires when a unit dies. Supports `unit` or `whenAllDead`.
- **ALL_UNITS_DEAD**: Fires when all of a set are destroyed.
- **ALL_BUILDINGS_DEAD**: Fires when all buildings of a set are destroyed.
- **ALL_CITIES_DEAD**: Special-case city destruction.
- **RECEIVED_DAMAGE**: Fires when a unit takes damage. Supports `radius`.
- **ATTACK**: Fires when a unit attacks.
- **UNIT_COMPLETED**: Fires when construction finishes. Supports `radius`.
- **TILE_REVEALED**: Fires when fog of war clears a tile.
- **AP_BOOST**: Action point boost.
- **TILE_DESTROYED**: Fires when a tile transitions to a destroyed state.
---
## 7. Actions
Actions are the building blocks of scripting.
They are executed in order and may include nested conditional sequences.
### Common Actions
- **COMPLETE_QUEST** – Marks a quest complete.
- **ADD_QUEST** – Adds a new quest.
- **SPAWN_UNIT** – Creates a unit at a position.
- **SPAWN_ATTACK** – Spawns an enemy attack force.
- **UNIT_STRATEGY** – Assigns AI behaviors (e.g. PROTECT, MOVE).
- **TURRET_STRATEGY** – Assigns turret targeting/chase rules.
- **DIALOG** – Displays character text.
- **SPEAK** – Unit “speaks” a line.
- **MOVE_CAMERA** – Focuses the camera.
- **MOVE_TOWARDS** – Moves unit(s) toward a destination.
- **ATTACK_TILE** – Orders attack on a tile (supports `totalDamage`).
- **DESTROY_TILE** – Forces tile state to degrade (e.g. forest → damaged forest).
- **SET_STARS** – Assigns mission rating (1–3 stars).
- **KILL** – Ends the mission by eliminating a player or enemy.
- **ON_TICK** – Repeats nested actions each game tick.
- **DELAY** – Pauses execution for X seconds.
- **NOOP** – Does nothing (used for disabling/commenting sequences).
### Special Notes
- `ATTACK_TILE` executes sub-actions after damage is applied.
- `ON_TICK` is powerful but must be used carefully for performance.
- `DESTROY_TILE` changes terrain evolution stage (e.g. plains → crater).
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
