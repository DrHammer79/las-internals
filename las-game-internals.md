# The Legend of Zelda: Link's Awakening (Switch) - game internals map

A lookup map of what the LAS Recomp co-op fork (a static recompilation of the game; `DrHammer79/LASRecomp`) has
reverse-engineered about the game, for people writing patches and mods. Terse by design: addresses, offsets, what
they are, and where the fact came from so it can be checked.

## 1. Conventions and sources

**Game and versions.** One main program serves two releases; the fork keys its address tables by the main
program's build ID (`runtime/game_addresses/versions.h`):

| Table | Releases | main build ID (exefs/main 0x40..0x53) |
|---|---|---|
| `v1_0_1` | 1.0.1 and 1.1.0 (same main, same romfs; only main.npdm differs) | `909E904AF78AC1B8DEEFE97AB2CCDB51968F0EC7` |
| `v1_0_0` | 1.0.0 | `AE16F71E002AF8CB059A9A74C4D90F34BA984892` |

Everything here is for **1.0.1 / 1.1.0** unless marked otherwise.

**Addresses.** The main module is loaded at `0x7000000000`, so `main+0xNNNN` = `0x7000000000 + 0xNNNN`
(e.g. `main+0xd7e780` = `0x7000d7e780`). The sdk module is at `0x7004000000`. This document writes full values;
the repo's comments mostly write `main+0x...`. Code in main ends at main+0x16a0000 (`MAIN_CODE_SIZE`); vtables and
rodata lie above that; the writable globals named here sit at `0x7001c9....`-`0x7001cc....`.

**1.0.0 vs 1.0.1.** The `v1_0_0` table has the same names; the 1.0.0 column in the generated tables says `same`,
the delta, or `not mapped` (0x0: the feature is off on 1.0.0 - every `COOP_*` site and the co-op runtime entries).
Observed pattern, from comparing the two tables: data/rodata and code up to at least main+0xded9f0 are identical
(main+0xdf8260 too, per save-format.md);
code from about main+0xe443fc (1.0.0) up to at least main+0x14311f0 is **+0xd0 in 1.0.1**. The boundary lies
somewhere between those two points.

**Important about the repo's comments.** Many non-co-op comments in `tools/patches.py` and `runtime/hle/*.c` were
written against **1.0.0** and quote 1.0.0 offsets (inferred by matching them against both tables; e.g. "the HUD update main+0xeaa800" is 1.0.0; the 1.0.1 site
is `0x7000eaa8d0`). Where such an address is in the shifted range and is not itself a table entry, this document
gives the 1.0.0 value from the comment and an `(inferred)` 1.0.1 value at +0xd0. Comments in the co-op code
(`runtime/hle/coop.c`, `runtime/hle/coop/*.inc`, `docs/coop.md`) and in the hookshot investigation were written
against 1.0.1.

**Sources** (repo-relative paths in the fork):

| Source | What it holds |
|---|---|
| `runtime/game_addresses/runtime_v1_0_1.h`, `patches_v1_0_1.h` (+ `_v1_0_0.h`) | Every named address (X-macro `GA_ADDR(NAME, value)`) |
| `runtime/game_addresses/coop_player_reads_v1_0_1.txt` | All 696 `ldr xT, [xN, #0x4b8]` in main (reads of the game manager's player) |
| `tools/patches.py` | Each patch / hook site, its expected instruction word, and what it is for |
| `runtime/hle/hooks.c` | Hook dispatch: return 0 = run the instruction, 1 = the hook did it, 2.. = jump to the site's targets |
| `runtime/hle/coop.c`, `runtime/hle/coop/*.inc` | Co-op: Link, actors, targets, DoAction, cameras, field screens |
| `runtime/hle/*.c` | warp, chamber, hookshot, doorjump, digs, shell_spots, apple_trees, miniboss, souls, pets, qol, ... |
| `runtime/nvn/weather.c`, `pace.c`; `runtime/ui/*.mm` | Link's hit filter call, frame controller, HUD / pause-menu panes, save editor |
| `docs/coop.md`, `docs/save-format.md`, `docs/debug-hooks.md` | Co-op internals, the save layout (1.0.0-mapped), test switches |
| `hookshot-drop-in-investigation.md` (distributed beside this file) | Actor spawn / activation / collision contact chain |

Marks: `(inferred)` = not stated by a source, derived here. `(1.0.0)` = the source quotes a 1.0.0 offset.

## 2. Core singletons and globals

### 2.1 Game manager - `[0x7001c97188]` (`GAME_MGR`)

A pointer global; dereference it to get the manager.

| Offset | What | Source |
|---|---|---|
| +0x498 | Container actors are registered in by name (`main+0xfb3360` registers; the pointer never changes) | investigation |
| +0x4a8 | The player entity (coop.c, warp.c) / "the current scene (weak)" (digs.c: handed to `SCENE_LOCK`). Chain: entity +0x88 = its object (via `0x7000011b70`); object +0x618 = player component; object +0x660 arrival point, +0x680 Link's spot on safe ground (snapped to 0.75), +0x690 facing, +0x694 "spot set" | coop.md, warp.c, digs.c |
| +0x4b8 | **The player: Link's actor.** Null on boot / title / file select; non-null = in play (`game_in_play`). "Link" to all game code (696 loads of it in main, 531 with the manager as base) | coop.md, warp.c |
| +0x4c0 | +0x4b8's shared_ptr control block (`COOP_SET_PLAYER` assigns both, counted) | coop.c |
| +0x4e0 | Frame controller: extra world steps this frame (30 fps mode = 2 steps a frame; counts the extra one down) | nvn/pace.c |
| +0x4e1 | Present interval - 1 (to nvnWindowSetPresentInterval) | nvn/pace.c |
| +0x4e2 | Least number of extra vsyncs the frame controller may choose (normally 0) | nvn/pace.c |
| +0x4e3 | Most extra vsyncs (1 = a 30 fps mode exists; 0 = none) | nvn/pace.c |

Setter: `0x70008387c0` (`COOP_SET_PLAYER`): x0 manager, x1 &shared_ptr -> +0x4b8 / +0x4c0.

Restart point: Link's update (`0x7000d355c0`) looks up the entity (`ldr x1, [x20, #0x4a8]` at `0x7000d35778`, then
`0x7000011b70`), `cbz` at `0x7000d35788` for no object, and standing on safe ground writes +0x680 / +0x690 / +0x694.
The save copies it to file +0x559c (`0x7000dfc710` -> `0x7000dfcb90`: +0x680 when +0x694 is set, else +0x660). (coop.md)

Level-jump processor's player component check: entity [gm+0x4a8] -> [+0x88] -> [+0x618] (type-checked shared_ptrs
via `0x700001b730` / `0x7000011b70`), then `0x7000864d50` is called with it unchecked (null+0x3f8 crash, return
address `0x700081bdc8`) if a jump is handed over mid level change. (warp.c)

### 2.2 PlayerStatus - `[0x7001cc1368]` (`PLAYER_STATUS_PTR`)

Pointer to the live save's PlayerStatus block (file offset 0x54C0; `save-format.md`, `ui/difficulty.mm`:
"PlayerStatus (save+0x54C0) + 0x120"). So PlayerStatus+N = live save + 0x54C0 + N.

| Offset | Save | Type | What | Source |
|---|---|---|---|---|
| +0x00 | 0x54C0 | char[0x40] | Current level name ("Field", "Lv01TailCave", "PanelAdventure"...) | doorjump.c, many |
| +0x40 | 0x5500 | char[0x40] | Current screen / room ("11C" on the Field) | tracker_overlay.mm, save-format |
| +0x80 | 0x5540 | u32 | Rupees (0-999) | save-format (inferred offset) |
| +0x84 | 0x5544 | s16 | **Health**, quarter hearts | patches.py, coop |
| +0x88 | 0x5548 | u64 | Secret Seashells found, a bit per index (0-49) | shell_spots.c |
| +0x90 | 0x5550 | u64 | Heart pieces, a bit per piece (0-31) | save-format (inferred offset) |
| +0x98 | 0x5558 | u16 | Heart containers, bits 0-8 | save-format (inferred offset) |
| +0x9a | 0x555A | u8 | Trade item (0 none .. 0xE) | save-format (inferred offset) |
| +0x9b | 0x555B | u8 | Ocarina songs (bits) | save-format (inferred offset) |
| +0x9c | 0x555C | u8 | Upgrades; tunics 0x40 red, 0x80 blue. Negative (bit 7) halves enemy hits | patches.py, qol.c |
| +0x9d | 0x555D | u8 | Companion: 0 none, 1 BowWow, 2 Marin, 3 Ghost, 4 Flying Rooster | pets.c |
| +0x9e / +0x9f / +0xa0 | 0x555E-0x5560 | u8 | Bombs / arrows / magic powder | save-format (inferred offset) |
| +0xa2 | 0x5562 | u8 | Item slot on **Y** (0xFF empty) | patches.py, escape.c |
| +0xa3 | 0x5563 | u8 | Item slot on **X** | patches.py, escape.c |
| +0xa4 | | u8 | Kill counter ("PowerUp" = Piece of Power when past a threshold; saturates at 255) | qol.c |
| +0xa5 | | u8 | Kill counter (>= 14: "DefenceUp" = Guardian Acorn) | qol.c |
| +0xa8 | 0x5568 | u32 | Inventory bits (bit n = item slot n: 0 sword .. 13/14/15 bottles 1/2/3) | dpad_items.c, escape.c |
| +0xa9 | 0x5569 | u8 | Bit 5 = first bottle owned (slot 0xd), bit 7 = third bottle owned (slot 0xf) | patches.py |
| +0xac | 0x556C | u32 | Key items / instruments (bit 0 Flippers, bits 7-14 instruments). patches.py: Moblin Chief's init deletes him unless "+0xac bit 7" (a story bit) is set; save-format lists bit 7 as the Full Moon Cello | patches.py, warp.c, save-format |
| +0xae | 0x556E | u8 | Seashell Sensor (bit 0) | save-format (inferred offset) |
| +0xc0 | 0x5580 | u8 | Bottle contents: bit n = bottle n holds something (empty = clear) | patches.py |
| +0xc2 | 0x5582 | u8 | Golden Leaves, bits 0-4 | save-format (inferred offset) |
| +0x120 | 0x55E0 | u8 | Hero Mode (== 1: damage << 1) | patches.py, difficulty.mm |

Kill counters are raised by shared death code `0x70007c6950`; the ItemDrop roll (`0x700089b960`) checks them for rows
with mAllowBuffReplace (+0x38). (qol.c)

### 2.3 Live save, flags

| Address | What |
|---|---|
| `0x7001cbb220` (`LIVE_SAVE`) | Live save buffer, 0x5EE0 bytes, byte-identical to `savedataN.bin`. Save manager at `0x7001cbb200`, buffer at its +0x20 (save-format, 1.0.0; same address in both tables) |
| `0x7001cc1388` (`FLAG_PTR`) | Pointer the flag object uses to reach the GlobalFlags bitfield (normally live save +0x5DC0, 256 bytes = 2048 flags; vanilla uses <= 1118) |
| `0x7001cc1390` (`FLAG_CAP`) | u32 byte capacity: every flag get / set is bounds-checked against it. Pointing FLAG_PTR at a larger buffer and raising FLAG_CAP extends the flags (flags_ext.c does 5632 bytes) |

Save layout summary (`docs/save-format.md`, mapped on 1.0.0, believed identical on 1.0.1):

| Range | What |
|---|---|
| 0x0000-0x00DF | Header (file-select summary; version u32 0xE at 0, header CRC at 4, name 0x20, health 0x32, max 0x34, level name 0x38, rupees 0x7C...) |
| 0x00E0 + 0x10*i | Dungeon records: dungeon n -> record 2(n-1), Color Dungeon 18; +0 small keys, +1 bits (0 compass, 1 map, 2 beak, 3 Nightmare Key) |
| 0x0220-0x481F | Room records, 0xE0 each |
| 0x4820-0x521F | 0x20-byte records |
| 0x5380-0x54BF | Profile: death count 0x5384, name 0x5388 (UTF-16, 9 units) |
| 0x54C0-0x561F | PlayerStatus (above). Restart point copied to 0x559C |
| 0x5DC0-0x5EBF | GlobalFlags (flag n = bit n) |
| 0x5EC4 | Body CRC |

CRCs: reflected CRC32 table (poly 0xEDB88320), **initial value 0xEDB88320, no final XOR**. Header CRC (0x04) covers
0x0000-0x00DF with 0x04-0x07 zeroed; body CRC (0x5EC4) covers 0x00E0-0x5EDF with 0x5EC4-0x5EC7 zeroed. Max health is
not stored: `12 + 4 * (popcount(u16 @0x5558) + popcount(u64 @0x5550) / 4)` (`0x7000df8260`, `PLAYER_MAX_HEALTH`).
Save code (1.0.0 offsets, `save-format.md`; same in 1.0.1 not confirmed): new game `0x7000dfab90` (body defaults
`0x7000df9890`), checksums `0x7000dfad00`, save `0x7000dfcc60` (header summary `0x7000dfc640`).

### 2.4 Other singletons

| Address | What | Source |
|---|---|---|
| `[0x7001cc6d98]` | Actor manager singleton (`0x7000fab540` hands it new actors) | investigation, coop.md |
| `0x7001cb2d00` (`ACTOR_REGISTRY`) | -> level's placed actors by key (unordered_map; first node +0x10; node +0 next, +0x10 key, +0x18 shared_ptr<Actor>) | pets.c |
| `0x7001bcc898` (`ACTOR_TYPES`) | Actor type table, 0x58 bytes a type: +0 create fn, +0x50 name. Some ids have no entry | coop/targets.inc |
| `[0x7001cb9fb8]` (`CHAMBER_MGR`) | Chamber Dungeon manager (section 7.2) | chamber.c |
| `[0x7001cb9e50]` (`PANEL_CATALOGUE`) | MapPieceList catalogue | chamber.c |
| `[0x7001cb24d8]` (`EVENT_MGR`) | Event manager | hint_sheet.c |
| `[0x7001cc6db8]` (`EVENT_RES_MGR`) | Event resource manager | hint_sheet.c |
| `0x7001cab2d0` (`STR_POOL`) | String pool for type-5 spawn parameters (vector of 32-byte {std::string, refs}; the game never fills it) | digs.c |
| `0x7001cacc68` / `78` / `88` | DoAction current / pending (weak_ptrs) / cooldown (float) | coop/events.inc |
| `0x7001cbae80` | Dug-hole list, count at +0x10 (oldest filled in beyond 80, `0x7000d22d10`) | patches.py |
| `[0x7004cf9e38]` (`SDK_SYSTEM_OBJECT`, sdk) | +0x680 -> current camera zone record: +0 u32 id, +0x10 min vec3, +0x20 max vec3 (none on the Field). Thin in z (< 6) = side-view room | widescreen.c |
| `0x7001701e40` | Table by level kind: box of screens whose objects are made (Field: 75 x 60) | coop.md |
| `0x7001701ed4` | Table by screen type: box of active screens | coop.md |

Global strings / data: section 8.1 tables (`STRING_*`, `EMPTY_STRING`, `ASPECT_LITERAL`); `0x70016cbd73` =
"PFXTiltShiftParam" (qol.c).

## 3. Actors

### 3.1 Common actor layout

| Offset | What | Source |
|---|---|---|
| +0x00 | vtable (class identity) | |
| +0x08 | Pointer to itself while live (enable_shared_from_this weak pointer; a live-actor check) | weather.c, hearts.inc |
| +0x18 | Name, inline. Level actors: `<Type>-<key hex>` ("EnemyOctorok-...", "ObjRock-61025C005BA1C308"); level objects are often cut short ("ObjDungeonCo-99000F005B920361" is an ObjDungeonCollapseTileLv05). Components too: "PlayerCollision", "Zombie:ReactionComp", "<kind>:TailorPlayerComp" | coop.md, doorjump.c |
| +0x40 | Flags (3.2) | |
| +0x41 | Byte the co-op mod sets to 1 on extra Links at a door walk-in (meaning unknown) | coop/spawn.inc |
| +0x43 | Player actors: bit 6 (0x40) set = not a player that triggers / is taken along by doors, stairs, warp holes (those keep a touching actor with it clear) | patches.py |
| +0x48 | Actors: their room (shared_ptr; +0x50 its control block - spawned items go there). **Components: their owner actor** | apple_trees.c, coop.md |
| +0x80 | Main component; for most actors the transform / collision component, position vec3 at comp+0x80. Not every actor has one | coop/targets.inc |
| +0x98 | Spawn key (u64, from the spawn info) | coop.md |
| +0xa0 | Actor type id (u16, from the spawn info) | coop.md |
| +0x398 | Name copy in item actors ("ItemHeartPiece-<id>"; may not be filled when it first asks to appear) | qol.c, miniboss.c |

### 3.2 Flags at +0x40

| Bits | Meaning | Source |
|---|---|---|
| 0x2 (bit 1) | Active: its screen is in the active box (ground solid, objects running) | coop.md |
| 0x8 | Set by the in-play spawn path before the actor is handed to the manager | investigation |
| 0x10 | Set by the registration command (`0x7000fae760`) | investigation |
| 0x17 | When `flags & 0x17 == 0x17` the registration calls virtual +0x50 | investigation |
| 0x100, 0x200, 0x400 (= 0x700) | Activation: set by virtual +0xc8 (`0x700001e8d0` for most classes) on every actor at a level load. Cleared by virtual +0xc0 (`0x700001e7d0`: `flags &= ~0x700`, then the same walk over the components; virtual +0xb0, `0x700001e7b0`, forwards to it - as +0xb8 forwards to +0xc8) | coop.md, investigation |
| 1<<30 | Killed / dead | apple_trees.c, coop/field.inc |

Observed values: a Link made mid-level 0x10f, his hookshot 0x11f, its attack component 0x11d; after a level load
0x71f / 0x71d. A DoAction component can act when `(flags & 0x40000017) == 0x17` (and the cooldown is not > 0).

### 3.3 Known virtual slots (actor vtable)

| Slot | What | Source |
|---|---|---|
| +0x20 | Called after construction on the in-play spawn path | investigation |
| +0x28 | Called by the registration command | investigation |
| +0x50 | Called by registration once flags & 0x17 == 0x17 | investigation |
| +0x58 | Noted by pets.c next to the kill call: "actor vtable +0x58 (0)" (meaning unclear) | pets.c |
| +0x68 | A type question every component update asks its actor each frame (`0x70000ec310`); a damage component calls its owner's check through it (Octorok `0x7000554130`) | patches.py, investigation |
| +0x78 | Called right after creation on the in-play spawn path | investigation |
| +0xa0 | Base actor slot used by a follower's leave (pets.c) | pets.c |
| +0xc0 | Deactivation (clears 0x700): `0x700001e7d0` | disassembly |
| +0xc8 | Activation: flags |= 0x700, then walks its components (`0x7000fa8f20` with callback `0x700001f5b0`), setting theirs; three component types handled specially (one enables collision: `0x70010d5990(..., 3, 0)`) | investigation |
| +0x100 | Called by the zone manager when its camera zone is entered (`AREA_EVENT_TRIGGER` `0x7000029520`) | miniboss.c |

### 3.4 Construction, registration, update, kill

- **Base class:** ctor `0x700001bdb0` (349 callers; every actor class's ctor goes through it) copies the spawn
  info's key and type to +0x98 / +0xa0 (type stored at `0x700001be64`). Base destructor `0x7000fa7970`. Covers every
  actor however it is made, including bosses / mini-bosses their builders make outside the factory (Master Stalfos,
  Gohma, Hooker, Rola, Hiploop, Dodongo, Tail Cave boss). (coop.md)
- **Factory:** `0x70008f5ea0` (`CREATE_ACTOR`): x0 out shared_ptr, x1 &room, x2 spawn info. Reads type (u16 at
  info+8), takes `ACTOR_TYPES[type]+0` as the create fn (`ldr x24, [x8]` at `0x70008f5ed4`); null fn = "no such
  actor" (caller gets an empty handle). coop.md also names `0x70008f64d8` as "the factory".
- **Spawn info** (0x1c0 bytes): +0 key (u64), +8 type (u16), +0xc zone (u32), +0x10 eight parameters of 8 bytes
  {u32 value, u8 kind at +4} (kind 2 float, 3 integer, 5 string = index into `STR_POOL`), +0x180 position vec3,
  +0x190 rotation quaternion (x y z w), +0x1a0 scale vec3. (digs.c, pot_glow.c, shell_spots.c, enemy_random.c)
- **Spawning in play** (investigation): `0x700083da50` makes it, virtual +0x78 and +0x20, flags |= 8, then
  `0x7000fab540` hands it to the actor manager `[0x7001cc6d98]`, which queues a registration command
  (`0x7000fae3a0`); the command's run function `0x7000fae760` adds it to the manager's list, calls virtual +0x28,
  sets flag 0x10, calls virtual +0x50 once flags & 0x17 == 0x17. `0x7000fb3360` then registers it by name in
  `[GAME_MGR]+0x498`. The same template as 412 other spawn sites (`0x7000030d9c` is one).
- **Level load:** additionally activates every actor (virtual +0xc8). A spawn in play does not. (3.2)
- **Screens:** the Field spawner makes a level object's actor again when its screen comes back and its actor is gone;
  new actor set-up seen at `0x7000c44a00`; screen reset `0x70008e5e60`. (coop.md)
- **Per-frame update loop:** `0x7000facb80`; each update is `blr x9` at `0x7000facbf8` with x0 = an update
  callback whose first capture ([x0+8]) is the actor; its result (a std::future<void>) is written through x8 and
  waited on by the loop. Runs for every actor and component (~21,000 calls a frame with many screens made).
- **Kill:** `0x7000fac040` (`ACTOR_KILL`, x0 = actor; 1.0.0 `0x7000fabf70`).
- **Live check:** shared_ptr control block: use count at +8 is -1 (`~0`) once the object is gone (digs.c, events.inc).

### 3.5 Type ids seen

| Id | Type | Source |
|---|---|---|
| 0x87 | Seashell (ItemDrop / shell) | shell_spots.c |
| 0xb0 | Heart Piece | shell_spots.c |
| 0xb2 / 0xb3 | ItemPowerPiece / ItemGuardianAcorn | qol.c |
| 0xb5 / 0xb6 | ObjLawn (tall grass) / ObjGrass (bush) | digs.c |
| 0x135 / 0x136 | Green apple / apple | apple_trees.c |
| 0x194 | SinkingSword item actor (LAS: standing items, pots, bushes, digs) | digs.c |
| 0x1cd | ObjHolesDug | digs.c |
| 0x233 | AreaShellDrop (bomb hole) | shell_spots.c |
| 0x51, 0x53, 0x57, 0x58, 0x59, 0x5a, 0x5c, 0x5f, 0x60 | Rolling Bones, Hinox, Lanmola, Cue Ball, Master Stalfos, Gohma, Smasher, Blaino, Avalaunch (inferred: numbers in miniboss.c's class comments) | miniboss.c |

Names the fork matches in the type table: `Enemy...` (enemies), `EnemyZolGreen`, `EnemyZolRed`, `EnemyMiniZol` (Gel),
`EnemyLikeLike`, `ObjBladeTrap`, `ObjShopItem`, `ObjHouseCollapseTile`, `ObjKataKataFloor`, `ObjMatFaceFallFloor`,
`ObjBrokenFloorLv5`, `PlayerHookshot`, `PlayerBoomerang`. Grass / rock object names: `ObjRock-`, `ObjGrass-`,
`ObjGrass2x2-`, `ObjLawn-`, `ObjSkullStone-`.

### 3.6 Components

- Common: +0x08 self pointer (shared-from-this), +0x18 name inline, +0x48 owner actor.
- **Reaction component** (enemies', objects' e.g. a Blade Trap's, NPCs'): one base ctor `0x700012fae0` (stores its
  vtable at `0x700012fb5c`), destructor `0x70000eb410`. The enemy life / reaction class is `ENEMY_LIFE_VTABLE`
  `0x7001b5be00` (names "...ReactionComp", "...ReactComp", "...ReactionCom"): +0x168 hit points;
  "Zombie:ReactionComp" keeps "Link" at +0x278.
- **Effect player** (TailorPlayer: "<kind>:TailorPlayerComp", "ActorTailorPlayer"): built in place (vtable stored at
  `0x7000c53ac0` and `0x7000e24c1c`), destructor `0x7000e3b510`. Its "Link" read `0x7000e4ffa4` compares Link's
  zone (+0xb0) with its owner's to decide whether effects play. `TAILOR_PREPARE` `0x70010d4ee0` / `TAILOR_PLAY`
  `0x70010d4410` (x0 tailor, x1 &{channel ptr, len}, x2 &{effect ptr, len}).
- **Event component** (what an event's PlayAnimation runs on; the action dispatcher `0x70001049a0`'s actor): +8
  points at itself, +0xb0 the play-animation callback (std::function callable; its vtable's call slot +0x30). Enemies
  hold theirs at +0x12e0. `PLAY_ANIMATION` `0x7000119810`: x0 = [comp+0xb0], x1 &{name ptr, len}, x2 mode (0 start),
  s0 blend, s1 start frame. (coop/hearts.inc)
- **State machines** (enemy / mini-boss classes): `change(x0 sm, w1 state)`; sm +0xd current state (u8), +0xe
  previous; states 0x30 apart from sm+0x40, each {ctx (= the actor), enter, exec, exit} member-function pointers.
  A state whose entry is an event state named "<Name>End" is the death state. (miniboss.c)

## 4. Link (the player)

### 4.1 Identity

| | |
|---|---|
| Actor | `[[0x7001c97188]+0x4b8]` |
| vtable | `0x7001c17b70` (`PLAYER_VTABLE`) |
| Made by | The player creator, an event action at `0x700083a470` (also makes his `PlayerHookshot` and `PlayerBoomerang`; zone set-up `0x700083afa0`; switches the scene camera to his via `0x70000bfda0`; sets him as the manager's player) |
| Placed | At a spawn point by `0x700085d5f0` (x0 = the spot; reads the Link to move at `0x700085d610`). Spawn placement call also seen: `0x7000d337a0` |

### 4.2 Link struct offsets

| Offset | What | Source |
|---|---|---|
| +0x08 | Self pointer (live check) | weather.c |
| +0x80 | **PlayerCollision** component (also at +0x2d8). vtable `0x7001c501f0`, name at +0x18. Position vec3 **+0x80** (copy +0xb0), rotation quat (pure yaw on foot, x y z w) **+0x90** (copy +0xc0). Writing +0x80 moves Link; every other copy follows. Capsule centre 0.72 above the feet | doorjump.c, hookshot.c, coop |
| +0x318 | Link's own camera (every Link has one) | coop.md |
| +0x43 | Player-actor flag byte (bit 6, see 3.1) | patches.py |
| +0x4e0 | State machine (change fn `0x7000d34660`: x0 = Link+0x4e0, w1 = state) | coop/events.inc |
| +0x4ed | Current state slot (u8, index into +0x1340) | hearts.inc, weather.c |
| +0x1340 | State objects, 8 bytes each (< 0x40). State object +0x28 = its **kind** (u8); its vtable's typeinfo at vt-8 | hearts.inc, weather.c |
| +0x1568 | Animation controller: layers at [+0x78], 0x18 bytes each, layer +0x10 = the playing animation's info | hearts.inc |
| +0x1948 | Loaded by the walk states right after the speed multiplier (pointer; meaning not stated) | patches.py |
| +0x1b40 | Safe spot vec3: where the game puts him back after a fall | coop.c |
| +0x1b8c | Flags: bit 0 = takes no hits; bit 0x10000 = power-up music sound-disable (read by four music paths) | weather.c, powerup_music.c |
| +0x1b98 / +0x1ba0 | Follower (companion) shared_ptr {ptr, control}; ptr is the follower's interface at follower actor +0x2a8 | pets.c |
| +0x1e80 | float: blink / no-hit time after a hit (each Link has his own) | weather.c, coop.md |

### 4.3 States

State **slots** (+0x4ed values) seen (weather.c): 0 standing / walking, 1 jumping, 2 sword swing, 10 shield up,
26 hookshot. Dead state: change to **0x12** (0x13 checked with it); its start `0x7000daaf00` requests Common.bfevfl
"GameOver" (at `0x7000daaf6c`).

State **kinds** (state object +0x28) seen (coop): 0 on his feet; 3 (seen alternating with 11 while knocked back in spikes); 6 pushing a
block; 7 lever pull; 11 knocked back; 16 walk-in (doors, dungeon room change); 23 caught by a Like Like. The "can he
go" check `0x7000d58d70` holds kinds 18, 19, 21, 22, 23, 26 apart (mask 0x4ec0000) and looks past the kind for 1, 3,
11 - the event kinds.

### 4.4 Link functions

| Address | What | Source |
|---|---|---|
| `0x7000d355c0` | Link's update (x19 = Link) | coop.md |
| `0x7000d57260` | Link's move (every state): sums the frame displacement at [x29-0x80], then the collision move | patches.py |
| `0x7000d58d70` | "Can he go" check | coop |
| `0x7000dcdae0` | Walk state | patches.py |
| `0x7000db7d30` | Shield walk | patches.py |
| `0x7000dd4950` | Door walk-in (x0 its state, x1 Link) | coop/spawn.inc |
| `0x7000dd8060` | "Open" state (waits for an event to move him on) | coop/events.inc |
| `0x7000d92480` | Per-update "can I act" (DoAction) check | coop/events.inc |
| `0x7000ded9f0` | 8-way stick snap -> 32-bit angle (2^32 = full turn, 0 = down) | patches.py |
| `0x7000dbb760` | Hookshot action Start state | patches.py |
| `0x7000d1f580` | Attack: red tunic or Piece of Power | qol.c |
| `0x7000df8260` | GetMaxHealth | runtime table |
| `0x7000d50e60`-`0x7000d52fb0` | The 12 Items-page items' "use it now?" predicates (`0x7000d50e60` third bottle, `0x7000d510c0` first bottle; Roc's Feather last). Each: buttons newly pressed (`bic`), `tbnz w8, #3` (Y) -> `ldrb wN, [x8, #0xa2]`, `cmp wN, #slot`; X: bit 2, +0xa3. Run only while Link may use items | patches.py |
| `0x7000d52880` / `0x7000d52970` | X/Y bomb condition / "is the last bomb still out" (patch point `0x7000d52958`: `eor w8, w0, #1` -> `mov w8, #1` = many bombs) | rando.c |

### 4.5 Health and damage

- Health: PlayerStatus+0x84 (s16 quarter hearts). Max: `0x7000df8260`.
- Damage = `dmg << hero` (hero = PlayerStatus+0x120 == 1) at three `lslv` sites, then subtracted from +0x84:
  `0x7000d7c8c0` (an event's damage command, `0x7000d7c870`; no Link), `0x7000d9694c` (Link-hit function
  `0x7000d96830`; Link in x20), `0x7000d4c738` (enemy-hit function; Link in x21). Fixed damage (pits, deep water):
  `csel w20, w10(-4), w9(-2), eq` at `0x7000db1f6c` (Link in x19).
- Enemy hits are then halved when HasItem(0x6f) (`0x70008d1a30`) or PlayerStatus+0x9c is negative (Blue Clothes,
  Guardian Acorn); both paths meet at `ldrh w8, [x8, #0x84]` `0x7000d4c750` with the hit in w20, x8 = PlayerStatus,
  x21 = the Link. Only for a hit >= 1 (`cmp w24, #1; b.lt` at `0x7000d4c788`) does the game then rumble his
  controller and start his no-hit time (+0x1e80).
- Heals: `strh w8, [x9, #0x84]` at `0x70008d0a5c` (a heart) and `0x70008a39d4` (a healing amount); w23 = total
  before the max cap.
- **Hitting Link from code** (weather.c, lightning): `LINK_HIT_FILTER` `0x7000d4ce50` (would a hit land) and
  `LINK_DAMAGE_CB` `0x7000d4c690` (apply it) are called as the collision system calls them: x0 = a lambda object
  {+0: 0, +8: Link}, x1 = a pair / compare buffer, x2 = hit info with +0x10 Link, +0x18 [Link+0x10], +0x20 attack
  data (e.g. `BUZZBLOB_ATTACK` `0x70016efe94`: 1 heart, electric shock), +0x28 defence data (`LINK_DEFENCE`
  `0x7001701178`). Returns w0. No attacker = no knockback.

### 4.6 Hookshot and boomerang

| Item | |
|---|---|
| `PlayerHookshot` ctor | `0x7000d7e780`; vtable `0x7001c1ab38`. States: Attach / Shot / Return / Adhere / WaitToReturn |
| Shot update | `0x7000d7ecc0`: casts a ray each frame (cast `0x7000d69f10`, filter word at hook+0x4fc = 0x810200d5). On a hit reads the surface attribute word ([sp+0x38], at `0x7000d7ee9c`): bit 0x4000 = hookshot target -> Adhere ("HookshotPierce"); else Return ("HookshotHit"). Hit record at [sp+0xa8]. The ray is for surfaces only |
| Attack collision | `HookAttackColComp` at hook **+0x2c8**, made at `0x7000d81244` (`0x7000043220` with `0x70000cd8c0`'s attack info); generic attack/damage collision class (section 5). Its collision body record is named `player:hook` (comp +0x190) |
| Link's hookshot action | x19 in the action: +0x210 = Link, +0x2f0 = firing direction (flat vec, stored at `0x7000dbb924`), given to the hook by `0x7000d82a80` |
| `PlayerBoomerang` | Made by the player creator with Link (no further offsets known) |

Hooks read "Link" in their own update when they hit; each Link has his own pair.

### 4.7 A-button actions (DoAction)

- Each actable object has a DoAction component. Its update `0x70000ea240` checks itself against "Link"
  (reach, facing, able: comp +0x88) and the frame's best goes into `DOACTION_PENDING` `0x7001cacc78`; the frame end
  `0x70000e8be0` makes it `DOACTION_CURRENT` `0x7001cacc68` and lowers `DOACTION_COOLDOWN` `0x7001cacc88` (float).
- Link asks for the current one through `0x70000e8d90` (x0 = where the weak_ptr comes back; return address
  `0x7000d924e0` = his per-update check, `0x7000dccb88` = his A handler, which reads the component unchecked).
- Component: +0x40 flags (can act: `& 0x40000017 == 0x17`), +0x48 owner actor, +0x88 able, +0x1a0 action type
  (3 = lever pull, 4 = lift; others play an event on the manager's player: chest, sign, talk), +0x1b4..+0x1c4 its
  last check's findings (0x11 bytes: angle, distance, height, score, "can act").
- `DOACTION_TRIGGER` `0x70000eae90` (x0 component; w0 bit 0 = started) starts the action as the "open" state does.

### 4.8 Controllers, colour, follower

- Controller: `ldp x22, x26, [x8]` at `0x7000d35ae4` loads Link's controller shared_ptr from the npad list (16-byte
  entries; x19 = Link).
- Tunic colour frame: `blr x8` at `0x7000d35760` with s0 = frame (0 green, 1 red, 3 blue).
- Controller manager (x19 in its update): +0x4e0 state (polling or not), +0x4e8 player limit; its controller
  applet's +0x1c8 (set to 1 at `0x7000881d8c`). More controllers than the limit -> it asks for the controller applet
  (`0x7000016038`). (coop.c)
- Follower: spawned on a level load by `0x700085d980` (level's allow_companions, then `COMPANION_SPAWN`
  `0x7000d53c20`); `COMPANION_LEAVE` `0x7000d53ed0`. (pets.c)

## 5. Collision

### 5.1 Generic attack / damage collision component

| | |
|---|---|
| vtable | `0x7001b5aef8` (the hook's attack component and an Octorok's damage component are this class) |
| comp +0x170 | Collision object: +0x20 "enabled" byte, +0x28 first shape |
| shape +0x18 | Engine shape, vtable `0x7001c8df60`. Filter data at **+0x50** and a copy at **+0x60**: words `0x21`, an index (+0x54 / +0x64), `0`, `0x80000001`. (Read from a dump: +0x40 and +0x48 are 0, +0x50 = 0x00000009_00000021.) The index looks like a running count: P1's hook 9, a player's from a load 6, one made mid-level 0x69b / 0x6c2 |
| engine shape +0x28 | Engine actor, vtable `0x7001c8c2a0`; its +0x18 = a record named after the body ("player:hook"); record +0x20 / +0x28 / +0x30 = a pointer list (begin, end, capacity; `0x70010315d0` appends) |
| comp +0x190 | The collision body (the hook's: "player:hook") |
| Contact | Each attack / damage component registers a contact callback (function objects, vtables `0x7001b5b150` / `0x7001b5b198`, call operator `0x70000d76c0`) that calls the component's contact handler **`0x70000d67d0`**. On a hook-vs-enemy contact it runs on both components; the damage component then calls its owner's check via owner vtable +0x68 |
| Queries | Attack-collision queries seen: `0x70010bbee0` from `0x700094bbf0` |

### 5.2 Other collision objects

- Physics item body: actor +0x80 -> +0x170 -> +0x670 = motion: +0xa0 centre of mass (0.25 above the item; +0xc0 that
  offset), +0xd0 velocity (units/s). Writing velocity there moves the body. (shell_spots.c)
- Contact record: +0x10 -> body; body +0x438 = owner component (as `0x700003ee40` reads); component +0x48 = actor.
  (coop/events.inc)
- `CharaBodyCollision` `0x70001841c0` (Master Stalfos' ctor; flag at `0x70004b47c8` = blocks Link).
- Link: PlayerCollision (3.2); `PLAYER_COLLISION_VT` `0x7001c501f0`.

### 5.3 PhysX character controllers

- Every character controller gets the other characters as user-capsule obstacles (TouchedGeomType 1 = eUSER_CAPSULE:
  endpoints as doubles at +0x2c / +0x44, radius +0x5c). In CCT SweepTest's geometry loop, x28 = a touched geometry,
  x19 = the swept volume (centre doubles at +8), x8 = its sweep fn (`cbz x8` skips): `0x70012a88c8` (skip to
  `0x70012a89ac`) and `0x70012a89d0` (skip `0x70012a8a70`). (coop/spawn.inc)
- RigidCore::onShapeChange / sibling and RigidSim::getSimForShape (1.0.0: `0x7001408db0`, `0x7001408d60`,
  `0x70014090a0`; inferred 1.0.1 +0xd0): getSimForShape returns the address **1** for a shape with no sim object
  (asserted only in PhysX debug builds); sites `0x7001408eb0` / `0x7001408e50` (issue #314, Hot Head + Magic Rod).

## 6. Enemies, bosses, mini-bosses

### 6.1 How enemies find "Link"

All enemy code reads the game manager's +0x4b8. The 696 64-bit loads at +0x4b8 in main are listed in
`coop_player_reads_v1_0_1.txt` (531 with the manager as base). Enemy parts with their own updates: the reaction
component and the effect player (3.6). Like Like state handlers run outside the Like Like's update
(`0x70004a64c0`-`0x70004ab1d0`, x19 = Like Like; catch at `0x70004a9810`): nine reads at `0x70004a65f4`,
`0x70004a7040`, `0x70004a7fb8`, `0x70004a89f4`, `0x70004a8ce8`, `0x70004a9400`, `0x70004a9898`, `0x70004aac54`,
`0x70004ab1f0`. Zol reads: `0x7000799ab4`, `0x700079ae3c`, `0x700079d96c`, `0x700079df00`.

| Object | Notes | Source |
|---|---|---|
| Collapse tile (`ObjDungeonCollapseTile...`) | Checked in a job of its own each frame (`0x7000b01a40`, x19 tile), not an actor update: drops (`0x7000b005b0`) when "Link" is within 1.35 across of its middle; reads the player at `0x7000b01c70`. The job first checks the player's state (`0x700003ed00`, then his +0x1568) | coop.md |
| Lifted object (pot, rock, bomb) | Asks for "Link" in its own update and flies the way he faces (pot update `0x7000b27da4` via `0x7000d56b80`) | coop.md |
| Blade trap (`ObjBladeTrap`, an object) | Shoots along its row / column at "Link" lined up; reaction part "BladeTrap:ReactionCom" | coop.md |
| Zombie | "Zombie:ReactionComp" +0x278 holds "Link" | coop.md |
| Armos Knight | Alone: follows Link, stops 1-1.5 units off, slams about every 10 s | coop.md |

### 6.2 Mini-boss classes

`*_CHANGE_STATE` = the class's state-machine change(x0 = actor + sm offset, w1 = state). "pos" = offset of its
spawn-point x y z floats in the actor. (miniboss.c)

| Class | vtable | change | sm offset | idle | fight | Notes |
|---|---|---|---|---|---|---|
| Hinox | `0x7001b80e90` | `0x70003bfc60` | 0xff0 | 0 | 1 | State 10 requests HinoxStart |
| Smasher | `0x7001b86e58` | `0x70004301d0` | 0x1110 | 0 | 1 | + iron ball |
| Avalaunch | `0x7001badc60` | `0x70006d4f40` | 0xff0 | 0 | 1 | |
| Rolling Bones | `0x7001ba32c8` | `0x700061f250` | 0xff0 | 0 | 1 | + spike bar |
| Gohma | `0x7001b7e0e8` | `0x700038bf50` | 0x1200 | 13 | 0 | See below |
| Blaino | `0x7001b91408` | `0x70004ec6c0` | 0x11a0 | 10 | 1 | Rest state 5 after a knock-out |
| Cue Ball | `0x7001b82f70` | `0x70003e59a0` | 0xff0 | 0 | 3 | |
| Lanmola | `0x7001b8bc08` | `0x700047db50` | 0xff0 | 0 | 1 | Start action `0x7000487ce0` (up out of the sand) |
| Master Stalfos | `0x7001b8ea08` | `0x70004b9be0` | 0x1230 | 12 | 13 | 12 hidden; 13 shows him, 1 is his fight |

All have pos at +0x260. A dungeon mini-boss idles until its room's camera zone is entered (`0x7000029520` -> actor
vfunc +0x100), which hands a callback to the room's AreaEventBox; Link in the box fires it, its next state requests
the start event, whose end moves it to the fight state.

Per-class offsets:

| Class | Offset | What |
|---|---|---|
| Gohma | +0x15a4 / +0x15a6 | Move timer (s16) / its limit (timer = limit ends the move) |
| | +0x15c0 | Shared movement controller (turns to the heading) |
| | +0x15e4 | float speed |
| | +0x15ec / +0x15f0 | Heading (int angle, 2^31 = pi) toward a track end (+-90 deg) |
| | +0x1600 | Track end (snapped onto at `0x7000399550`) |
| Cue Ball | +0x1410 | Corners of its rail loop; +0x1760 the one it heads for; +0x1731 direction (0 forward). States 2 roll to first corner, 7 to the next, 8 turn |
| Master Stalfos | [+0x1160]+0x28c | Movement component: height at jump takeoff; jump is state 4 (vertical speed 6*sin(phase) - height) |
| Shadow Link | +0x16c0 | Holder; [holder+0x58] = his tailor (effects, channel "Other", effect "Sign") |
| | +0x1703 / +0x1704 | Appear event requested / running (he pops up once +0x1704 clears). Appear request `0x7000d157f0` within 3 units (ShadowLink.bfevfl); his music `0x7000ce8ce0` (BGM_PANEL_SHADOW_LINK), held until `0x7000ce8f00` |
| Blaino | | Knock-out countdown object `0x7000d97df0`: +0xb8 counter, +0xc8 its reference to the puncher; then MightPunchCrashOut (restart warp to his ring) |
| Moblin Chief | | Init `0x70001f8af0` deletes him unless PlayerStatus+0xac bit 7 |

### 6.3 Enemy life

`ENEMY_LIFE_VTABLE` `0x7001b5be00`: the first member of an enemy that is a life component (scan +0x80..+0x3000 for
this vtable); +0x168 = hit points (0 = dies). (miniboss.c)

## 7. Levels, events, HUD, audio, items, camera, field

### 7.1 Level jumps and warps

- Event action GameControl.RequestLevelJump `0x70007ff160` fills the pending request object R through
  `0x700081b950`: level name R+0xa8 (length u64 R+0xe8), locator / spawn point (e.g. "Field_11C") R+0xf0 (length
  R+0x130), offset vec4 R+0x140. R registers itself in its parent while a level loads (`str xR, [x8, #0x1c0]` at
  `0x700080c464` / `0x700080d718`).
- Request processor `0x700081ba80` (x0 = request; every level change passes its prologue). A warp = fill a 0x200-byte
  request with those four fields and call it with x0 = request on the game thread, after the last load settled and
  while no level change of the game's own is pending (R's destination fields set) and the player component exists.
- Locators that throw on arrival without the Flippers (start mode WaterTopOutForward / WaterSide...): Field_03O,
  Field_15K, Field_03K, EagleTowerExit_01A_b, FishingBoat_01A, MamboCave_01A; Warp_DanpeiHouse and
  Warp_ShellHouseEast have no plain start. (warp.c)

### 7.2 Chamber Dungeon (Dampe, "PanelAdventure")

| | |
|---|---|
| Manager | `[0x7001cb9fb8]`, 0x1920 bytes, ctor `0x7000cdf1c0` (allocate with `GUEST_NEW` `0x7001431240`) |
| Layout | manager+8: 64 u32 panel ids (MapPieceList mId, row-major 8x8, 0xffffffff empty). Cells the engine reads: layout+0x100 + i*0x18 = {sheet, shared ref, catalogue row} (fill with `PANEL_LOOKUP` `0x7000cd0970`) |
| Cached values | layout+0x700 room count, layout+0x1578 / +0x157c entrance panel id / cell, valid while bits at layout+0x740 are set (bit0 room count, bit1 entrance; clear to recompute) |
| Catalogue row | +0x20 -> int[4] door types (N, W, E, S), +0x84 mFlag |
| Run | `CHAMBER_PREPARE` `0x7000ce34b0`(mgr) first; mgr+0x15e0 / +0x15e1 mode bytes (3, 3 as a challenge sets them), mgr+0x15e4 challenge index (MapPieceTheme row; -1 crashes), mgr+0x18fc = 1 marks an adventure; `CHAMBER_START_LOCATOR` `0x7000cdbd80`(layout) -> start locator; then a level jump to "PanelAdventure" with it. Entered normally by the PanelAdventure event action `0x70007ee040` |
| Exit prompt | `0x70007f1540` opens it (`bl` at `0x70007f15e0` -> `0x7000f05350`); callback stores the choice at mgr+0x18f8 (0 = leave) |
| Flags | GlobalFlags 470-472 = AdvTutorialFirst / Knockdown / Timeover (Dampe's intros) |

### 7.3 Events

- Request an entry of a flowchart (as the game requests GameOver at `0x7000daaf6c`; hint_sheet.c), scratch S:
  1. `EVENT_RES_LOAD` `0x7000fbbe50`(x0 = S+0x60 out, x1 = [EVENT_RES_MGR], x2 = &{"event/Common.bfevfl", 19})
     -> {pending load, resource} shared_ptrs; if S+0x60 non-null, `EVENT_RES_WAIT` `0x700003dc20`(x0 = it).
  2. `EVENT_FLOW_HANDLE` `0x700003d9f0`(x0 = S+0x30 out, x1 = S+0x60) -> flowchart handle.
  3. `EVENT_REQUEST_NEW` `0x700080c290`(x0 = S+0x20 out, x1 = S+0x30, x2 = &{entry ptr, len}) -> request;
     `EVENT_REQUEST_PREPARE` `0x700080db00`(x0 = request).
  4. `EVENT_SUBMIT` `0x700081f2e0`(x0 = S+0x50 out, x1 = [EVENT_MGR], x2 = S+0x20).
  5. Release the shared_ptr controls at S+0x58, +0x28, +0x38, +0x68, +0x78 with `SP_RELEASE` `0x7000bae080`.
- Any event request holds Link while it runs.
- An actor's AI starting an event: `0x700094a510` (x20 = AI, [AI+0x58] = actor, entry name {ptr, len} at
  [x29-0x40]); the request is built at `0x700094a568` (`bl 0x700080c290`).
- Event action dispatch through the actor's handler map (1.0.0 `0x700108e300`; inferred 1.0.1 `0x700108e3d0`): an
  empty std::function throws bad_function_call (1.0.1 site `0x700108ee98`), caught in the same function to abort
  the event. Event dispatcher (1.0.0 `0x700108ddc0`) calls a handler through the std::function call (1.0.0
  `0x700108dfc0`; 1.0.1 call site `0x700108df18`). Missing event parameters throw: String (1.0.0 `0x700109fb80`),
  Float (1.0.0 `0x700109fad0`; inferred 1.0.1 +0xd0 each). The Float reader `0x70007f3c90` treats time <= 0 as
  instant. Recompiled code cannot unwind, so these are hooked (section 8.2).
- AreaEventBox (fight-room lock, boss / mini-boss start, cutscene trigger): bare collision shape `0x7000068110`;
  contact signal calls the box's fire `0x7000067210` (x0 box, x1 contact) for any Link, with no check; the box keeps
  one contact (+0x370, the last begun) and fires with it the next frame (`0x700006a980`).
- AreaLevelOpen (doors, loading zones): contact `0x7000076dc0` -> `0x7000076e90` keeps a touching player actor (+0x43
  bit 6 clear) as its pending exit (+0x400); its update `0x7000075320` plays the exit for the manager's player once he
  can go (`0x7000d58d70`; the exit event `0x7000075ac0`). Contact end functor `0x7000076aa0` drops the pending exit
  (`0x7000076b20`).
- AreaUpstairs / AreaDownstairs: stepped on `0x70000617a0`, off `0x7000061600`, keeps the player at +0x3f0, update
  `0x7000060140` starts the stairs event.
- AreaWarpHoleEventBox: stepped in `0x700008c560` (jump, "Where to?" menu), out `0x700008c3d0`. Overworld warp-hole
  objects (`0x7000cba000`..., event/Field.bfevfl "WarpHole"): in `0x7000cbd3e0` (sets +0x405; its update walks the
  manager's player over and jumps him in), out `0x7000cbd250`.
- "ZeldaScroll" event (`0x70000bc540`): the walk into a new scroll area, answered by a camera with +0x515 set.

### 7.4 Items

| Address | What | Source |
|---|---|---|
| `0x70007e18b0` | AddItem(itemId, count, index, autoEquip) = Inventory.AddItemByKey. itemId = Items.gsheet `itemID` (rows 41+ differ from row numbers). Dungeon items go to the current dungeon only | warp.c, keysanity.c |
| `0x70008d08e0` | Inner add (34 callers: events, chests, drops); sets tunic bits in PlayerStatus+0x9c and clears the other (`0x70008d1118`, `0x70008d12f8`) | patches.py |
| `0x70008d1a30` | HasItem(item, count, index) (`HAS_ITEM`) | patches.py |
| `0x70007e2b7c` / `0x70008d1a50` / `0x70008d090c` | HasItem(itemType) (EventFlow enum) / HasItemByKey / AddItemByKey (1.0.0 offsets; this range matched 1.0.1 everywhere the tables overlap) | save-format |
| `0x70007e1c70` | AddPrizeByKey: looks up the prize row by symbol, calls AddPrize(row, count, index) `0x70001489c0` (rows 0-20) | shopsanity.c |
| `0x700089b960` | ItemDrop roll (reads row +0x38 mAllowBuffReplace); also called by the heart-piece pickup `0x70000855a0` to add a heart piece | patches.py |
| `0x70008c4510` | Item actor's "should I appear?": HasItem(item +0x2e8, 1, index +0xbc) | patches.py |
| `0x7000a4a9b4` | Shopkeeper exit check: adds the carried good's item (good +0x488). LAS's steal patch is at `0x7000a4a8f0` | shopsanity.c |
| `0x70008dd860` | Seashell Sensor check (skips a beacon whose +0x14c is 0); AreaShellDrop: +0x2b8 ShellBeacon, +0x2c8 seashell index, set-up `0x70000849f0`, removed `0x7000085510` once owned | shell_spots.c |
| `0x7000cb04e0` | Apple tree bonk (`TREE_BONK`); tree +0x4c0 is-apple-tree, +0x4c8 apples, handles (actor, control) at +0x4f8 / +0x508 / +0x518, +0x528 set after a bonk; knock-down `0x7000cb18f0` | apple_trees.c |
| `0x7000ddc530` | Shovel dig commit; snaps to a tile centre `0x7000ddafc0`; hole spawn `0x7000d228d0` | digs.c |

Item ids (AddItem): bomb 4, magic powder 11, compass 44, map 45, stone beak 47, small key 48, Nightmare Key 49,
arrow 60, blue rupee 102, PanelDungeonPiece 114. Items-page item slots checked by the 12 predicates, in order:
0xf, 0xe, 0xd, 0x6, 0x5, 0x4, 0x3, 0x2, 0x7, 0xc, 0x8, 0x9. HasItem(0x70) and HasItem(0x6f) are what powerup_music.c checks for an active power-up (which is the Piece of Power and which the Guardian Acorn is not stated; 0x6f is also the enemy-hit halving check).

### 7.5 HUD, menus, text (ui2d)

- HUD: set-up `HUD_SETUP` `0x7000ea85c0` (vtable slot 6, on every screen load) copies health / max into HUD
  +0x61c0 / +0x61c2, then follows them a quarter heart at a time. Normal state update prologue `0x7000eaa8d0`
  (x0 = HUD). Action prompt pane placed at `0x7000eab208`.
- ui2d Pane (`nn::ui2d::Pane`): +0x8 link node, +0x18 parent, +0x20 child list root (next at +0x28), +0x30 / +0x34
  translate x / y, +0x48 scale (+0x4c y), +0x58 flags (bit 0 visible, 0x10 matrix dirty - set after moving), +0x59
  alpha, +0xb0 name. HUD panes: Cm_HUD's N_HUD_00, N_SetItem_00, L_Interact_00.
- Pause menu: update `0x7000ee7050`; Items page `0x7000eb7da0` (x0 page: +0x4020 cursor 0-11, cells +0x598 +
  0x280*cell = item slot; cursor move `0x7000eb80ac`, neighbour 0x27 = none); Map page `0x7000e938d0` (page +0x640 =
  N_MapCore_00); dungeon map page `0x7000e78810` (page+0x8 -> +0x10 -> panes: N_Map_00 +0x70, N_MapPosition_00
  +0x7a8, N_MapOrigin_00 via +0xf0 -> +0x6a8); a page's state byte +0xfd (0x01 / 0x10 opening, 0x02 ...). Item
  name / explanation messages "UI:ItemName_{0}" / "UI:ItemExplanation_{n}" (1.0.0 builder `0x7000eb6604`).
- File select: (1.0.0 `0x7000e61040`, list helper `0x7000e6b160`; inferred 1.0.1 +0xd0), cursor byte +0x159b0.
- Message window: opening (1.0.0 `0x7000f53c80`) sets the text step pane +0x284 = 2 (x pane +0x280 frames per
  character unit); pane update (1.0.0 `0x7000f534e0`); B mid-page (1.0.0 `0x7000f53b60`). Question set-up (1.0.0
  `0x7000ef43b0`) stores the answer count at window +0x1cc4; Selecting (1.0.0 `0x7000ef2250`, `0x7000ef2300`)
  indexes the InOutChoice2/3/4 panes by it unchecked. All inferred 1.0.1 +0xd0.
- Countdown (Dampe's timed challenges): `region_common/ui/Game.arc` -> `blyt/Dg_TimeCount.bflyt`, pane N_Time_00
  (translate x at pane+0x2c in the file).

### 7.6 Audio

- `BUFF_MUSIC_START` `0x70000af560` (x0 = 0: resume loop without the pickup intro) / `BUFF_MUSIC_STOP`
  `0x70000afa60`. The pickup starts BGM_NUTS directly at `0x7000d79774`.
- Link +0x1b8c bit 0x10000 mutes the power-up music; read at `0x70000a1638` (music update), `0x70000a67d8` (zone
  music), `0x7000805acc` (event music), `0x7000d6d11c` (power-up presentation).
- `PLAY_SYSTEM_SE` `0x70000a57e0`: (x0 &{label ptr, len}, x1 0, s0 volume, s1 pitch).
- Movie player `0x7000912640` (slot 6 Stop), its decoder thread `0x70009114b0`. (movie.c)

### 7.7 Camera

Camera object (coop/camera.inc, coop.md):

| Offset | What |
|---|---|
| +0x2a8 | Loaded right after the view matrix is set (at `0x7000010908`) |
| +0x350, +0x560, +0x734 | Shadow reach values (24 each on the Field) |
| +0x3b0 | Object the camera's update `0x70000c7fd0` runs on |
| +0x4fc / +0x500 | New camera area / the one before |
| +0x508 | Level kind (Field = 3; houses, caves, dungeons below) - set by the level's camera set-up `0x7000855040` for the manager's Link's camera only |
| +0x510 | Node; eye position at node+0xb0 (the culling pass builds its view from it) |
| +0x515 | Scrolls: answers a new scroll area with "ZeldaScroll" (set for kind < 3) |
| +0x564 | Vertical field of view (radians) |
| +0x568 | Read by the culling pass at `0x7000852ad8` |
| +0x56c | Far plane |
| +0x570 | Aspect (culling pass) |
| +0x580..+0x5bf | View matrix (columns +0x580 / +0x590 / +0x5a0, translation +0x5b0) |

- A camera of kind <= 2 told of a new area tells the screen manager (`0x7000866130`), which makes every loaded screen
  outside it fresh (a room change).
- Camera-area manager `0x7000836270`, signal at +0x750 (x1 = the area the manager's player now stands in).
- View set-up `0x7000010540` (once per drawn frame): makes the shadow map's world box around the player (view +0x200
  min, +0x210 max; copied to a second view at `0x7000011228`); shadow projection fit `0x700100a460`; lookup matrix
  `0x7001009f30` (renderer "View" +0x2a0). Shadow map 2048 x 2048, one per frame.
- Culling pass `0x70008529b0` (x27 = view camera), once per world step (twice a frame in 30 fps mode).
- 16:9 aspect literal `0x70016afba4`; sites in section 8.2.
- Post-process: tilt-shift object at post-process manager +0x620 (set up once, 1.0.1 `0x70010712f0`): +0x40 / +0x80
  focus distance (26.5 on the Field), +0x50 its uniform buffer (buffer +0x60 = pointer to the block name
  "PFXTiltShiftParam"), +0x58 std::string copy of the name. A wrong name = no blur.
- Renderer: +0x648 dynamic-resolution scaler (enabled byte, table indexes, level tables - docked [0 3 6 9 10]),
  +0x6e4c resolution level (1080 - 36 x level lines at 16:9), +0x81c0 GPU timing counters. Frame timer
  `0x700000c910` reads the swap interval from the system object +0x812c (written only by `0x7000017800`). Dynres
  set-up / step (1.0.0 `0x700107cb90` / `0x700107d110`; inferred +0xd0).

### 7.8 Field screens and level objects

- The Field: 16 x 16 screens (`Field_01A` .. `Field_16P`), each 15 x 12 units, .leb grid 10 x 8 tiles of 1.5 units.
  Tile id used by the fork: ((row - 1) * 16 + column) * 80 + tile.
- `0x700085c280` (each frame) handles the level's screens with three boxes centred on the manager's player:
  made (s1 x s2 at `0x700085c430`; Field 75 x 60 = 7 x 7 screens; a column is made when its middle comes within 45),
  kept (90 x 72 at `0x700085c628`; s0 west, s1 east, s3 north, s2 south as offsets from the centre), active (s3, s4
  half sizes, s5 offset deep at `0x700085c848`; about 4 x 3 screens). One `MapStatic` ground actor per screen.
- **No "destroyed" record:** a thrown rock's actor stays as a dead stub while its screen is kept; once the screen is
  let go and made again, the spawner makes a new actor for an entry with none (grass and rocks come back fresh).
- Screen holders (digs.c): scene = `SCENE_LOCK`(out, [GAME_MGR]+0x4a8); scene +0x228 = Link's screen's holder,
  holder +0x1b8 its room. Holders are 0x260-byte pool entries: +0x50 self, +0x240 1 while loaded, +0x1a8 info
  (info+0x74 = the screen's file name, "Field_11A.leb"), +0x1b8 room, +0x1c0 its control.
- Sky / sea north of the overworld (MapSkySea): pieces 60 apart, box loads at `0x70008fb1b8` / `0x70008fb1c4`.

### 7.9 Engine notes

- Frame jobs (issue #19): per-frame kick-off `0x7000015730` (four inlined copies) rebuilds owner X's frame job J at
  X+0xc8 (vtable `0x7001c51308`, [J+0x10] = 0, then the delegate). Worker / task loop (1.0.0 `0x70013c21e0`) runs J
  (slot +0x10, 1.0.0 `0x70010cceb0`) and then its finish (slot +0x38, 1.0.1 `0x70010cd070`). The recycled action
  class `0x7001c51358`: slot +0x00 re-inits (clears [this+0x10]), +0x28 / +0x30 forward to [this+0x10] unchecked,
  +0x38 checks its delegate (+0x18). 1.0.0 offsets in delegate_null.c; inferred 1.0.1 +0xd0.
- Owner-thread call (class vtable `0x7001c490b0`, slot +0x48; 1.0.0 `0x7001048cf0`, task 1.0.0 `0x7001049ed0`): locks
  O+0x18, posts to [O+8], waits on O+0x38 once without a predicate; the task notifies without the mutex.
- Font: missing-character box glyph U+25A1 (font +0x34) used unchecked (sites in section 8.2).
- Bone-following effect emitter: bone node at emitter +0x50 (looked up once at creation; null without a model
  instance).
- Heaps: nn::mem::StandardAllocator, nn::lmem unit heaps, frame heaps of fixed sizes; running out returns NULL
  (seen as a null weak_ptr in 1.0.0 `0x70010d0f00` with ~450 actors). (heaps.c)

## 8. Address tables and hook / patch sites

### 8.1 Named addresses (runtime table)

Generated from `runtime/game_addresses/runtime_v1_0_1.h` (and `_v1_0_0.h` for the 1.0.0 column).

#### Globals and data

| Name | 1.0.1 | 1.0.0 | What it is |
|---|---|---|---|
| `GAME_MGR` | `0x7001c97188` | same | Pointer to the game manager (also read as the frame controller by `runtime/nvn/pace.c`). Fields: section 2. |
| `PLAYER_STATUS_PTR` | `0x7001cc1368` | same | Pointer to PlayerStatus (inventory, health, level name at +0). Section 2. |
| `LIVE_SAVE` | `0x7001cbb220` | same | The live save buffer: an exact image of `savedataN.bin` (0x5EE0 bytes); `save-format.md` puts the save manager at main+0x1cbb200 with this buffer at its +0x20. |
| `FLAG_PTR` | `0x7001cc1388` | same | Pointer the flag object uses to reach the GlobalFlags bitfield (normally live save +0x5DC0). |
| `FLAG_CAP` | `0x7001cc1390` | same | u32 byte capacity of that bitfield; every flag get / set bounds-checks against it. |
| `CHAMBER_MGR` | `0x7001cb9fb8` | same | Pointer to the Chamber Dungeon (Dampe / PanelAdventure) manager, a 0x1920-byte object. Section 7. |
| `PANEL_CATALOGUE` | `0x7001cb9e50` | same | Pointer to the MapPieceList catalogue table object (chamber panels). |
| `ACTOR_REGISTRY` | `0x7001cb2d00` | same | Pointer to the level's placed actors by key: unordered_map, first node at +0x10; node +0 next, +0x10 key, +0x18 shared_ptr<Actor>. |
| `ACTOR_TYPES` | `0x7001bcc898` | not mapped | The actor factory's type table: 0x58 bytes per type, +0 create function, +0x50 name (char*). Indexed by actor type id. |
| `STR_POOL` | `0x7001cab2d0` | same | The game's string pool for type-5 (string) spawn parameters: std::vector<{std::string, u64 refs}> (32-byte entries). The game never fills it. |
| `EVENT_RES_MGR` | `0x7001cc6db8` | same | Pointer to the event resource manager (loads .bfevfl flowcharts). |
| `EVENT_MGR` | `0x7001cb24d8` | same | Pointer to the event manager (takes event requests). |
| `DOACTION_CURRENT` | `0x7001cacc68` | not mapped | A-button DoAction slot: the current component (weak_ptr {ptr, control}, 16 bytes). Section 4. |
| `DOACTION_PENDING` | `0x7001cacc78` | not mapped | A-button DoAction slot: the frame's best so far (weak_ptr, 16 bytes). |
| `DOACTION_COOLDOWN` | `0x7001cacc88` | not mapped | float: countdown after an action (A does nothing while > 0). |
| `SDK_SYSTEM_OBJECT` | `0x7004cf9e38` | same | In the sdk module. [obj]+0x680 = the current camera zone record (+0 u32 id, +0x10 min vec3, +0x20 max vec3); none on the overworld / title / loading. Zone thin in z (< 6) = a side-view room. |

#### Functions

| Name | 1.0.1 | 1.0.0 | What it is |
|---|---|---|---|
| `CREATE_ACTOR` | `0x70008f5ea0` | same | Actor factory: (x0 out shared_ptr, x1 &room, x2 spawn info). Takes the create fn from ACTOR_TYPES by the spawn info's type (+8). |
| `ACTOR_KILL` | `0x7000fac040` | +0xd0 | Kill an actor (x0 = actor). The game's own path used for Link's follower and SinkingSword.Destroy. |
| `ADD_ITEM` | `0x70007e18b0` | same | AddItem(x0 itemId, x1 count, x2 index, x3 autoEquip): the call behind Inventory.AddItemByKey; itemId = Items.gsheet `itemID` (not the row). |
| `GUEST_NEW` | `0x7001431240` | +0xd0 | Allocates guest memory: called with a size (0x1920 for the chamber manager), returns the object (operator new, inferred). |
| `SP_RELEASE` | `0x7000bae080` | same | shared_ptr release (x0 = control block). |
| `WEAK_RELEASE` | `0x70014312c0` | +0xd0 | std::__shared_weak_count::__release_weak (x0 = control block). |
| `SCENE_LOCK` | `0x700001b730` | same | (x0 out shared_ptr, x1 scene handle = [GAME_MGR]+0x4a8): what the shovel's dig commit calls to get the scene. |
| `SPAWN_HOLE` | `0x7000d228d0` | same | (x0 out, x1 transform, x2 zone, x3 &room, w4, s0): spawn ObjHolesDug (type 0x1cd) as the shovel does. |
| `HOLE_LIST_INIT` | `0x7000d22860` | same | First-use init of the game's dug-hole list (the shovel calls it first). |
| `STR_POOL_INIT` | `0x700001d510` | same | First-use init of STR_POOL (clears the vector). |
| `CHAMBER_CTOR` | `0x7000cdf1c0` | same | Chamber manager constructor (x0 = new 0x1920-byte object). |
| `CHAMBER_PREPARE` | `0x7000ce34b0` | same | (x0 = manager): prepares a run; resets the manager's state (call before writing the layout). |
| `CHAMBER_START_LOCATOR` | `0x7000cdbd80` | same | (x0 = layout = manager+8): returns the entrance panel's start locator (char*). |
| `PANEL_LOOKUP` | `0x7000cd0970` | same | (x0 out cell, x1 catalogue, x2 panel id): out = {sheet, ref, row ptr}. |
| `HUD_SETUP` | `0x7000ea85c0` | +0xd0 | The HUD's set-up (its vtable slot 6, run on every screen load; x0 = the HUD): copies health / max health into HUD +0x61c0 / +0x61c2. |
| `COMPANION_SPAWN` | `0x7000d53c20` | same | (x0 shared_ptr out, x1 Player): spawn the save's companion beside Link. |
| `COMPANION_LEAVE` | `0x7000d53ed0` | same | (x0 Player): companion byte to 0, the follower sent away (Link.LeaveCompanion). |
| `TREE_BONK` | `0x7000cb04e0` | same | An apple tree's (ObjTree) bonk handler: knocks every live apple down (via main+0xcb18f0). |
| `LINK_HIT_FILTER` | `0x7000d4ce50` | same | Link's damage component's hit filter (state, blink +0x1e80, +0x1b8c bit 0): would a hit land now. Calling convention: section 4.5. |
| `LINK_DAMAGE_CB` | `0x7000d4c690` | same | Link's damage callback as the collision system calls it: damage (difficulty sites included), sound, blink. |
| `PLAY_SYSTEM_SE` | `0x70000a57e0` | same | Audio.PlayOneshotSystemSE: (x0 &{label ptr, len}, x1 0, s0 volume, s1 pitch). |
| `BUFF_MUSIC_START` | `0x70000af560` | same | Guardian Acorn / Piece of Power music start (x0 = 0: resume the loop without the pickup intro). |
| `BUFF_MUSIC_STOP` | `0x70000afa60` | same | Power-up music stop (restores volume on existing handles; does not restart the area track). |
| `AREA_EVENT_TRIGGER` | `0x7000029520` | same | Zone manager -> actor vfunc +0x100: entering a room's camera zone hands a callback to the room's AreaEventBox (how a dungeon mini-boss is started). |
| `LANMOLA_START` | `0x7000487ce0` | same | Lanmola's own start action (what his screen's event runs: up out of the sand); called with x0 = the actor. |
| `TAILOR_PREPARE` | `0x70010d4ee0` | +0xd0 | Effect player (Tailor) prepare: (x0 tailor, x1 &{channel}, x2 &{effect name}). |
| `TAILOR_PLAY` | `0x70010d4410` | +0xd0 | Effect player play, same arguments. |
| `PLAYER_MAX_HEALTH` | `0x7000df8260` | not mapped | GetMaxHealth(): quarter hearts = 12 + 4 x (containers + pieces / 4). |
| `PLAY_ANIMATION` | `0x7000119810` | not mapped | An actor's play-animation (what an event's PlayAnimation runs): x0 the callback at event component +0xb0, x1 &{name ptr, len}, x2 mode (0 start), s0 blend time, s1 start frame. |
| `DOACTION_TRIGGER` | `0x70000eae90` | not mapped | Start a DoAction component's action (x0 = component; returns w0 bit 0 = started). What Link's "open" state calls. |
| `COOP_SET_PLAYER` | `0x70008387c0` | not mapped | The game manager's player setter: x0 manager, x1 &shared_ptr; assigns +0x4b8 / +0x4c0 with reference counting. |

#### Return addresses used to tell callers apart

| Name | 1.0.1 | 1.0.0 | What it is |
|---|---|---|---|
| `DOACTION_TAKE_RET` | `0x7000dccb88` | not mapped | Return address inside Link's A handler when it asks for the current DoAction's kind (it then reads the component unchecked). |
| `DOACTION_CHECK_RET` | `0x7000d924e0` | not mapped | Return address of Link's per-update "can I act" check (main+0xd92480) asking for the current DoAction. |

#### Vtables (class identity: compare [obj+0])

| Name | 1.0.1 | 1.0.0 | What it is |
|---|---|---|---|
| `PLAYER_VTABLE` | `0x7001c17b70` | not mapped | Link's actor class (every co-op Link is one). |
| `PLAYER_COLLISION_VT` | `0x7001c501f0` | same | "PlayerCollision" component (Link +0x80; name inline at +0x18). Position vec3 at +0x80 (copy +0xb0), rotation quat at +0x90 (copy +0xc0). |
| `SINKING_SWORD_VTABLE` | `0x7001c0e4e0` | same | SinkingSword item actor (type 0x194): what LAS uses for standing items, pots, bushes, digs. |
| `ITEM_HEART_PIECE_VTABLE` | `0x7001bc9d50` | same | ItemHeartPiece actor (also keeps its name at +0x398: "ItemHeartPiece-<id>"). |
| `ENEMY_LIFE_VTABLE` | `0x7001b5be00` | same | Enemy life / reaction component class ("Zombie:ReactionComp", "ZolRed:EnemyReactionCom"... one class). +0x168 hit points (written 0 = dead on next hit check). |
| `SHADOW_LINK_VTABLE` | `0x7001c16730` | same | Shadow Link actor class. |
| `DELEGATE_JOB_VTABLE_LIVE` | `0x7001c51308` | same | Frame job class as rebuilt by the per-frame kick-off (main+0x15730); job lives at owner X+0xc8. |
| `DELEGATE_JOB_VTABLE_BASE` | `0x7001c51358` | same | The recycled "action" class whose slots +0x28 / +0x30 forward to [this+0x10] unchecked (issue #19). |

#### Mini-boss classes (vtable, state-machine change(sm, state))

| Name | 1.0.1 | 1.0.0 | What it is |
|---|---|---|---|
| `HINOX_VTABLE` | `0x7001b80e90` | same | see section 6.2 |
| `HINOX_CHANGE_STATE` | `0x70003bfc60` | same | see section 6.2 |
| `SMASHER_VTABLE` | `0x7001b86e58` | same | see section 6.2 |
| `SMASHER_CHANGE_STATE` | `0x70004301d0` | same | see section 6.2 |
| `AVALAUNCH_VTABLE` | `0x7001badc60` | same | see section 6.2 |
| `AVALAUNCH_CHANGE_STATE` | `0x70006d4f40` | same | see section 6.2 |
| `ROLLING_BONES_VTABLE` | `0x7001ba32c8` | same | see section 6.2 |
| `ROLLING_BONES_CHANGE_STATE` | `0x700061f250` | same | see section 6.2 |
| `GOHMA_VTABLE` | `0x7001b7e0e8` | same | see section 6.2 |
| `GOHMA_CHANGE_STATE` | `0x700038bf50` | same | see section 6.2 |
| `BLAINO_VTABLE` | `0x7001b91408` | same | see section 6.2 |
| `BLAINO_CHANGE_STATE` | `0x70004ec6c0` | same | see section 6.2 |
| `CUE_BALL_VTABLE` | `0x7001b82f70` | same | see section 6.2 |
| `CUE_BALL_CHANGE_STATE` | `0x70003e59a0` | same | see section 6.2 |
| `LANMOLA_VTABLE` | `0x7001b8bc08` | same | see section 6.2 |
| `LANMOLA_CHANGE_STATE` | `0x700047db50` | same | see section 6.2 |
| `MASTER_STALFOS_VTABLE` | `0x7001b8ea08` | same | see section 6.2 |
| `MASTER_STALFOS_CHANGE_STATE` | `0x70004b9be0` | same | see section 6.2 |

#### Event requests (as the game requests GameOver from event/Common.bfevfl)

| Name | 1.0.1 | 1.0.0 | What it is |
|---|---|---|---|
| `EVENT_RES_LOAD` | `0x7000fbbe50` | +0xd0 | (x0 out {pending load, resource} (two shared_ptrs), x1 EVENT_RES_MGR object, x2 &{path ptr, len}). |
| `EVENT_RES_WAIT` | `0x700003dc20` | same | (x0 pending load): wait for it. |
| `EVENT_FLOW_HANDLE` | `0x700003d9f0` | same | (x0 out flowchart handle shared_ptr, x1 the load result pair). |
| `EVENT_REQUEST_NEW` | `0x700080c290` | same | (x0 out request shared_ptr, x1 &flowchart handle, x2 &{entry name ptr, len}). Also the call an actor's AI makes at ACTOR_EVENT_SITE. |
| `EVENT_REQUEST_PREPARE` | `0x700080db00` | same | (x0 request). |
| `EVENT_SUBMIT` | `0x700081f2e0` | same | (x0 out shared_ptr, x1 EVENT_MGR object, x2 &request). The event holds Link while it runs. |

#### Game data (strings, attack data)

| Name | 1.0.1 | 1.0.0 | What it is |
|---|---|---|---|
| `STRING_COMMON_FLOW` | `0x70016c2ca8` | same | "event/Common.bfevfl" (length 19). |
| `STRING_OTHER` | `0x70016dc7b9` | same | "Other" (effect channel, length 5). |
| `STRING_SIGN` | `0x70016e6ef9` | same | "Sign" (Shadow Link's glow effect, length 4). |
| `BUZZBLOB_ATTACK` | `0x70016efe94` | same | Buzz Blob attack data (1 heart, reaction 1 = electric shock, "dmg_elec"). |
| `LINK_DEFENCE` | `0x7001701178` | same | Link's defence data (paired with an attack in a hit info). |

#### Runtime-only bounds and engine hooks

| Name | 1.0.1 | 1.0.0 | What it is |
|---|---|---|---|
| `SAVE_SCAN_BEGIN` | `0x7001c00000` | same | Scan range for pointers into the live save (runtime tool, not a game object). |
| `SAVE_SCAN_END` | `0x7001d00000` | same | End of that range. |
| `MAIN_CODE_SCAN_END` | `0x7001600000` | same | Upper bound used to treat a value as a main code pointer. |
| `MAIN_IMAGE_SCAN_END` | `0x7002000000` | same | Upper bound of the main image (vtables / rodata lie between code end and here). |
| `MAIN_CODE_SIZE` | `0x00016a0000` | same | Size of main's code (.text); not an address. Vtables are above main+this. |
| `GFX_CMDBUF_MEMORY_EVENT` | `0x700113e790` | not mapped | nn::gfx command buffer memory event: the NVN callback that passes it to the buffer's own handlers. |


### 8.2 Patch and hook sites

Every entry of `runtime/game_addresses/patches_v1_0_1.h`, grouped by feature; the purpose is from `tools/patches.py`
(and `docs/coop.md` for the co-op sites). "target" / "return" rows are jump targets and return addresses used by a
site, not patch points. The instruction is what `tools/patches.py` asserts is there.

#### Widescreen, field of view, culling, sky

| Name | 1.0.1 | 1.0.0 | Instruction / kind | What it is |
|---|---|---|---|---|
| `ASPECT_LITERAL` | `0x70016afba4` | same | data | 16:9 float literal loaded as the default projection aspect. |
| `ASPECT_SITES_0` | `0x700001092c` | same | `ldr sN, [xM, #0xba4]` | A 3D camera loading that 16:9 literal (site id 0). |
| `ASPECT_SITES_1` | `0x7000010b58` | same | `ldr sN, [xM, #0xba4]` | A 3D camera loading that 16:9 literal (site id 1). |
| `ASPECT_SITES_2` | `0x70000c1c84` | same | `ldr sN, [xM, #0xba4]` | A 3D camera loading that 16:9 literal (site id 2). |
| `ASPECT_SITES_3` | `0x70000c3228` | same | `ldr sN, [xM, #0xba4]` | A 3D camera loading that 16:9 literal (site id 3). |
| `ASPECT_SITES_4` | `0x70000c3edc` | same | `ldr sN, [xM, #0xba4]` | A 3D camera loading that 16:9 literal (site id 4). |
| `ASPECT_SITES_5` | `0x700085f824` | same | `ldr sN, [xM, #0xba4]` | A 3D camera loading that 16:9 literal (site id 5). |
| `CULL_ASPECT` | `0x7000852ad4` | same | `ldr s9, [x27, #0x570]` | Culling pass (main+0x8529b0, x27 view camera) loads the camera aspect (+0x570). |
| `FOVY_SITES_0` | `0x7000010914` | same | `ldr s0, [x23, #0x564]` | Main camera loads its vertical fov (camera +0x564, radians). |
| `FOVY_SITES_1` | `0x7000852ad0` | same | `ldr s8, [x27, #0x564]` | Culling pass loads the vertical fov. |
| `INDOOR_SITE` | `0x7001023d40` | +0xd0 | `ldr s1, =16/9` | Indoor shadow set-up (community note "Indoor Shadow AR"): runs while an interior renders. |
| `SKYSEA_BOX_MAX_SITE` | `0x70008fb1b8` | same | `ldr q1, [x23, #0x2e0]` | MapSkySea piece box max (pieces 60 apart; drawn while Link is inside centre +-(67.5, 50, 27)). |
| `SKYSEA_BOX_MIN_SITE` | `0x70008fb1c4` | same | `ldr q2, [x23, #0x2d0]` | Same, box min. |

#### Levels, warps, chamber dungeons

| Name | 1.0.1 | 1.0.0 | Instruction / kind | What it is |
|---|---|---|---|---|
| `WARP_REQUEST_SITES_0` | `0x700080c464` | same | `str x20, [x8, #0x1c0]` | The level-jump request object registers itself in its parent while a level loads (scene kind 1). |
| `WARP_REQUEST_SITES_1` | `0x700080d718` | same | `str x19, [x8, #0x1c0]` | Same, the other scene kind. |
| `LEVEL_REQUEST_SITE` | `0x700081ba80` | same | `sub sp, sp, #0x80` | Prologue of the level-jump request processor (x0 = request; every level change passes here). |
| `ADV_EXIT_DIALOG_SITE` | `0x70007f15e0` | same | `bl` | Chamber staircase exit prompt: main+0x7f1540 opens it here; callback stores the choice at [CHAMBER_MGR]+0x18f8 (0 = leave). |
| `ADV_EXIT_DIALOG_TARGET` | `0x7000f05350` | +0xd0 | target | The prompt dialog function called from that site. |
| `ADV_EXIT_DIALOG_RETURN` | `0x70007f15e4` | same | return | Return address after that call. |

#### Damage and health

| Name | 1.0.1 | 1.0.0 | Instruction / kind | What it is |
|---|---|---|---|---|
| `DAMAGE_LSL_SITES_0` | `0x7000d7c8c0` | same | `lslv` | damage << hero (PlayerStatus+0x120): an event's damage command (main+0xd7c870); no Link in a register (always Player 1). |
| `DAMAGE_LSL_SITES_1` | `0x7000d9694c` | same | `lslv` | damage << hero in another Link-hit function (main+0xd96830); the Link is in x20. |
| `DAMAGE_LSL_SITES_2` | `0x7000d4c738` | same | `lslv` | damage << hero in the enemy-hit function; the Link is in x21. Then halved (see COOP_DAMAGE_HALVED_SITE) and subtracted from PlayerStatus+0x84. |
| `DAMAGE_FIXED_SITE` | `0x7000db1f6c` | same | `csel w20, w10, w9, eq` | Fixed damage -2 quarter hearts (-4 Hero Mode): falls into pits / deep water; Link in x19. |

#### Items, inventory, menus, HUD, text

| Name | 1.0.1 | 1.0.0 | Instruction / kind | What it is |
|---|---|---|---|---|
| `ADD_ITEM_SITE` | `0x70008d08e0` | same | `sub sp, sp, #0x50` | Prologue of the inner add-item (34 callers: events, chests, drops; AddItem wraps it). Dungeon items go to the current dungeon only. |
| `PRIZE_ADD_SITE` | `0x70007e1d10` | same | `mov w2, w20` | In AddPrizeByKey (main+0x7e1c70): about to call AddPrize(row, count, index) (main+0x1489c0; rows 0-20 only). |
| `HAS_ITEM` | `0x70008d1a30` | same | function | HasItem(x0 item, x1 count, x2 index). Called with 0x6f / 0x70 for the power-up buffs (powerup_music.c); 0x6f is the halving check in the enemy-hit path. |
| `HEART_PIECE_ADD` | `0x700089b960` | same | function | Called by the heart-piece actor's pickup (main+0x855a0) to add a heart piece; `patches.py` also calls main+0x89b960 the ItemDrop "drop roll" (BUFF_REPLACE_SITE is inside it). |
| `BUFF_REPLACE_SITE` | `0x700089ba34` | same | `ldrb w8, [x24, #0x38]` | Drop roll reads its ItemDrop row's mAllowBuffReplace (+0x38): set, kill counters may turn the drop into a Piece of Power / Guardian Acorn. |
| `TUNIC_CLEAR_SITES_0` | `0x70008d1118` | same | `and w9, w9, #0xffffffbf` | Add-item: blue tunic set, red (0x40) cleared from PlayerStatus+0x9c. |
| `TUNIC_CLEAR_SITES_1` | `0x70008d12f8` | same | `and w9, w9, #0x7f` | Add-item: red tunic set, blue (0x80) cleared. |
| `UNDERWATER_OWNED_SITE` | `0x70008c45b8` | same | `bl HasItem` | Item actor's "should I appear?" (main+0x8c4510): HasItem(item +0x2e8, 1, index +0xbc); x20 = the item actor. |
| `UNDERWATER_OWNED_RETURN` | `0x70008c45bc` | same | return | Return address after that call. |
| `UNDERWATER_VIEW_SITE` | `0x7000ff5590` | +0xd0 | `cset w27, ne` | Model set-up reads a material's `under_water` shader option (only such models show in the underwater view). |
| `UNDERWATER_PICKUP_SITE` | `0x70000856e4` | same | `bl 0x89b960` | Heart-piece actor's pickup (main+0x855a0, x19 = actor) adds a heart piece. |
| `UNDERWATER_PICKUP_RETURN` | `0x70000856e8` | same | return | Return address after that call. |
| `ITEM_EXPLANATION_SITE` | `0x7000eb6790` | +0xd0 | `ldr x22, [sp, #0xf0]` | Items page: the "UI:ItemExplanation_{n}" message just looked up (null = no text path). |
| `ITEM_NAME_SITE` | `0x7000eb6728` | +0xd0 | `bl` | Items page: sets the item-name pane from "UI:ItemName_{0}". |
| `ITEM_NAME_TARGET` | `0x7000f8c980` | +0xd0 | target | The name-pane setter called from that site. |
| `ITEM_NAME_RETURN` | `0x7000eb672c` | +0xd0 | return | Return address after it. |
| `ITEMS_PAGE_SITE` | `0x7000eb7da0` | +0xd0 | `sub sp, sp, #0x90` | Prologue of the pause menu Items page update (once per frame only while that page shows). x0 = page; +0x4020 u32 cursor (0-11), cells +0x598 + 0x280 * cell = item slot. |
| `PAUSE_MENU_SITE` | `0x7000ee7050` | +0xd0 | `sub sp, sp, #0xe0` | Prologue of the pause menu update (every page and between them). |
| `ITEMS_NAV_SITE` | `0x7000eb80ac` | +0xd0 | `ldr w22, [x8]` | Items page cursor move: loads the cell the direction leads to (neighbour table; 0x27 = none). |
| `FILE_SELECT_NAV_SITE` | `0x7000e6115c` | +0xd0 | `tbnz w0, #0` | File select (each frame): after the list helper moved the cursor byte (+0x159b0; previous +0x159b1). |
| `MAP_PAGE_SITE` | `0x7000e938d0` | +0xd0 | `sub sp, sp, #0x80` | Prologue of the pause menu Map page update (only while it shows). |
| `MAP_PAGE_END_SITE` | `0x7000e93c50` | +0xd0 | `add sp, sp, #0x80` | Its epilogue (layout updated this frame). |
| `DGN_MAP_SITE` | `0x7000e78810` | +0xd0 | `sub sp, sp, #0x90` | Prologue of the dungeon map page update (x0 = page). |
| `DGN_MAP_END_SITE` | `0x7000e78938` | +0xd0 | `add sp, sp, #0x90` | Its epilogue. |
| `HUD_HEARTS_SITE` | `0x7000eaa8d0` | +0xd0 | `sub sp, sp, #0xb0` | Prologue of the HUD's normal state update (x0 = the HUD). |
| `PROMPT_POS_SITE` | `0x7000eab208` | +0xd0 | `stp s2, s0, [x8, #0x30]` | HUD update places the in-world action prompt pane (translate x, y at pane +0x30/+0x34). |
| `BOTTLE3_USE_SITE` | `0x7000d50f0c` | same | `cset w0, eq` | The bottle-use prompt's predicate of the third bottle (slot 0xf): owned = PlayerStatus+0xa9 bit 7, empty = +0xc0 bit 2 clear. |
| `BOTTLE1_USE_SITE` | `0x7000d5116c` | same | `cset w0, eq` | Same for the first bottle (slot 0xd): owned = +0xa9 bit 5, empty = +0xc0 bit 0. |
| `EQUIP_Y_TEST_SITES_0` | `0x7000d50e94` | same | `tbnz w8, #3` | Items page item use check 0: Y newly pressed? |
| `EQUIP_Y_TEST_SITES_1` | `0x7000d50fc4` | same | `tbnz w8, #3` | Items page item use check 1: Y newly pressed? |
| `EQUIP_Y_TEST_SITES_2` | `0x7000d510f4` | same | `tbnz w8, #3` | Items page item use check 2: Y newly pressed? |
| `EQUIP_Y_TEST_SITES_3` | `0x7000d51558` | same | `tbnz w8, #3` | Items page item use check 3: Y newly pressed? |
| `EQUIP_Y_TEST_SITES_4` | `0x7000d51dc8` | same | `tbnz w8, #3` | Items page item use check 4: Y newly pressed? |
| `EQUIP_Y_TEST_SITES_5` | `0x7000d51f64` | same | `tbnz w8, #3` | Items page item use check 5: Y newly pressed? |
| `EQUIP_Y_TEST_SITES_6` | `0x7000d52088` | same | `tbnz w8, #3` | Items page item use check 6: Y newly pressed? |
| `EQUIP_Y_TEST_SITES_7` | `0x7000d528b8` | same | `tbnz w8, #3` | Items page item use check 7: Y newly pressed? |
| `EQUIP_Y_TEST_SITES_8` | `0x7000d52af4` | same | `tbnz w8, #3` | Items page item use check 8: Y newly pressed? |
| `EQUIP_Y_TEST_SITES_9` | `0x7000d52c18` | same | `tbnz w8, #3` | Items page item use check 9: Y newly pressed? |
| `EQUIP_Y_TEST_SITES_10` | `0x7000d52d64` | same | `tbnz w8, #3` | Items page item use check 10: Y newly pressed? |
| `EQUIP_Y_TEST_SITES_11` | `0x7000d52f34` | same | `tbnz w8, #3` | Items page item use check 11: Y newly pressed? |
| `EQUIP_Y_READ_SITES_0` | `0x7000d50ed8` | same | `ldrb wN, [x8, #0xa2]` | Check 0: the item on Y (PlayerStatus+0xa2) compared with slot 0xf. |
| `EQUIP_Y_READ_SITES_1` | `0x7000d51008` | same | `ldrb wN, [x8, #0xa2]` | Check 1: the item on Y (PlayerStatus+0xa2) compared with slot 0xe. |
| `EQUIP_Y_READ_SITES_2` | `0x7000d51138` | same | `ldrb wN, [x8, #0xa2]` | Check 2: the item on Y (PlayerStatus+0xa2) compared with slot 0xd. |
| `EQUIP_Y_READ_SITES_3` | `0x7000d5159c` | same | `ldrb wN, [x8, #0xa2]` | Check 3: the item on Y (PlayerStatus+0xa2) compared with slot 0x6. |
| `EQUIP_Y_READ_SITES_4` | `0x7000d51e0c` | same | `ldrb wN, [x8, #0xa2]` | Check 4: the item on Y (PlayerStatus+0xa2) compared with slot 0x5. |
| `EQUIP_Y_READ_SITES_5` | `0x7000d51fa8` | same | `ldrb wN, [x8, #0xa2]` | Check 5: the item on Y (PlayerStatus+0xa2) compared with slot 0x4. |
| `EQUIP_Y_READ_SITES_6` | `0x7000d520cc` | same | `ldrb wN, [x8, #0xa2]` | Check 6: the item on Y (PlayerStatus+0xa2) compared with slot 0x3. |
| `EQUIP_Y_READ_SITES_7` | `0x7000d52900` | same | `ldrb wN, [x8, #0xa2]` | Check 7: the item on Y (PlayerStatus+0xa2) compared with slot 0x2. |
| `EQUIP_Y_READ_SITES_8` | `0x7000d52b38` | same | `ldrb wN, [x8, #0xa2]` | Check 8: the item on Y (PlayerStatus+0xa2) compared with slot 0x7. |
| `EQUIP_Y_READ_SITES_9` | `0x7000d52c5c` | same | `ldrb wN, [x8, #0xa2]` | Check 9: the item on Y (PlayerStatus+0xa2) compared with slot 0xc. |
| `EQUIP_Y_READ_SITES_10` | `0x7000d52da8` | same | `ldrb wN, [x8, #0xa2]` | Check 10: the item on Y (PlayerStatus+0xa2) compared with slot 0x8. |
| `EQUIP_Y_READ_SITES_11` | `0x7000d52f78` | same | `ldrb wN, [x8, #0xa2]` | Check 11: the item on Y (PlayerStatus+0xa2) compared with slot 0x9. |
| `CHOICE_COUNT_SITE` | `0x7000ef4a98` | +0xd0 | `str w8, [x19, #0x1cc4]` | Message window question set-up: answer count (2..4 in the game's own text) stored at window +0x1cc4. |
| `TEXT_RATE_SITE` | `0x7000f53d90` | +0xd0 | `str w8, [x0, #0x284]` | A message window opening sets the pane's text step to 2 (pane +0x284; x pane +0x280 frames per character unit). |
| `TEXT_SNAP_SITE` | `0x7000f53638` | +0xd0 | `b.lt` | Message pane update: text frame still short of the page's end. |

#### Actors, spawning, field objects

| Name | 1.0.1 | 1.0.0 | Instruction / kind | What it is |
|---|---|---|---|---|
| `BOSS_SPAWN_SITE` | `0x70008f5ed4` | same | `ldr x24, [x8]` | Actor factory: loads the type's create fn (x8 = x27 + type*0x58, x27 = ACTOR_TYPES, x23 = type, x2 = spawn info). Null fn = the factory's "no such actor" path. |
| `MB_SEEN_SITE` | `0x70000ec354` | same | `ldr x8, [x21]` | In f_70000ec310 (a component's per-frame update), which asks its actor (x21) a type question through vtable +0x68 every frame: sees every actor. |
| `ACTOR_EVENT_SITE` | `0x700094a568` | same | `bl main+0x80c290` | An actor's AI starting one of its events (main+0x94a510; x20 = AI, [AI+0x58] = actor, entry name at [x29-0x40] {ptr, len}), right before the event request is built. |
| `ACTOR_EVENT_SKIP` | `0x700094a7e4` | same | target | That function's no-event return. |
| `DIG_HOLE_SITE` | `0x7000ddc5cc` | same | `ldr x22, [sp, #8]` | Shovel dig commit (f_7000ddc530) right after the hole spawn: hole at [sp+0x10], x21 its transform (+0x30 ground point on the tile centre), x20 +0xb0 zone id. |
| `HOLE_CAP_SITES_0` | `0x7000d22d30` | same | `cmp x8, #0x51` | Dug-hole list limit (fills in the oldest beyond 80; list at main+0x1cbae80, count +0x10). |
| `HOLE_CAP_SITES_1` | `0x7000d22dc0` | same | `cmp x8, #0x50` | Same. |

#### Link: movement and items

| Name | 1.0.1 | 1.0.0 | Instruction / kind | What it is |
|---|---|---|---|---|
| `HOOKSHOT_ATTR_SITE` | `0x7000d7ee9c` | same | `ldr w23, [sp, #0x38]` | Hookshot Shot update (main+0xd7ecc0) loads the hit surface's attribute word; bit 0x4000 = hookshot target (Adhere), else Return. Hit record at [sp+0xa8]. |
| `HOOKSHOT_DIR_SITE` | `0x7000dbb924` | same | `str q0, [x19, #0x2f0]` | Link's hookshot action entering Start (main+0xdbb760) stores the flat firing direction it gives the hook (main+0xd82a80). x19+0x210 = Link. |
| `SNAP_DIR_SITE` | `0x7000ded9f0` | same | `sub sp, sp, #0x40` | Prologue of the 8-way stick snap (s1 x, s2 y) -> 32-bit angle (full turn 2^32, 0 = down). |
| `WALK_SPEED_SITES_0` | `0x7000dcdea4` | same | `ldr x8, [x19, #0x1948]` | Walk state (main+0xdcdae0): speed multiplier in s0 (1.0, 1.15 with a Piece of Power) just before this load. |
| `WALK_SPEED_SITES_1` | `0x7000dcff24` | same | `ldr x8, [x19, #0x1948]` | Same, second path. |
| `WALK_SPEED_SITES_2` | `0x7000db80a8` | same | `ldr x8, [x20, #0x1948]` | Shield walk (main+0xdb7d30), same multiplier. |
| `WALK_SPEED_SITES_3` | `0x7000db8eac` | same | `ldr x8, [x20, #0x1948]` | Same, second path. |
| `WALK_ANIM_SITE_0` | `0x7000dce320` | same | `ldr x8, [x19, #0x1568]` | Walk state: walk animation play rate in s8 (1, 0.75 slow ground, x1.15 PoP). +0x1568 = animation controller. |
| `WALK_ANIM_SITE_1` | `0x7000db8168` | same | `ldr x8, [x22, #8]` | Shield walk: animation rate in s0. |
| `WIND_PUSH_SITE` | `0x7000d5749c` | same | `ldrb w8, [x19, x8]` | Link's move (main+0xd57260, every state): frame displacement summed at [x29-0x80] (x19 Link, s1 dt), about to go to the collision move. |

#### Mini-bosses outside their arenas (enemy randomizer)

| Name | 1.0.1 | 1.0.0 | Instruction / kind | What it is |
|---|---|---|---|---|
| `CRASHOUT_SITE` | `0x7000d97e2c` | same | `str wzr, [x0, #0xb8]` | Blaino knock-out: countdown object (main+0xd97df0, +0xb8) starts MightPunchCrashOut (fade, restart warp). |
| `CRASHOUT_SKIP` | `0x7000d97ea4` | same | target | Skip target. |
| `GOHMA_HEADING_SITES_0` | `0x70003996e4` | same | `str w8, [x19, #0x15ec]` | Gohma move heading (+-90 deg along its track; then the same to +0x15f0). |
| `GOHMA_HEADING_SITES_1` | `0x700039a2c4` | same | `str w8, [x19, #0x15ec]` | Same, second move. |
| `GOHMA_ARRIVED_SITE` | `0x700039957c` | same | `ldr q0, [x19, #0x1600]` | Track end vs Gohma's spot (main+0x399550 snaps onto it). |
| `GOHMA_ARRIVED_SKIP` | `0x70003995c0` | same | target | Skip target. |
| `MSTALFOS_BODY_SITE` | `0x70004b47c8` | same | `mov w8, #1` | Master Stalfos' CharaBodyCollision (main+0x1841c0 from his ctor): its "blocks Link" flag (x6 -> sp+0x328). |
| `MSTALFOS_JUMP_SITE` | `0x70004bde08` | same | `fsub s0, s0, s1` | Master Stalfos jump (state 4 exec): vertical speed = 6*sin(phase) - height (arc from height 0). |
| `MCHIEF_STORY_SITE` | `0x70001f8b30` | same | `tbnz w8, #7` | Moblin Chief init (main+0x1f8af0): deletes himself unless PlayerStatus+0xac bit 7 (story bit). |
| `MCHIEF_STORY_SKIP` | `0x70001f8b58` | same | target | Skip target. |
| `SHADOW_LINK_APPEAR_SITE` | `0x7000d15828` | same | `ldr x9, [x0, #0x230]` | Shadow Link's Appear event request (main+0xd157f0, within 3 units; ShadowLink.bfevfl), right after its already-requested check. |
| `SHADOW_LINK_APPEAR_SKIP` | `0x7000d15814` | same | target | Its return. |
| `SHADOW_LINK_BGM_SITE` | `0x7000d11230` | same | `bl main+0xce8ce0` | Shadow Link starts the chamber music (BGM_PANEL_SHADOW_LINK; held until his defeat calls main+0xce8f00). |

#### Audio

| Name | 1.0.1 | 1.0.0 | Instruction / kind | What it is |
|---|---|---|---|---|
| `BUFF_MUSIC_PICKUP_SITE` | `0x7000d79774` | same | `bl main+0xaf560` | Guardian Acorn / Piece of Power pickup starts BGM_NUTS directly. |
| `BUFF_MUSIC_FLAGS_SITES_0` | `0x70000a1638` | same | `ldr w21, [x8, #0x1b8c]` | Music update reads Link +0x1b8c (power-up sound-disable bit 0x10000). |
| `BUFF_MUSIC_FLAGS_SITES_1` | `0x70000a67d8` | same | `ldr w20, [x8, #0x1b8c]` | Zone music, same flags. |
| `BUFF_MUSIC_FLAGS_SITES_2` | `0x7000805acc` | same | `ldr w21, [x8, #0x1b8c]` | Event music, same flags. |
| `BUFF_MUSIC_FLAGS_SITES_3` | `0x7000d6d11c` | same | `ldr w21, [x8, #0x1b8c]` | Power-up presentation, same flags. |

#### Engine: frame pacing, resolution, post-process

| Name | 1.0.1 | 1.0.0 | Instruction / kind | What it is |
|---|---|---|---|---|
| `FRAME_STEPS_SITE` | `0x700000c954` | same | `ldr w8, [x8, x9]` | Frame timer (f_700000c910) reads the swap interval from the system object (+0x812c; written only by f_7000017800). Feeds the game's dynamic resolution budget, not gameplay speed. |
| `GAME_DYNRES_SITE` | `0x700000c938` | same | `ldrb w8, [x23, #0x648]` | Game dynamic-resolution scaler enabled byte (renderer +0x648 object). |
| `GAME_DYNRES_INIT_SITE` | `0x700107cd38` | +0xd0 | `ldr w12, [x12, x13]` | Set-up reads the boot default level (renderer +0x6e4c; 1080 - 36 x level lines at 16:9). |
| `TILT_INIT_SITE` | `0x70010712f0` | +0xd0 | `sub sp, sp, #0x90` | Tilt-shift post-process set-up (once at boot), x0 = the tilt-shift object (post-process manager +0x620). |
| `RES_NAME_SITE` | `0x70010df000` | +0xd0 | `sub sp, sp, #0x140` | Builds a resource handle from a name (x2 -> {char *name, ...}) and looks it up. |

#### Engine: crash and hang guards (recompiled code cannot unwind C++ exceptions)

| Name | 1.0.1 | 1.0.0 | Instruction / kind | What it is |
|---|---|---|---|---|
| `MISSING_STRING_SITE` | `0x700109fd78` | +0xd0 | `mov w0, #0x10` | String event-parameter lookup throws when missing (allocates the exception here). |
| `MISSING_STRING_RETURN` | `0x700109fd48` | +0xd0 | target | Where the hook resumes with "" returned. |
| `EMPTY_STRING` | `0x70016cf23a` | same | data | An empty string (the terminator before a message) used as that "". |
| `MISSING_FLOAT_SITE` | `0x700109fbe8` | +0xd0 | `mov w0, #0x10` | Float event-parameter lookup throws when missing; reader main+0x7f3c90 treats time <= 0 as instant. |
| `NO_HANDLER_SITES_0` | `0x700108ee98` | +0xd0 | `mov w0, #8` | Event action dispatch through the actor's handler map: empty std::function -> bad_function_call. |
| `NO_HANDLER_SITES_1` | `0x700108f010` | +0xd0 | `bl __cxa_begin_catch` | Its catch block. |
| `NO_HANDLER_SITES_2` | `0x700108f144` | +0xd0 | `bl __cxa_end_catch` | Its catch block end. |
| `NO_HANDLER_CATCH` | `0x700108f00c` | +0xd0 | target | The catch block (aborts the event). |
| `EMPTY_HANDLER_SITE` | `0x700108df18` | +0xd0 | `bl` std::function call | Event dispatcher calling an event handler (throws when empty). |
| `EMPTY_HANDLER_CATCH` | `0x700108ded0` | +0xd0 | target | Where its catch goes (returns 0, "not handled"). |
| `DELEGATE_FORWARDER_SITES_0` | `0x70010ccf20` | +0xd0 | `mov x1, x0` | Slot +0x28 of DELEGATE_JOB_VTABLE_BASE class: forwards to [this+0x10] unchecked. |
| `DELEGATE_FORWARDER_SITES_1` | `0x70010ccf00` | +0xd0 | `mov x1, x0` | Slot +0x30, same. |
| `FRAME_JOB_KICKOFF_SITES_0` | `0x7000015780` | same | `add x19, x22, #0x1e0` | Per-frame kick-off (main+0x15730, four inlined copies) rebuilds owner X's frame job J (X+0xc8). |
| `FRAME_JOB_KICKOFF_SITES_1` | `0x7000015834` | same | `add x19, x22, #0x1e0` | Same, copy 2. |
| `FRAME_JOB_KICKOFF_SITES_2` | `0x70000158e8` | same | `add x19, x22, #0x1e0` | Same, copy 3. |
| `FRAME_JOB_KICKOFF_SITES_3` | `0x70000159b0` | same | `add x19, x21, #0x1e0` | Same, copy 4. |
| `FRAME_JOB_FINISH_SITE` | `0x70010cd070` | +0xd0 | `ldr x8, [x0]` | Frame job finish (-> slot +0x00: base vtable, delegate cleared). |
| `OWNER_CALL_NOTIFY_SITE` | `0x7001049fd8` | +0xd0 | `add x0, x21, #0x38` | A call run on its object's owner thread: task stores the result at O+0x10 and notify_one on O+0x38 without taking O's mutex (O+0x18). |
| `FONT_BOX_GLYPH_SITE` | `0x700111e0d0` | +0xd0 | `ldrb w8, [x0, #0x38]` | Font set-up marks the "box" glyph U+25A1 without checking it exists. |
| `FONT_BOX_GLYPH_SKIP` | `0x700111e13c` | +0xd0 | target | Function end. |
| `FONT_FALLBACK_SITES_0` | `0x700111e330` | +0xd0 | read of the box glyph | A missing character is drawn / measured as the box glyph (font +0x34): its width function reads it unchecked. |
| `FONT_FALLBACK_SITES_1` | `0x700111e3ec` | +0xd0 | read of the box glyph | Its widths function, same. |
| `FONT_FALLBACK_SITES_2` | `0x700111e570` | +0xd0 | read of the box glyph | Its glyph function, same (first read). |
| `FONT_FALLBACK_SITES_3` | `0x700111e574` | +0xd0 | read of the box glyph | Its glyph function, same (second read). The four reads are `ldrh w0,[x0,#0x4c]`, `mov x9,x0`, `ldrh w22,[x0,#0x4c]`, `ldrh w8,[x0,#0x4e]` (which site has which is not stated). |
| `SHAPE_SIM_SITES_0` | `0x7001408eb0` | +0xd0 | `mov x20, x0` | PhysX RigidCore::onShapeChange after RigidSim::getSimForShape (returns address 1 when the shape has no sim object). |
| `SHAPE_SIM_SITES_1` | `0x7001408e50` | +0xd0 | `mov x1, x0` | Its sibling, same lookup. |
| `SHAPE_SIM_SKIP_0` | `0x7001408edc` | +0xd0 | target | Function return. |
| `SHAPE_SIM_SKIP_1` | `0x7001408e68` | +0xd0 | target | Function return. |
| `EFFECT_BONE_SITE` | `0x7000e44e48` | +0xd0 | `ldr x0, [x22, #0x50]` | Bone-following effect emitter (kind 1) update reads its bone node (emitter +0x50; null if the actor had no model instance when made). |
| `EFFECT_BONE_SKIP` | `0x7000e444cc` | +0xd0 | target | The kind-0 path (actor's own transform). |

#### Local co-op hook sites (1.0.1 / 1.1.0 only; all 0x0 in 1.0.0)

| Name | 1.0.1 | 1.0.0 | Instruction / kind | What it is |
|---|---|---|---|---|
| `COOP_CREATE_SITE` | `0x700083a470` | not mapped | `str x28, [sp, #-0x40]!` | Player creator (an event action) first instruction: makes Link, sets him as the game manager's player, switches the scene camera to his. |
| `COOP_CREATE_RET` | `0x700083a728` | not mapped | target | The creator's `ret`. |
| `COOP_ZONE_SITE` | `0x700083a678` | not mapped | `bl main+0x83afa0` | The creator's zone set-up (returns a shared_ptr in x0's {ptr, control}, released by the creator). |
| `COOP_DUP_SITE` | `0x700083ab64` | not mapped | `cbnz x0` | Creator's "an actor of that name exists already" check. |
| `COOP_PLACE_SITE` | `0x700085d5f0` | not mapped | `stp x22, x21, [sp, #-0x30]!` | Placement of Link at a spawn point (x0 = the spot). |
| `COOP_PLACE_RET` | `0x700085d718` | not mapped | target | Its `ret`. |
| `COOP_PLACE_PLAYER_SITE` | `0x700085d610` | not mapped | `ldr x20, [x19, #0x4b8]` | The Link the placement moves (x19 = game manager). |
| `COOP_SPAWN_CHECK_SITE` | `0x70008ec4f8` | not mapped | `ldr x0, [x22, #0x208]` | A spawn check reading an object (x22) that can be null. |
| `COOP_SPAWN_CHECK_SKIP` | `0x70008ec508` | not mapped | target | Skip target. |
| `COOP_CTRL_SITE` | `0x7000d35ae4` | not mapped | `ldp x22, x26, [x8]` | Link's controller shared_ptr from the npad list (16-byte entries; x19 = Link). |
| `COOP_COLOR_SITE` | `0x7000d35760` | not mapped | `blr x8` | Link's tunic colour frame (s0: 0 green, 1 red, 3 blue; x19 = Link). |
| `COOP_ENTER_SITE` | `0x7000dd4950` | not mapped | `sub sp, sp, #0xa0` | Door walk-in (x0 its state, x1 Link) prologue. |
| `COOP_ENTER_RET` | `0x7000dd501c` | not mapped | target | Its `ret`. |
| `COOP_SCROLL_SITE` | `0x70000bc628` | not mapped | `str x8, [sp]` | Builds the event entry name "ZeldaScroll" (the scroll walk into a new camera area). |
| `COOP_PAD_STATE_SITE` | `0x7000015e5c` | not mapped | `b.ne` | Controller manager update (x19): state (+0x4e0) not "polling". Player limit at +0x4e8. |
| `COOP_APPLET_SITE` | `0x7000881ebc` | not mapped | `strb w20, [sp, #0x11]` | Controller applet arguments: player count. |
| `COOP_APPLET_SINGLE_SITE` | `0x7000881ee4` | not mapped | `strb w8, [sp, #0x15]` | Controller applet arguments: single-player mode. |
| `COOP_CCT_SWEEP_SITE_0` | `0x70012a88c8` | not mapped | `cbz x8` | PhysX CCT SweepTest geometry loop: x28 a touched geometry, x8 its sweep fn (none = skip). |
| `COOP_CCT_SWEEP_SKIP_0` | `0x70012a89ac` | not mapped | target | Skip target. |
| `COOP_CCT_SWEEP_SITE_1` | `0x70012a89d0` | not mapped | `cbz x8` | Same, the other variant. |
| `COOP_CCT_SWEEP_SKIP_1` | `0x70012a8a70` | not mapped | target | Skip target. |
| `COOP_ACTOR_EXEC_SITE` | `0x7000facbf8` | not mapped | `blr x9` | Actors' per-frame update loop (main+0xfacb80): x0 an update callback, [x0+8] its actor; result std::future<void> written through x8. |
| `COOP_ACTOR_EXEC_DONE` | `0x7000facbfc` | not mapped | `ldr x8, [sp, #8]` | Right after that update. |
| `COOP_CAMERA_SITE` | `0x7000010908` | not mapped | `ldr x8, [x23, #0x2a8]` | View set-up (main+0x10540, once per drawn frame): camera x23's view matrix just set (+0x580..+0x5bf); x24 +0x20 the renderer's copy. |
| `COOP_HEAL_SITE_0` | `0x70008d0a5c` | not mapped | `strh w8, [x9, #0x84]` | Hearts after a heart pickup (w23 = total before the max cap). |
| `COOP_HEAL_SITE_1` | `0x70008a39d4` | not mapped | `strh w8, [x9, #0x84]` | Hearts after a healing amount. |
| `COOP_EXIT_TOUCH_SITE` | `0x7000076f5c` | not mapped | `ldrb w9, [x23, #0x43]` | AreaLevelOpen (door / loading zone) contact (main+0x76e90): touching player actor x23, its +0x43 bit 6; x20 = the exit. |
| `COOP_STAIRS_TOUCH_SITE` | `0x70000617e0` | not mapped | `ldrb w8, [x8, #0x43]` | AreaUpstairs / AreaDownstairs stepped on (main+0x617a0); x19 = the stairs. |
| `COOP_STAIRS_LEAVE_SITE` | `0x7000061640` | not mapped | `ldrb w8, [x8, #0x43]` | Stairs stepped off (main+0x61600). |
| `COOP_EXIT_LEAVE_SITE` | `0x7000076aa0` | not mapped | `stp x20, x19, [sp, #-0x20]!` | AreaLevelOpen contact-end functor (x1 = the contact). |
| `COOP_EXIT_LEAVE_CLEAR` | `0x7000076acc` | not mapped | `bl main+0x76b20` | Drops the pending exit. |
| `COOP_CAMERA_SCROLL_SITE` | `0x70000bbdcc` | not mapped | `ldrb w8, [x19, #0x515]` | A camera (x19) told of a new scroll area by the camera-area manager; +0x515 set = answer with "ZeldaScroll" (main+0xbc540). |
| `COOP_DOACTION_SITE` | `0x70000ea240` | not mapped | `sub sp, sp, #0x60` | DoAction component update (x0): checks itself against "Link" and may become the pending slot. |
| `COOP_DOACTION_RET` | `0x70000ea36c` | not mapped | target | Its `ret`. |
| `COOP_DOACTION_FRAME_SITE` | `0x70000e8be0` | not mapped | `stp x20, x19, [sp, #-0x20]!` | Frame end: pending DoAction becomes current; lowers the cooldown. |
| `COOP_DOACTION_GET_SITE` | `0x70000e8d90` | not mapped | `stp x20, x19, [sp, #-0x20]!` | Link asks for the current DoAction (x0 = where the weak_ptr is returned; x30 tells the caller). |
| `COOP_DOACTION_GET_RET` | `0x70000e8db8` | not mapped | target | Its `ret`. |
| `COOP_EVENT_BOX_SITE` | `0x7000067210` | not mapped | `str x19, [sp, #-0x20]!` | AreaEventBox fire (x0 box, x1 contact): no check of who touched it. |
| `COOP_EVENT_BOX_RET` | `0x700006724c` | not mapped | target | Its `ret`. |
| `COOP_WARP_HOLE_TOUCH_SITE` | `0x700008c5ac` | not mapped | `ldrb w8, [x8, #0x43]` | AreaWarpHoleEventBox stepped in (main+0x8c560): player actor x8. |
| `COOP_WARP_HOLE_LEAVE_SITE` | `0x700008c410` | not mapped | `ldrb w8, [x8, #0x43]` | Stepped out (main+0x8c3d0). |
| `COOP_DEAD_STATE_SITE` | `0x7000d34660` | not mapped | `sub sp, sp, #0x30` | State machine change(x0 machine, w1 state): prologue. Link's machine is Link+0x4e0; dead state 0x12 (0x13 checked with it). |
| `COOP_DEAD_STATE_RET` | `0x7000d3472c` | not mapped | target | Its `ret`. |
| `COOP_LOAD_BOX_SITE` | `0x700085c430` | not mapped | `adrp x23` | Level screens update (main+0x85c280): s1 x s2 = box whose screens' objects are made (table by level kind, main+0x1701e40; Field 75 x 60). |
| `COOP_KEEP_BOX_SITE` | `0x700085c628` | not mapped | `ld1 {v0.s}[1], [x23]` | Box of screens kept: s0 west / s1 east / s3 north / s2 south, offsets from Player 1 (west, north negative; Field 90 x 72). |
| `COOP_ACTIVE_BOX_SITE` | `0x700085c848` | not mapped | `fsub s8, s0, s3` | Box of active screens (ground solid, objects running): s3, s4 half sizes, s5 offset deep (main+0x1701ed4 by screen type). |
| `COOP_WARP_OBJ_TOUCH_SITE` | `0x7000cbd42c` | not mapped | `ldrb w8, [x8, #0x43]` | Overworld warp-hole object stepped in (main+0xcbd3e0: sets its +0x405). |
| `COOP_WARP_OBJ_LEAVE_SITE` | `0x7000cbd290` | not mapped | `ldrb w8, [x8, #0x43]` | Stepped out (main+0xcbd250). |
| `COOP_SHADOW_BOX_SITE` | `0x70000109b8` | not mapped | `stp q1, q0, [x8, #0x200]` | Shadow map world box around Player 1 stored in the view (+0x200 min, +0x210 max; copied to a second view at main+0x11228). |
| `COOP_TILE_PLAYER_SITE` | `0x7000b01c70` | not mapped | `ldr x8, [x20, #0x4b8]` | Collapse tile job (main+0xb01a40, x19 the tile): the Link it measures (drop within 1.35 across). |
| `COOP_LIKELIKE_SITE_0` | `0x70004a65f4` | not mapped | `ldr x22, [x21, #0x4b8]` | Like Like state handler reading "Link" (x19 = the Like Like; handlers run outside its update). |
| `COOP_LIKELIKE_SITE_1` | `0x70004a7040` | not mapped | `ldr x21, [x20, #0x4b8]` | Like Like state handler reading "Link" (x19 = the Like Like; handlers run outside its update). |
| `COOP_LIKELIKE_SITE_2` | `0x70004a7fb8` | not mapped | `ldr x9, [x20, #0x4b8]` | Like Like state handler reading "Link" (x19 = the Like Like; handlers run outside its update). |
| `COOP_LIKELIKE_SITE_3` | `0x70004a89f4` | not mapped | `ldr x21, [x20, #0x4b8]` | Like Like state handler reading "Link" (x19 = the Like Like; handlers run outside its update). |
| `COOP_LIKELIKE_SITE_4` | `0x70004a8ce8` | not mapped | `ldr x21, [x20, #0x4b8]` | Like Like state handler reading "Link" (x19 = the Like Like; handlers run outside its update). |
| `COOP_LIKELIKE_SITE_5` | `0x70004a9400` | not mapped | `ldr x8, [x20, #0x4b8]` | Like Like state handler reading "Link" (x19 = the Like Like; handlers run outside its update). |
| `COOP_LIKELIKE_SITE_6` | `0x70004a9898` | not mapped | `ldr x8, [x20, #0x4b8]` | Like Like state handler reading "Link" (x19 = the Like Like; handlers run outside its update). |
| `COOP_LIKELIKE_SITE_7` | `0x70004aac54` | not mapped | `ldr x8, [x21, #0x4b8]` | Like Like state handler reading "Link" (x19 = the Like Like; handlers run outside its update). |
| `COOP_LIKELIKE_SITE_8` | `0x70004ab1f0` | not mapped | `ldr x21, [x20, #0x4b8]` | Like Like state handler reading "Link" (x19 = the Like Like; handlers run outside its update). |
| `COOP_DAMAGE_HALVED_SITE` | `0x7000d4c750` | not mapped | `ldrh w8, [x8, #0x84]` | Enemy-hit function after the halving (flag 0x6f via HasItem, or PlayerStatus+0x9c negative): hit in w20, x8 PlayerStatus, x21 the Link. |
| `COOP_RESTART_SITE` | `0x7000d35778` | not mapped | `ldr x1, [x20, #0x4a8]` | Link update (main+0xd355c0, x19 Link) looks up the player entity to write the save's restart point. |
| `COOP_ACTOR_TYPE_SITE` | `0x700001be64` | not mapped | `strh w8, [x19, #0xa0]` | Actor base-class ctor (main+0x1bdb0) stores the type (key already at +0x98). |
| `COOP_ACTOR_GONE_SITE` | `0x7000fa7970` | not mapped | `stp x22, x21, [sp, #-0x30]!` | Actor base-class destructor prologue (x0 = actor). |
| `COOP_PART_SITE_0` | `0x700012fb5c` | not mapped | `str x8, [x19]` | Reaction component base ctor (main+0x12fae0) stores its vtable. |
| `COOP_PART_SITE_1` | `0x7000c53ac0` | not mapped | `str x8, [x20]` | Effect player (TailorPlayer) built in place stores its vtable. |
| `COOP_PART_SITE_2` | `0x7000e24c1c` | not mapped | `str x8, [x22]` | Same, the other construction. |
| `COOP_PART_GONE_SITE_0` | `0x70000eb410` | not mapped | `sub sp, sp, #0x30` | Reaction component destructor prologue. |
| `COOP_PART_GONE_SITE_1` | `0x7000e3b510` | not mapped | `sub sp, sp, #0x30` | Effect player destructor prologue. |
| `COOP_CULL_SITE` | `0x7000852ad8` | not mapped | `ldr s10, [x27, #0x568]` | Culling pass (once per world step) reads the view camera (x27); eye at [cam+0x510]+0xb0, far plane +0x56c read right after. |
| `COOP_JOIN_CAMERA_SITE` | `0x700083a5e4` | not mapped | `bl main+0xbfda0` | Creator switches the camera manager to the new Link's camera (x3 camera, shared_ptr back at sp+0x50). |


Plus 696 more patch sites not in the table: every `ldr xT, [xN, #0x4b8]` in main
(`coop_player_reads_v1_0_1.txt`, `<address> <instruction word> <gm|?>`), minus the 11 that are hooks above.

## 9. Open questions

1. **What does a level load do to every existing actor's collision that a spawn in play doesn't?** A Link (and his
   `PlayerHookshot`) made mid-level by the player creator walks and his sword hits, but his hookshot's attack
   collision never gets enemy contacts: the contact handler `0x70000d67d0` runs on the hook's component (ground,
   walls) but never on the enemy's damage component - the engine never delivers the pair. Any level load (a door, a
   warp) fixes it. Activation (virtual +0xc8, flags 0x700) is not it; a byte-for-byte snapshot of the hook, its
   component, collision object, shape, engine shape and engine actor shows no change across the fixing load.
   Leads: the engine shape's filter-data index (+0x54 / +0x64, a running count?), a per-level pair table or filter
   callback, or a scene registration that existing shapes are (re)added to at load time. Full write-up:
   `hookshot-drop-in-investigation.md` in this folder. More: a hook made at a load and switched off and on
   (virtual +0xc0 then +0xc8) still hits; one made in play and switched off and on still doesn't. The co-op fork
   now avoids it by making every player's Link as the level loads and keeping the unused ones parked, switched
   off and not updated until a player joins (2026-11-16).
2. Is there a function the game itself uses to register a newly made actor's collision with the current level?
3. Unconfirmed in 1.0.1: the 1.0.0-era function offsets marked `(inferred)` above; save code offsets in 2.3.
4. Not explained: Link +0x41, +0x1948; GAME_MGR +0x4a8 described both as the player entity and as the scene handle.
