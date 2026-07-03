# Custom Changes — Wownerubian

Este documento lista todas las modificaciones custom sobre AzerothCore para el proyecto Wownerubian.

## Core Changes

### 7. Cross-Faction Trade — Same group/raid bypass

**File:** `src/server/game/Handlers/TradeHandler.cpp`

Allows cross-faction trading when both players are in the same party or raid group.
Previously the server rejected trade between Alliance and Horde with `TRADE_STATUS_WRONG_FACTION`
unless the initiator had `RBAC_PERM_ALLOW_TWO_SIDE_TRADE` (GM permission 51).

**Change:** Added `&& !_player->IsInSameRaidWith(pOther)` to the faction check condition.

## Core Changes

### 1. ChannelMgr — `SetChannel()` method

**Files:** `src/server/game/Chat/Channels/ChannelMgr.cpp`, `src/server/game/Chat/Channels/ChannelMgr.h`

Added public method `SetChannel(name, channel)` to manually register/override a `Channel` object in the channel manager. Used by `mod-world-chat` to inject the `World` channel.

**Diff:** +8 lines cpp, +1 line header.

### 2. Wintergrasp Portal — Attackers can teleport

**File:** `src/server/scripts/Northrend/zone_wintergrasp.cpp`

Removed the defender-only check from `spell_wintergrasp_portal`. Previously: `wintergrasp->GetDefenderTeam() != target->GetTeamId()` blocked attackers. Now: any player level 75+ can use the Dalaran → Wintergrasp portal.

**Diff:** `- (wintergrasp->GetDefenderTeam() != target->GetTeamId())`

### 3. Valithria Dreamwalker — Wipe reset & respawn fixes

**File:** `src/server/scripts/Northrend/IcecrownCitadel/boss_valithria_dreamwalker.cpp`

Three fixes:
- **Respawn Archmages after wipe:** Risen Archmages killed early corpse-decay and leave the world; the existing `CreatureWorker` can't reach them. New code iterates `map->GetCreatureRespawnTimes()` and forces respawn of `NPC_RISEN_ARCHMAGE` and `NPC_VALITHRIA_DREAMWALKER` at `GameTime + 11s`.
- **DespawnOrUnsummon replacement:** Replaced `RemoveCorpse(false)` + `SetRespawnTime(11)` with `DespawnOrUnsummon(0ms, 11s)` which works even after corpse decay.
- **Wipe detection:** Custom `UpdateAI` checks if any player is alive during `IN_PROGRESS`; if all dead, calls `DoAction(ACTION_DEATH)` to reset the encounter.

**Diff:** +57 lines, new include `<GameTime.h>`.

### 4. Halls of Reflection — Falric & Marwyn wipe respawn

**File:** `src/server/scripts/Northrend/FrozenHalls/HallsOfReflection/instance_halls_of_reflection.cpp`

On wipe reset: if Falric or Marwyn are dead they now get `Respawn()` + `SetVisible(false)`, and `_falricPhaseComplete` resets to `false` so the wave sequence can restart from scratch.

**Diff:** +12 lines.

### 4b. Halls of Reflection — Boss immunity fix (SetImmuneToAll + ClearUnitState EVADE)

**Files:** `src/server/scripts/Northrend/FrozenHalls/HallsOfReflection/boss_falric.cpp`, `boss_marwyn.cpp`

Two fixes in `DoAction(1)`:
1. **`me->SetImmuneToPC(false)` → `me->SetImmuneToAll(false)`**: Reset() sets `SetImmuneToAll(true)` (both IMMUNE_TO_PC + IMMUNE_TO_NPC), but DoAction(1) only cleared IMMUNE_TO_PC.
2. **Added `me->ClearUnitState(UNIT_STATE_EVADE)`**: HandleWaveWipe calls EnterEvadeMode() which sets UNIT_STATE_EVADE. DoZoneInCombat() checks IsInEvadeMode() and returns immediately — boss never engages combat. ClearEvadeState fixes this.

**Diff:** 4 lines (2 per boss).

### 5. CMakeLists.txt — AutoCollect include moved earlier

**File:** `CMakeLists.txt`

Moved `include(AutoCollect)` from after `include(GroupSources)` to before `# Loading dyn modules`, so that module subdirectories are collected before the dyn module CMake code runs.

**Diff:** `+include(AutoCollect)` at line 68, removed from its previous position.

### 6. Submodules added

**File:** `.gitmodules`

| Submodule | URL |
|-----------|-----|
| `modules/mod-ale` | https://github.com/azerothcore/mod-ale.git |

### 7. env/dist — Removed .gitkeep files

Deleted `env/dist/.gitkeep`, `env/dist/etc/.gitkeep`, `env/dist/logs/.gitkeep`.

## Tracking

- Created: 2026-07-01
- Upstream base: `be01c9f` (AzerothCore master, Jul 2026)
- Last merge: 271 commits ahead

To see diff of all custom changes: `git diff be01c9f..HEAD -- src/ CMakeLists.txt .gitmodules`
