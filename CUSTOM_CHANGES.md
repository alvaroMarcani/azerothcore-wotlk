# Custom Changes â€” Wownerubian

Este documento lista todas las modificaciones custom sobre AzerothCore para el proyecto Wownerubian.

## Core Changes

### 7. Cross-Faction Trade â€” Same group/raid bypass

**File:** `src/server/game/Handlers/TradeHandler.cpp`

Allows cross-faction trading when both players are in the same party or raid group.
Previously the server rejected trade between Alliance and Horde with `TRADE_STATUS_WRONG_FACTION`
unless the initiator had `RBAC_PERM_ALLOW_TWO_SIDE_TRADE` (GM permission 51).

**Change:** Added `&& !_player->IsInSameRaidWith(pOther)` to the faction check condition.

## Core Changes

### 1. ChannelMgr â€” `SetChannel()` method

**Files:** `src/server/game/Chat/Channels/ChannelMgr.cpp`, `src/server/game/Chat/Channels/ChannelMgr.h`

Added public method `SetChannel(name, channel)` to manually register/override a `Channel` object in the channel manager. Used by `mod-world-chat` to inject the `World` channel.

**Diff:** +8 lines cpp, +1 line header.

### 2. Wintergrasp Portal â€” Attackers can teleport

**File:** `src/server/scripts/Northrend/zone_wintergrasp.cpp`

Removed the defender-only check from `spell_wintergrasp_portal`. Previously: `wintergrasp->GetDefenderTeam() != target->GetTeamId()` blocked attackers. Now: any player level 75+ can use the Dalaran â†’ Wintergrasp portal.

**Diff:** `- (wintergrasp->GetDefenderTeam() != target->GetTeamId())`

### 3. Valithria Dreamwalker â€” Wipe reset & respawn fixes

**File:** `src/server/scripts/Northrend/IcecrownCitadel/boss_valithria_dreamwalker.cpp`

Three fixes:
- ~~**SetBossState(DONE) on kill**~~ **REVERTED 2026-08-21.** A custom `_instance->SetBossState(DONE)` was added on 2026-07-30 (commit dffeb46, image v25) in `HealReceived()` under the false premise that upstream never sets DONE. **Upstream DOES set DONE**: `SpellHit` (on SPELL_DREAM_SLIP hit) â†’ `Unit::Kill(me, trigger)` â†’ trigger's `BossAI::_JustDied()` â†’ `SetBossState(DONE)` (`Unit.cpp` â†’ `ai->JustDied(killer)`). The custom line in `HealReceived()` blocked `EVENT_DREAM_SLIP` forever (UpdateAI's `!= IN_PROGRESS` guard) â†’ Dream Slip never cast â†’ SpellHit never ran â†’ **no loot chest** (the "no loot" bug reported 08-2026). Moving it into `EVENT_DREAM_SLIP` (commit 236e7ea, v28) was redundant: nothing in the DONE transition interrupts the pending Dream Slip cast or the chest flow (verified: no module hooks, `UpdateMinionState` no-op on DONE, movement interrupt is player-only). **Current state: 100% upstream for the success path.** The 7-day respawn concern is moot: DB `spawntimesecs` is now 300s and DONE bosses don't respawn anyway.
- **2026-08-24 merge (upstream #26682):** upstream ported TrinityCore's archmage summon groups â€” Risen Archmages are now SUMMONED via `SummonCreatureGroup` (DB `creature_summon_groups`) instead of permanent spawns, re-summoned on every Valithria Reset. Adopted as-is (AC es fuente de verdad); the "archmages gone after corpse decay" problem the loop below solved no longer exists under this model. The wipe/respawn custom block below is RETAINED (no interference): it still forces the 11s reset after a wipe; its `GetCreatureRespawnTimes()` loop now effectively only matches the two permanent Valithria spawns (aligned with her own 11s despawn).
- **Respawn Archmages after wipe:** Risen Archmages killed early corpse-decay and leave the world; the existing `CreatureWorker` can't reach them. New code iterates `map->GetCreatureRespawnTimes()` and forces respawn of `NPC_RISEN_ARCHMAGE` and `NPC_VALITHRIA_DREAMWALKER` at `GameTime + 11s`. *(Post-merge: solo aplica a los spawns permanentes de Valithria; ver nota de arriba.)*
- **DespawnOrUnsummon replacement:** Replaced `RemoveCorpse(false)` + `SetRespawnTime(11)` with `DespawnOrUnsummon(0ms, 11s)` which works even after corpse decay.
- **Wipe detection:** Custom `UpdateAI` checks if any player is alive during `IN_PROGRESS`; if all dead, calls `DoAction(ACTION_DEATH)` to reset the encounter.

**Diff:** +58 lines, new include `<GameTime.h>`. No custom code on the success/Dream Slip path.

### 4. Halls of Reflection â€” Boss immunity + evade fix

**Files:** `src/server/scripts/Northrend/FrozenHalls/HallsOfReflection/boss_falric.cpp`, `boss_marwyn.cpp`

**Archivo:** `instance_halls_of_reflection.cpp` â€” **REVERTIDO a codigo original.**

Two fixes in `DoAction(1)`:
1. **`me->SetImmuneToPC(false)` â†’ `me->SetImmuneToAll(false)`**: Reset() sets `SetImmuneToAll(true)` (both IMMUNE_TO_PC + IMMUNE_TO_NPC), but DoAction(1) only cleared IMMUNE_TO_PC.
2. **Added `me->ClearUnitState(UNIT_STATE_EVADE)`**: HandleWaveWipe calls EnterEvadeMode() which sets UNIT_STATE_EVADE. DoZoneInCombat() checks IsInEvadeMode() and returns immediately â€” boss never engages combat after wipe.

**Diff:** 4 lines (2 per boss). Instance script unchanged from upstream.

### 5. CMakeLists.txt â€” AutoCollect include moved earlier

**File:** `CMakeLists.txt`

Moved `include(AutoCollect)` from after `include(GroupSources)` to before `# Loading dyn modules`, so that module subdirectories are collected before the dyn module CMake code runs.

**Diff:** `+include(AutoCollect)` at line 68, removed from its previous position.

### 8. Nerubian World Boss â€” Class loot for ALL participants

**File:** `src/server/scripts/Custom/boss_nerubian_devastador.cpp`

Custom world boss script (entry 902006). Anti-kite, HP% phases (fear/whelps/enrage), dynamic HP/DMG scaling by player count. On death (`JustDied`), gives a random class-appropriate PvE armor piece to EVERY non-GM player on the threat list (not just one winner).

**Change (2026-07-25):** `GiveClassLoot()` changed from single random winner to all participants. Added yell announcement with player count.

### 9. Four New World Bosses â€” Weekly Schedule (Lun/Mie/Vie/Sab)

**Files:**
- `src/server/scripts/Custom/boss_nerubian_viuda_cristal.cpp` â€” La Viuda de Cristal (902020)
- `src/server/scripts/Custom/boss_nerubian_golem_runico.cpp` â€” Golem de Runa FÃ©rrea (902021)
- `src/server/scripts/Custom/boss_nerubian_profeta_sombrio.cpp` â€” Profeta SombrÃ­o (902022)
- `src/server/scripts/Custom/boss_nerubian_draco_tempestad.cpp` â€” Draco Tempestad (902023)

All four inherit from `WorldBossAI` with shared framework: `ApplyPlayerScaling()` (HP/DMG Ã— players/20, clamp 1Ã—â€“3Ã—), `GiveClassLoot()` (PvE class item to ALL non-GM participants), anti-kite (SummonPlayer if >50yd). Same loot structure: 100% Primordial Saronite 5-10 + Group 2 legendaries 6% + C++ class loot.

Each boss has 3 phases (75%/50%/25%) with one unique mechanic:
- **Viuda de Cristal (Lun, STV):** CristalizaciÃ³n de TelaraÃ±a â€” webs every 15s under non-tanks, slow + magic dmg reduction, 3+ active = boss heals 5% hp/s
- **Golem de Runa FÃ©rrea (MiÃ©, Burning Steppes):** Rune Forging â€” every 20s forges rune (Fire DOT / Ice immune / Shadow +50% phys / Storm +100% healing)
- **Profeta SombrÃ­o (Vie, Tirisfal):** Shadow Link â€” pairs players every 25s, >30yd pull+stun, <5yd shadow resist buff
- **Draco Tempestad (SÃ¡b, Winterspring):** Wind Shear â€” every 20s frontal 120Â° cone KB after 3s facing random direction

**Diff:** +4 new .cpp files (~11-13KB each), +20 lines to `custom_script_loader.cpp` (declarations + calls).

### 9b. Nerubian World Bosses â€” Class reward via retail bag (20367)

**Files:** all 5 `boss_nerubian_*.cpp` (ignored by `.gitignore`, local-only)

**Change (2026-07-30):** `GetClassBag()` removed; replaced with `static constexpr uint32 BAG_REWARD_ENTRY = 20367`. All participants now receive the same retail item 20367 (Hunting Gear, known to the 3.3.5a client â€” no MPQ patch needed, correct icon + right-click loot). The per-class filtering moved from C++ to DB: `item_loot_template` on 20367 has 10 groups (one per class, Chance=0 random between the 5 slots) + `conditions` (SourceTypeOrReferenceId=5, ConditionType=15 class masks). Loot generates per-player on open, conditions hide non-matching classes.

**Previous attempts (superseded):** GO chest 902051 (uninteractable), custom bags 970701-710 (unknown entries â†’ `?` icon, no MPQ). Removed custom bags from `item_template`.

**Diff:** ~20 lines changed per file (1 constant + `player->AddItem(BAG_REWARD_ENTRY, 1)`).

### 10. World Boss Rework â€” Respawn fix + new mechanics + bag gold (2026-07-31)

**Files:** all 5 `src/server/scripts/Custom/boss_nerubian_*.cpp`

Three-part change:

1. **Bag gold:** `GiveClassBag()` in all 5 scripts now also grants `player->ModifyMoney(urand(2,500,000, 5,000,000))` = 250-500g per participant (item loot templates cannot give gold). Image v26.
2. **Boss-specific mechanics:**
   - **Devastador (902006):** new berserk after 8 min (`SPELL_ENRAGE` 33653, one-shot event) + whelps also summoned at the 75% phase (was 50/25 only).
   - **Viuda (902020):** web-crystal heal raised 3% â†’ 5% hp/s; new `Volatile Infection` (24928, Emeriss-style stacking poison DoT) on the tank every 90-120s.
   - **Golem (902021):** new Lethon + Doomwalker mechanics â€” `Shadow Bolt Whirl` (24834) self-aura on engage (reuses registered `spell_shadow_bolt_whirl` spellscript), `Draw Spirit` (24811) at 50% summoning `NPC_SPIRIT_SHADE` (15261, AI from emerald dragons) at each hit target, and `Overrun` (32636) charge+knockback every 15-20s.
   - **Profeta (902022):** new `Lightning Wave` (24819, Ysondre chain lightning, re-themed "Onda del VacÃ­o") every 10-20s; void adds now summoned at all three phase transitions (75/50/25%) scaled to half the raid (min 1, max 15, Ysondre-style).
   - **Draco (902023):** new Taerar-style banish at 75/50/25% â€” summons 3 `Espiritu de Tormenta` (NPC 902024), boss becomes `UNIT_FLAG_NOT_SELECTABLE | UNIT_FLAG_NON_ATTACKABLE` + `REACT_PASSIVE` for **12s** (or until the 3 adds die); new `Arcane Blast` (24857) every 7-12s. Overrides `UpdateAI` to handle the banish state.
3. **SQL (applied via `2026_07_31_00_worldboss_rework.sql`):** respawn 15 min (`spawntimesecs` 900), game events 201/203/204/205/206 extended to 26h so last respawn never crosses the event window, Draco relocated from Felwood river-level to valid Winterspring ground, HP/damage bumps (Devastador 240/45, Golem 240, Profeta 240), new NPC 902024 + materials/Naxx loot rows.

**Diff:** ~10-40 lines per .cpp (largest: Draco +~75).

### 11. World Bosses ï¿½?" Shared base class + Dragon breath fix + CC immunities (2026-08-03)

**Files:**
- `src/server/scripts/Custom/boss_nerubian_shared.h` ï¿½?" NEW. Shared `NerubianWorldBossAI` (derives from `WorldBossAI`).
- all 5 `src/server/scripts/Custom/boss_nerubian_*.cpp` ï¿½?" refactored to inherit the shared base.

**1. Shared base class (`boss_nerubian_shared.h`):** centralizes the duplicated framework that every boss had:
- `ApplyPlayerScaling()` (HP/DMG x players/20, clamp 1xï¿½?"3x), `GiveClassLoot()` via retail bag 20367 + 250-500g, anti-kite (SummonPlayer 24776 > 50yd).
- `ApplyBossImmunities()` ï¿½?" NEW: `ApplySpellImmune(0, IMMUNITY_MECHANIC, ...)` for **STUN, FREEZE, KNOCKOUT, POLYMORPH, CHARM, SLEEP, FEAR, HORROR, BANISH, DISORIENTED, SAPPED**. All bosses are now stun-immune (bosses were stunnable before).
- Clean virtual hooks used by each script: `ResetBossState()`, `ScheduleBossEvents()`, `ExecuteBossEvent()`, `OnEngaged()`, `OnJustDied()`. Base `UpdateAI` handles victim check + anti-kite event; phase stage (`_stage`) shared.
- Refactor is pure duplication-removal; each boss keeps its own mechanics/timers/phase logic in `ExecuteBossEvent()`/`DamageTaken()`.

**2. Draco Frost Breath fix (28522 ï¿½?" 69527):** `SPELL_FROST_BREATH` was `28522` = **Icebolt** (Sapphiron's 25s ice-block stun), so every "breath" froze the entire raid for ~24s and bypassed stun/immunity. Now `69527` = proper dragon **Frost Breath** (frost damage + melee haste penalty, no stun).

**3. Draco banish 12s ï¿½?" 8s:** `_banishedTimer` 12000 ï¿½?" 8000 (phase adds phase at 75/50/25%).

**4. SQL buff (in `2026_07_31_01_worldboss_consolidated.sql`):** 4 bosses buffed (Draco kept as baseline reference):

| Boss | HealthMod | DamageMod |
|------|:--------:|:---------:|
| Devastador 902006 | 240 ï¿½?" 320 | 45 ï¿½?" 70 |
| Viuda 902020 | 200 ï¿½?" 280 | 40 ï¿½?" 65 |
| Golem 902021 | 240 ï¿½?" 320 | 40 ï¿½?" 65 |
| Profeta 902022 | 240 ï¿½?" 320 | 40 ï¿½?" 65 |

**Diff:** +1 new header (~90 lines), ~60-100 lines removed per .cpp (duplicated framework), Draco spell id + timer change.

### 6. Submodules added

**File:** `.gitmodules`

| Submodule | URL |
|-----------|-----|
| `modules/mod-ale` | https://github.com/azerothcore/mod-ale.git |

### 7. env/dist â€” Removed .gitkeep files

Deleted `env/dist/.gitkeep`, `env/dist/etc/.gitkeep`, `env/dist/logs/.gitkeep`.

### 12. GM additem â€” Self-only for level-3 GMs (2026-08-19)

**Files:** `src/server/game/Accounts/RBAC.h`, `src/server/scripts/Commands/cs_misc.cpp`

New RBAC permission `RBAC_PERM_COMMAND_ADDITEM_ANY_TARGET = 100000` (row `(100000, 'Command: additem any target')` must exist in `acore_auth.rbac_permissions`).

`HandleAddItemCommand` and `HandleAddItemSetCommand` now deny when the target player differs from the executing player UNLESS the session holds permission `RBAC_PERM_COMMAND_ADDITEM_ANY_TARGET` (granted only to gmlevel-4 accounts via role 100004). Console/SOAP sessions (no player) are unaffected.

This works together with the auth DB script `sql/custom/db_auth/2026_08_19_00_gm_level4_rbac.sql` which:
- sets gmlevel 4 as the top tier (default role 192, full admin),
- restricts gmlevel 3 to role 100003 (admin minus `.server*`, `.send items/mail/money`, additem-to-others),
- moves those dangerous commands + permission 100000 into level-4-only role 100004.

**Diff:** +1 line RBAC.h, +9 lines per handler in cs_misc.cpp (marked `// CUSTOM:`).

### 13. Console can set gmlevel 4 (2026-08-20)

**File:** `src/server/scripts/Commands/cs_account.cpp`

`HandleAccountSetGmLevelCommand` refuses any `gm >= playerSecurity`. Since the console runs with `playerSecurity = SEC_CONSOLE = 4`, it could never promote an account to the top tier (gmlevel 4). The console is now exempt from that check (same exemption pattern already used a few lines below for the `gmRealmID == -1` rank check): `if (!AccountMgr::IsConsoleAccount(playerSecurity) && (targetSecurity >= playerSecurity || gm >= playerSecurity))`.

In-game GM accounts keep the original restriction (can't set a level >= their own).

**Diff:** +1 condition line in cs_account.cpp (marked `// CUSTOM:`).

### 14. Upstream sync 2026-08-24 â€” adaptaciÃ³n de comandos GM (merge ca8e6d78)

Merge de upstream master (+621 commits desde be01c9f). Adaptaciones sobre los cambios upstream de comandos para preservar nuestro esquema (gmlevel 4 top tier / gmlevel 3 restringido / permiso 100000):

- **cs_misc.cpp â€” `.additem` con targets offline (#26714):** upstream eliminÃ³ el early-return de `playerTarget` para soportar remover items de jugadores offline vÃ­a DB. Nuestro guard RBAC 100000 se re-aplicÃ³ adaptado: comparaciÃ³n por **GUID** (`player->GetGUID() != executor->GetGUID()`) en vez de puntero, asÃ­ cubre tambiÃ©n targets offline. Consola/SOAP sigue exenta (sin sesiÃ³n). El guard de `.additemset` auto-fusionÃ³ sin cambios.
- **cs_account.cpp â€” fix realm -1 (#27088):** el fix upstream (bloqueo de escalada vÃ­a `gmRealmID == -1` cuando el target tiene rank mayor en otro realm) convive con nuestra exenciÃ³n de consola; ambos guards respetan `IsConsoleAccount`.
- **TradeHandler.cpp â€” clustering (#16832):** el rewrite de clustering mantuvo intacta la condiciÃ³n de facciÃ³n; nuestro bypass `IsInSameRaidWith` sobreviviÃ³ sin ediciÃ³n. Upstream aÃ±adiÃ³ encima la restricciÃ³n de trade para trial accounts (nueva feature, aceptada).
- **Valithria:** adoptado el modelo summon groups de upstream #26682 (ver secciÃ³n 3).
- **RBAC.h:** permiso 100000 conservado junto a los nuevos perms upstream (`ACCOUNT_FLAG_*`, `ACCOUNT_INFO`, etc.).

**Nuevos seeds RBAC upstream:** los comandos migrados a RBAC (#26607: autobroadcast/mail/npc/pool/spellinfo) requieren sus filas nuevas en `acore_auth.rbac_permissions`/`rbac_linked_permissions` â€” aplicar el update `pending_db_auth` correspondiente al habilitar el updater.

### 15. Upstream sync 2026-09-09 â€” merge 75ca8559 (+196 commits, ca8e6d78 â†’ 75ca8559)

Merge limpio: **ninguno de los 196 commits upstream tocÃ³ nuestros archivos custom** (TradeHandler, cs_account, cs_misc, RBAC.h, Valithria, Falric/Marwyn, Wintergrasp, ChannelHandler). Ãšnico conflicto: `AGENTS.md` (reescritura upstream, sin contenido custom nuestro) â€” resuelto tomando upstream.

Cambios relevantes para nosotros (solo contexto, sin adaptaciÃ³n requerida):
- **Ulduar (25 commits):** rework grande de vehÃ­culos de Flame Leviathan (respawn, lÃ­mite de 2 monturas, aggro, approach arc, escalado de gear por vehÃ­culo, tar despawn), Yogg-Saron (8 Guardians en fase 1, Constrictor Tentacle escape limitado, Grim Reprisal sin reflect a totems), Thorim CC pack + NPCs, Freya roots despawn en wipe, Kologarn hard reset, Formation Grounds teleporter usable en wipe. **Relevant QA: Ulduar sanity** tras este rework (ver pendiente QA in-game).
- **Core (32 commits):** port del rewrite de combat/threat (#27347 + fix #27347), hook `UnitAI::OnDespawn` (#27285), taxi flight speed configurable (#27183), fix de estado invÃ¡lido al polimorfar/miedo a un MC'ing priest (#27263), e2e suite de protocolo live (#27158).
- **DB (121 commits):** fixes de quests/spawns/SAI varios (LBRS patrols, Champion of Hodir Freezing Breath al 2Âº threat, Keleseth, Mord'rethar, CoT Stratholme specimens, The Hunter and the Prince restaurada, Gortok sonidos, Nishera patrulla, Decrepit Clefthoof despawn, Gnomeregan boss damage).

**Nota updates DB:** ~121 updates `db_world` nuevos (2026_08_25_00 â†’ 2026_09_09_00) â€” se aplicarÃ¡n con `AC_UPDATES_ENABLE_DATABASES=7` en el prÃ³ximo arranque del worldserver local/VPS. Ninguno interfiere con contenido custom (Nerubian Store, world bosses, etÃ©rea) â€” verificado por Ã¡rea: no hay updates tocando entries 900000+ ni 970xxx.

### 16. NPCBots (Trinity-Bots v5.4.659a) â€” core patch aplicado (2026-09-09)

**Fuente:** `https://github.com/trickerer/Trinity-Bots` (branch master, commit `1e75a30`, "NPCBots v5.4.659a"). NO es un mÃ³dulo: se aplica como **patch del core** (`Trinity-Bots/AC/NPCBots.patch`).

**AplicaciÃ³n:**
- `git apply --check` FALLA en 1 hunk (`Unit.cpp` `GetEffectiveResistChance` â€” upstream refactorizÃ³ esa zona el 05-09 Sep, despuÃ©s del base del patch `cabcfcaec7`). `patch -p1` (GNU patch, mÃ©todo oficial del README) aplica TODO limpio salvo ese hunk con `fuzz 2 (offset -2)`: el cÃ³digo NPCBot (resist/penetration) queda insertado en el punto correcto (lÃ­nea 2485).
- Archivos tocados que coinciden con customs nuestros: `RBAC.h` (aÃ±ade perms NPCBot 70001-70037 antes de `RBAC_PERM_MAX`; nuestro `100000` intacto en la lÃ­nea 321) y `zone_wintergrasp.cpp` (hunk en `CanControlVehicle`, NO en nuestro portal custom) â€” ambos sin conflicto.
- Nuestros customs NO tocados: `TradeHandler.cpp`, `cs_account.cpp`, `cs_misc.cpp`, `boss_valithria_dreamwalker.cpp`, `boss_falric/marwyn.cpp`, `ChannelHandler`.
- El patch ademÃ¡s: des-ignora `data/sql/custom/db_{auth,characters,world}` en `.gitignore` (los SQLs NPCBots se commitean) y aÃ±ade `src/server/game/AI/NpcBots/` (63 archivos).

**SQL (pipeline AC, sin duplicar):**
- `data/sql/base/db_characters/characters_npcbot*.sql` (4) + `data/sql/base/db_world/creature_template_npcbot_appearance/extras/outfits.sql` (3): tablas base. **Solo se auto-aplican en DBs vacías** (AutoSetup). Para DBs existentes se transformaron en el repo wownerubian (`sql/custom/`): sin DROP, `CREATE TABLE IF NOT EXISTS` + `INSERT IGNORE` — **prefijo `0000-00-00_` OBLIGATORIO** (los updates del patch hacen `ALTER TABLE` sobre las tablas base y fallan con 1146 si el base ordena después). Locale esES de bot texts (`data/sql/Bots/locales/esES/npc_text_locale.sql`) igualmente transformado (DELETE+INSERT, idempotente).
- `data/sql/custom/db_{auth,characters,world}/*.sql` (~75): se aplican solos vía `ac-db-import` (dbimport escanea `updates_include` → `$/data/sql/custom/db_*` estado CUSTOM, registra en `updates`). Verificado: 127 CUSTOM en acore_world, 138 queries aplicadas.
- **Trampa encontrada:** el bind-mount `./sql/custom/db_auth:/azerothcore/data/sql/custom/db_auth` del override REEMPLAZA el directorio completo → los 2 SQLs RBAC de NPCBots (auth) NO llegaban a la imagen. Fix: copiados a `sql/custom/db_auth/` del repo wownerubian (mount host) + re-run de ac-db-import.

**RBAC (adaptación a nuestro esquema, SQL `sql/custom/db_auth/2026_09_09_00_npcbot_rbac_roles.sql`):**
El patch linkea los perms NPCBot a los roles AC estándar 196 (GM4)/197 (GM3)/199 (player), que **no existen** en nuestro esquema post-refactor (gm3=100003, gm4=192, player=195). Re-linkeados:

| Rol nuestro | Perms NPCBot | Fuente |
|---|---|---|
| 192 (gm4) | 20 (admin: dump, createnew, add/spawn/kill…) | 196+197 |
| 100003 (gm3) | 20 (idem, sin dump/createnew…) | 196+197 |
| 195 (player) | 17 (player commands: base, info, hide, recall, follow, standstill…) | 199 |

Los roles 196/197/199 siguen existiendo con sus links originales (huérfanos, sin uso).

**Config (repo wownerubian `var/etc/worldserver.conf`):** sección `# NPCBOT CONFIGURATION` completa (847 líneas del `worldserver.conf.dist`) + `Appender.NpcBots=1,2,0` + `Logger.npcbots=2,NpcBots Server`. Cambios sobre defaults:
- `NpcBot.Enable.Raid = 1` (objetivo: compañeros para raids)
- `NpcBot.HideSpawns = 0` (bots libres spawneados siempre visibles/contratables)
- `NpcBot.Cost.Hire = 200000000` (20,000g base a lvl 80; escala por nivel <10=10g … 40-79=10-19.8kg; clases ex ×2/×5)
- `NpcBot.WanderingBots.Continents.Count = 30` (test con 316 confirmó máximo sin duplicar templates: 317 − contratados; ASSERT si desired > spare)

**Contenido custom adicional (repo wownerubian):** spawn del Botgiver Lagretta (70000, guid 903001, Tienda Etérea) + 13 nodos de wander urbanos (ids 10000-10014, SW/OG/Dalaran — coords validadas de NPCs existentes; los nodos del patch en Dalaran están bajo la plataforma, no reutilizados).

**Imagen:** `alvarexp7/ac-wotlk-worldserver:v33` (+latest). QA local: `NPCBots config loaded / system enabled`, tablas cargadas sin errores, `.npcbot` y `.npcbot lookup` responden por SOAP, `.reload rbac` OK, 1 bot contratado en QA. Memoria: 30 wanderers ≈ +260 MB; 316 ≈ +0.3-0.5 GB (grids compartidas, no lineal).

### 17. NPCBots ownership — lease 1h + rent + offline grace 10 min (2026-09-11, imagen v34)

**Motivo:** el owner quiere simultáneamente: (a) límite de tiempo de contratación (1h desde el hire), (b) alquiler recurrente por oro, (c) reset de owner si el jugador se desconecta >10 min. El stock solo soporta UNA base de tiempo (`NpcBot.OwnershipExpireMode`, 0=offline ó 1=hire, mutuamente excluyentes).

**Cambio C++ (`src/server/game/AI/NpcBots/`):**
- `botconfig.h/cpp`: nueva key `NpcBot.OwnershipOfflineExpireTime` (segundos, default 0=disabled) + getter `GetOwnershipOfflineExpireTime()`. Marcado `//CUSTOM`.
- `bot_ai.cpp` `CheckOwnerExpiry()`: se computa el `MAX(logout_time)` de la cuenta del owner una sola vez; deadline efectivo = `max(baseTimeStamp + OwnershipExpireTime, lastLogout + OwnershipOfflineExpireTime)` donde baseTimeStamp sigue la semántica del modo (0=logout, 1=hire_time). Es decir: el bot queda reservado hasta el deadline de propiedad (lease) Y al menos N segundos tras la última desconexión (lo que sea más tarde).
- `bot_ai.cpp` `CalculateOwnershipCheckTime()`: si hay offline grace configurado, la cadencia de poll pasa a `min(min(expireTime, offlineExpire), urand(3-7min))` para detectar el deadline offline con prontitud (el cálculo "exacto" de hire sólo aplica sin offline grace).
- `bot_ai.cpp` líneas 315/322/461: el timer de ownership se arma/desarma considerando AMBAS keys (`||`).

**Config (repo wownerubian `var/etc/worldserver.conf`):**
- `NpcBot.Cost.Rent = 2000000` (200g/h, cobrado cada 10 min ≈33.3g; si no puede pagar → bot despedido BOT_REMOVE_UNAFFORD, owner reset)
- `NpcBot.OwnershipExpireTime = 3600` + `NpcBot.OwnershipExpireMode = 1` (lease 1h desde el hire)
- `NpcBot.OwnershipOfflineExpireTime = 600` (CUSTOM: 10 min de gracia tras la última desconexión)

**Comportamiento resultante:**
- Jugador online con bot contratado: nunca expira por timer (el check solo corre si el bot está libre/detached) — el "límite" mientras juegas es el RENT.
- Al desconectar: el bot queda reservado hasta `max(hire+1h, logout+10min)`; si la sesión duró <50 min el lease (1h) es el que manda; si duró >50 min, se libera 10 min tras el logout.
- Al expirar: equipo devuelto por correo (subject "Bot ownership expired due to inactivity"), owner=0, shared owners limpios, spec/roles a default, sale del grupo. Coste de hire NO se reembolsa; el bot vuelve al pool de Lagretta.

**Imagen:** `alvarexp7/ac-wotlk-worldserver:v34` (+latest).

### 18. Traducción de módulos al español (2026-09-15, imagen v35)

**Motivo:** los módulos custom mostraban texto en inglés a los jugadores. Auditados los 8 módulos: `mod-transmog` (module_string_locale esES/esMX), `mod-autobalance` (Message.cpp esES/esMX), `mod-breaking-news-override` (HTML ya en español) y `mod-ale` (scripts Lua en español) YA tenían traducción. Traducidos los 4 restantes (hardcode en español, el servidor es esES-only):

- **mod-world-chat** (`src/WorldChat.cpp`): announce de login, mensajes `.chat` (disabled/muted/hidden/visible). ⚠️ No es submodule ni está trackeado en acore-src (gitignored) — los cambios viven solo en la imagen.
- **mod-anticheat** (submodule): avisos de ban/jail (`AnticheatMgr.cpp`), announce de login (`AnticheatScripts.cpp`), comando `.anticheat warn` + salidas GM (`cs_anticheat.cpp`).
- **mod-cfbg** (⚠️ gitignored, no submodule): mensajes de `.cfbg race` (`cs_cfbg.cpp`), anuncios de cola BG y "Has Joined" (`CFBG.cpp`).
- **mod-progression-system** (submodule): `.progression info` (`cs_progression_module.cpp`).
- **mod-autobalance**: fix menor — faltaba la clave `esMX` de `AB_LEAVING_INSTANCE_COMBAT` (con `ABGetLocaleText` retornando `""` si no existe, el mensaje salía vacío para clientes esMX).

**Imagen:** `alvarexp7/ac-wotlk-worldserver:v35` (+latest).

### 19. Nuevos módulos instalados (2026-09-15): guildhouse, auto-shutdown, boss-announcer, 1v1-arena, war-effort

**Instalados como submodules** (`.gitmodules` + gitlinks en acore-src) y traducidos al español. El SQL y las configs viven en el repo wownerubian (`sql/general/` + `var/etc/modules/` — ver AGENTS.md sección Per-Session 2026-09-15).

| Módulo | Submodule | SQL (repo) | Conf (repo) | Traducción |
|---|---|---|---|---|
| mod-guildhouse | ✅ | guildhouse ×5 (db_world) + ×1 (db_characters) | mod_guildhouse.conf | Textos por DB (`mod_guildhouse_locale`, esES completo) + 2 strings C++ a locale table |
| mod-server-auto-shutdown | ✅ | — | ServerAutoShutdown.conf | Mensaje de anuncio en config (español) |
| mod-boss-announcer | ✅ | — | mod_boss_announcer.conf | C++ (login announce, wipe, kill) |
| mod-1v1-arena | ✅ | 1v1_battlemaster.sql (NPC 999991 + npc_text 999992 esES) | 1v1arena.conf | C++ (comandos .q1v1, gossip, cola) |
| mod-war-effort | ✅ | war_effort ×3 (db_world) + ×1 (db_characters) | mod_aq_war_effort.conf | C++ (mensajes + comando .wareffort) |

**⚠️ NO instalado — mod-player-bot-guildhouse (DustinHendrickson):** requiere `mod-playerbots` (liyunfan1223) — `#include "PlayerbotMgr.h"`/`PlayerbotAI.h` — que NO tenemos (usamos NPCBots/Trinity-Bots, patch del core). Compilaría con error → eliminado del árbol.

**Verificación de compatibilidad con el core (75ca8559 + NPCBots):**
- mod-boss-announcer: hooks `OnUnitEnterEvadeMode(Unit*, uint8)`/`OnUnitEnterCombat` coinciden con `ScriptMgr.h`; `HasHealSpec`/`HasTankSpec`/`IsDungeonBoss` OK.
- mod-1v1-arena: `ArenaTeam::ArenaSlotByType`/`ArenaReqPlayersForType`, `BattlegroundMgr::queueToBg`/`ArenaTypeToQueue`/`QueueToArenaType`, hooks `PLAYERHOOK_ON_GET_MAX_PERSONAL_ARENA_RATING_REQUIREMENT`/`ON_GET_ARENA_TEAM_ID`/`NOT_SET_ARENA_TEAM_INFO_FIELD` — todos presentes.
- mod-guildhouse: `Guild::Member::IsRankNotLower`, `sMapMgr->FindMap`, gossip API estándar.
- mod-war-effort: TaskScheduler, PlayerScript `PLAYERHOOK_ON_PLAYER_COMPLETE_QUEST`, CharacterDatabase.
- ⚠️ **Los SQL de módulos NO se instalan en la imagen** (el Dockerfile solo copia `data/`; el DBUpdater busca `SourceDirectory/modules/...` que no existe en runtime) → se copiaron a `sql/general/` del repo wownerubian y los aplica ac-db-import (patrón establecido).
- ⚠️ mod-war-effort: su SQL borra criaturas base de AQ (`game_event_creature`/`creature` ids 15383-15758, guids 3115xxx) y crea spawns permanentes + guids 311600-311620. Sin colisión con guids custom (901xxx/902xxx/5301xxx). `ModWarEffort.Enable = 0` por defecto (desactivado).

## Tracking

- Created: 2026-07-01
- Upstream base: `be01c9f` (AzerothCore master, Jul 2026)
- Last merge: 2026-09-09, sync completo hasta `75ca8559` (+196 commits desde ca8e6d78)
- NPCBots: Trinity-Bots v5.4.659a (`Trinity-Bots` repo local, gitignored; patch `Trinity-Bots/AC/NPCBots.patch`), aplicado 2026-09-09 sobre `75ca8559`+customs (commit `0433b617`)

To see diff of all custom changes: `git diff be01c9f..HEAD -- src/ CMakeLists.txt .gitmodules`

