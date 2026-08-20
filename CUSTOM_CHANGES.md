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

### 3. Valithria Dreamwalker — Wipe reset, DONE state & respawn fixes

**File:** `src/server/scripts/Northrend/IcecrownCitadel/boss_valithria_dreamwalker.cpp`

Four fixes:
- **SetBossState(DONE) on kill:** Boss state was never set to DONE after a successful encounter. It is now set in the `EVENT_DREAM_SLIP` handler in `UpdateAI` (initially it was set in `HealReceived()` at 100% health, but that blocked `EVENT_DREAM_SLIP` from ever executing because `UpdateAI` returns early unless the state is IN_PROGRESS — the Dream Slip spell never landed, so the Lich King controller never cast SPELL_SPAWN_CHEST and no loot chest spawned; that is why Valithria appeared to drop no loot). Without the DONE state, the state stayed IN_PROGRESS; on re-entry `ReadSaveDataBossStates` converted IN_PROGRESS → NOT_STARTED, but Valithria's respawn timer was set to 7 days (spawntimesecs) → invisible until instance reset.
- **Respawn Archmages after wipe:** Risen Archmages killed early corpse-decay and leave the world; the existing `CreatureWorker` can't reach them. New code iterates `map->GetCreatureRespawnTimes()` and forces respawn of `NPC_RISEN_ARCHMAGE` and `NPC_VALITHRIA_DREAMWALKER` at `GameTime + 11s`.
- **DespawnOrUnsummon replacement:** Replaced `RemoveCorpse(false)` + `SetRespawnTime(11)` with `DespawnOrUnsummon(0ms, 11s)` which works even after corpse decay.
- **Wipe detection:** Custom `UpdateAI` checks if any player is alive during `IN_PROGRESS`; if all dead, calls `DoAction(ACTION_DEATH)` to reset the encounter.

**Diff:** +58 lines, new include `<GameTime.h>`.

### 4. Halls of Reflection — Boss immunity + evade fix

**Files:** `src/server/scripts/Northrend/FrozenHalls/HallsOfReflection/boss_falric.cpp`, `boss_marwyn.cpp`

**Archivo:** `instance_halls_of_reflection.cpp` — **REVERTIDO a codigo original.**

Two fixes in `DoAction(1)`:
1. **`me->SetImmuneToPC(false)` → `me->SetImmuneToAll(false)`**: Reset() sets `SetImmuneToAll(true)` (both IMMUNE_TO_PC + IMMUNE_TO_NPC), but DoAction(1) only cleared IMMUNE_TO_PC.
2. **Added `me->ClearUnitState(UNIT_STATE_EVADE)`**: HandleWaveWipe calls EnterEvadeMode() which sets UNIT_STATE_EVADE. DoZoneInCombat() checks IsInEvadeMode() and returns immediately — boss never engages combat after wipe.

**Diff:** 4 lines (2 per boss). Instance script unchanged from upstream.

### 5. CMakeLists.txt — AutoCollect include moved earlier

**File:** `CMakeLists.txt`

Moved `include(AutoCollect)` from after `include(GroupSources)` to before `# Loading dyn modules`, so that module subdirectories are collected before the dyn module CMake code runs.

**Diff:** `+include(AutoCollect)` at line 68, removed from its previous position.

### 8. Nerubian World Boss — Class loot for ALL participants

**File:** `src/server/scripts/Custom/boss_nerubian_devastador.cpp`

Custom world boss script (entry 902006). Anti-kite, HP% phases (fear/whelps/enrage), dynamic HP/DMG scaling by player count. On death (`JustDied`), gives a random class-appropriate PvE armor piece to EVERY non-GM player on the threat list (not just one winner).

**Change (2026-07-25):** `GiveClassLoot()` changed from single random winner to all participants. Added yell announcement with player count.

### 9. Four New World Bosses — Weekly Schedule (Lun/Mie/Vie/Sab)

**Files:**
- `src/server/scripts/Custom/boss_nerubian_viuda_cristal.cpp` — La Viuda de Cristal (902020)
- `src/server/scripts/Custom/boss_nerubian_golem_runico.cpp` — Golem de Runa Férrea (902021)
- `src/server/scripts/Custom/boss_nerubian_profeta_sombrio.cpp` — Profeta Sombrío (902022)
- `src/server/scripts/Custom/boss_nerubian_draco_tempestad.cpp` — Draco Tempestad (902023)

All four inherit from `WorldBossAI` with shared framework: `ApplyPlayerScaling()` (HP/DMG × players/20, clamp 1×–3×), `GiveClassLoot()` (PvE class item to ALL non-GM participants), anti-kite (SummonPlayer if >50yd). Same loot structure: 100% Primordial Saronite 5-10 + Group 2 legendaries 6% + C++ class loot.

Each boss has 3 phases (75%/50%/25%) with one unique mechanic:
- **Viuda de Cristal (Lun, STV):** Cristalización de Telaraña — webs every 15s under non-tanks, slow + magic dmg reduction, 3+ active = boss heals 5% hp/s
- **Golem de Runa Férrea (Mié, Burning Steppes):** Rune Forging — every 20s forges rune (Fire DOT / Ice immune / Shadow +50% phys / Storm +100% healing)
- **Profeta Sombrío (Vie, Tirisfal):** Shadow Link — pairs players every 25s, >30yd pull+stun, <5yd shadow resist buff
- **Draco Tempestad (Sáb, Winterspring):** Wind Shear — every 20s frontal 120° cone KB after 3s facing random direction

**Diff:** +4 new .cpp files (~11-13KB each), +20 lines to `custom_script_loader.cpp` (declarations + calls).

### 9b. Nerubian World Bosses — Class reward via retail bag (20367)

**Files:** all 5 `boss_nerubian_*.cpp` (ignored by `.gitignore`, local-only)

**Change (2026-07-30):** `GetClassBag()` removed; replaced with `static constexpr uint32 BAG_REWARD_ENTRY = 20367`. All participants now receive the same retail item 20367 (Hunting Gear, known to the 3.3.5a client — no MPQ patch needed, correct icon + right-click loot). The per-class filtering moved from C++ to DB: `item_loot_template` on 20367 has 10 groups (one per class, Chance=0 random between the 5 slots) + `conditions` (SourceTypeOrReferenceId=5, ConditionType=15 class masks). Loot generates per-player on open, conditions hide non-matching classes.

**Previous attempts (superseded):** GO chest 902051 (uninteractable), custom bags 970701-710 (unknown entries → `?` icon, no MPQ). Removed custom bags from `item_template`.

**Diff:** ~20 lines changed per file (1 constant + `player->AddItem(BAG_REWARD_ENTRY, 1)`).

### 10. World Boss Rework — Respawn fix + new mechanics + bag gold (2026-07-31)

**Files:** all 5 `src/server/scripts/Custom/boss_nerubian_*.cpp`

Three-part change:

1. **Bag gold:** `GiveClassBag()` in all 5 scripts now also grants `player->ModifyMoney(urand(2,500,000, 5,000,000))` = 250-500g per participant (item loot templates cannot give gold). Image v26.
2. **Boss-specific mechanics:**
   - **Devastador (902006):** new berserk after 8 min (`SPELL_ENRAGE` 33653, one-shot event) + whelps also summoned at the 75% phase (was 50/25 only).
   - **Viuda (902020):** web-crystal heal raised 3% → 5% hp/s; new `Volatile Infection` (24928, Emeriss-style stacking poison DoT) on the tank every 90-120s.
   - **Golem (902021):** new Lethon + Doomwalker mechanics — `Shadow Bolt Whirl` (24834) self-aura on engage (reuses registered `spell_shadow_bolt_whirl` spellscript), `Draw Spirit` (24811) at 50% summoning `NPC_SPIRIT_SHADE` (15261, AI from emerald dragons) at each hit target, and `Overrun` (32636) charge+knockback every 15-20s.
   - **Profeta (902022):** new `Lightning Wave` (24819, Ysondre chain lightning, re-themed "Onda del Vacío") every 10-20s; void adds now summoned at all three phase transitions (75/50/25%) scaled to half the raid (min 1, max 15, Ysondre-style).
   - **Draco (902023):** new Taerar-style banish at 75/50/25% — summons 3 `Espiritu de Tormenta` (NPC 902024), boss becomes `UNIT_FLAG_NOT_SELECTABLE | UNIT_FLAG_NON_ATTACKABLE` + `REACT_PASSIVE` for **12s** (or until the 3 adds die); new `Arcane Blast` (24857) every 7-12s. Overrides `UpdateAI` to handle the banish state.
3. **SQL (applied via `2026_07_31_00_worldboss_rework.sql`):** respawn 15 min (`spawntimesecs` 900), game events 201/203/204/205/206 extended to 26h so last respawn never crosses the event window, Draco relocated from Felwood river-level to valid Winterspring ground, HP/damage bumps (Devastador 240/45, Golem 240, Profeta 240), new NPC 902024 + materials/Naxx loot rows.

**Diff:** ~10-40 lines per .cpp (largest: Draco +~75).

### 11. World Bosses �?" Shared base class + Dragon breath fix + CC immunities (2026-08-03)

**Files:**
- `src/server/scripts/Custom/boss_nerubian_shared.h` �?" NEW. Shared `NerubianWorldBossAI` (derives from `WorldBossAI`).
- all 5 `src/server/scripts/Custom/boss_nerubian_*.cpp` �?" refactored to inherit the shared base.

**1. Shared base class (`boss_nerubian_shared.h`):** centralizes the duplicated framework that every boss had:
- `ApplyPlayerScaling()` (HP/DMG x players/20, clamp 1x�?"3x), `GiveClassLoot()` via retail bag 20367 + 250-500g, anti-kite (SummonPlayer 24776 > 50yd).
- `ApplyBossImmunities()` �?" NEW: `ApplySpellImmune(0, IMMUNITY_MECHANIC, ...)` for **STUN, FREEZE, KNOCKOUT, POLYMORPH, CHARM, SLEEP, FEAR, HORROR, BANISH, DISORIENTED, SAPPED**. All bosses are now stun-immune (bosses were stunnable before).
- Clean virtual hooks used by each script: `ResetBossState()`, `ScheduleBossEvents()`, `ExecuteBossEvent()`, `OnEngaged()`, `OnJustDied()`. Base `UpdateAI` handles victim check + anti-kite event; phase stage (`_stage`) shared.
- Refactor is pure duplication-removal; each boss keeps its own mechanics/timers/phase logic in `ExecuteBossEvent()`/`DamageTaken()`.

**2. Draco Frost Breath fix (28522 �?" 69527):** `SPELL_FROST_BREATH` was `28522` = **Icebolt** (Sapphiron's 25s ice-block stun), so every "breath" froze the entire raid for ~24s and bypassed stun/immunity. Now `69527` = proper dragon **Frost Breath** (frost damage + melee haste penalty, no stun).

**3. Draco banish 12s �?" 8s:** `_banishedTimer` 12000 �?" 8000 (phase adds phase at 75/50/25%).

**4. SQL buff (in `2026_07_31_01_worldboss_consolidated.sql`):** 4 bosses buffed (Draco kept as baseline reference):

| Boss | HealthMod | DamageMod |
|------|:--------:|:---------:|
| Devastador 902006 | 240 �?" 320 | 45 �?" 70 |
| Viuda 902020 | 200 �?" 280 | 40 �?" 65 |
| Golem 902021 | 240 �?" 320 | 40 �?" 65 |
| Profeta 902022 | 240 �?" 320 | 40 �?" 65 |

**Diff:** +1 new header (~90 lines), ~60-100 lines removed per .cpp (duplicated framework), Draco spell id + timer change.

### 6. Submodules added

**File:** `.gitmodules`

| Submodule | URL |
|-----------|-----|
| `modules/mod-ale` | https://github.com/azerothcore/mod-ale.git |

### 7. env/dist — Removed .gitkeep files

Deleted `env/dist/.gitkeep`, `env/dist/etc/.gitkeep`, `env/dist/logs/.gitkeep`.

### 12. GM additem — Self-only for level-3 GMs (2026-08-19)

**Files:** `src/server/game/Accounts/RBAC.h`, `src/server/scripts/Commands/cs_misc.cpp`

New RBAC permission `RBAC_PERM_COMMAND_ADDITEM_ANY_TARGET = 100000` (row `(100000, 'Command: additem any target')` must exist in `acore_auth.rbac_permissions`).

`HandleAddItemCommand` and `HandleAddItemSetCommand` now deny when the target player differs from the executing player UNLESS the session holds permission `RBAC_PERM_COMMAND_ADDITEM_ANY_TARGET` (granted only to gmlevel-4 accounts via role 100004). Console/SOAP sessions (no player) are unaffected.

This works together with the auth DB script `sql/custom/db_auth/2026_08_19_00_gm_level4_rbac.sql` which:
- sets gmlevel 4 as the top tier (default role 192, full admin),
- restricts gmlevel 3 to role 100003 (admin minus `.server*`, `.send items/mail/money`, additem-to-others),
- moves those dangerous commands + permission 100000 into level-4-only role 100004.

**Diff:** +1 line RBAC.h, +9 lines per handler in cs_misc.cpp (marked `// CUSTOM:`).

## Tracking

- Created: 2026-07-01
- Upstream base: `be01c9f` (AzerothCore master, Jul 2026)
- Last merge: 271 commits ahead

To see diff of all custom changes: `git diff be01c9f..HEAD -- src/ CMakeLists.txt .gitmodules`
