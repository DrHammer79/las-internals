# Drop-in co-op: a joined player's hookshot goes through enemies

LAS Recomp co-op fork (DrHammer79/LASRecomp, branch `coop-rebase`), game 1.0.1 (address table `v1_0_1`).
All addresses are main-module virtual addresses (`main+0x...` = `0x7000000000 + ...`). Findings from 2026-11-15/16.

## The bug

Drop-in co-op lets a controller join mid-level: its Plus held makes a new Link for that player with the game's
own player creator (the event action at `main+0x83a470`), run from our per-frame hook instead of during a level
load. That player can walk, swing his sword and kill enemies, but **his hookshot passes through enemies**.

| Case | Hook vs Octorok |
|---|---|
| Player 2 in from the level load (Link made by the creator as the level loads) | hits (stars, stunned) |
| Player 2 joined mid-level | passes through, hits the wall behind |
| Player 2 joined mid-level, Octorok made after the join | passes through |
| Player 2 joined mid-level, then any level load (a door, a warp) | hits |
| Player 2 joined mid-level, his **sword** | hits (kills) |

So it is not who the enemy targets, not enemy creation order, and not the Link's own attacks - only his
separate item actors (`PlayerHookshot`, almost certainly `PlayerBoomerang` too), and only until a level load.

Repro: `tools/coop_tests/hook_join.sh <id> join|start` (Field_12D, Mabe Village south: an Octorok on open grass at
20.25, 153.75; Player 1 put east of it with `LARECOMP_PLACE`, Player 2 west of it with `LARECOMP_COOP_MOVE`, his Y
every 30 frames). The Octorok wanders: the script prints the frames it is in line; look at
`build/frames/<id>_frame_<f>.png` for those.

## How the hook is made

- Each Link has his own hookshot and boomerang actors, made by the creator with him (`PlayerHookshot`, constructor
  `main+0xd7e780`; vtable `0x7001c1ab38`).
- The Link is made through the same spawn steps as any actor spawned in play: `main+0x83da50` makes it, its
  virtual `+0x78` and `+0x20` are called, flags `+0x40 |= 8`, then `main+0xfab540` hands it to the actor manager
  (singleton at `[0x7001cc6d98]`), which queues a registration command (`main+0xfae3a0`; the command's run
  function `main+0xfae760`: adds it to the manager's list, calls virtual `+0x28`, sets flag `0x10`, calls
  virtual `+0x50` once flags `& 0x17 == 0x17`). The same template as 412 other spawn sites (`main+0x30d9c` is
  one). `main+0xfb3360` then registers it by name in the container at `[game manager + 0x498]` (that pointer
  never changes).
- The hook's attack collision: `HookAttackColComp`, made at `main+0xd81244` (`main+0x43220` with
  `main+0xcd8c0`'s attack info), kept at the hook's `+0x2c8`; vtable `0x7001b5aef8` - the generic attack/damage
  collision component class (the Octorok's damage collision is the same class).

## What a level load does that a mid-level spawn doesn't (what we know)

1. **Activation.** A level load calls every actor's virtual `+0xc8` (`main+0x1e8d0` for most classes): flags
   `+0x40 |= 0x700`, then walks its components (`main+0xfa8f20` with the callback at `main+0x1f5b0`), setting
   theirs and handling three component types specially (one enables collision: `main+0x10d5990(..., 3, 0)`).
   A Link made in play had flags `0x10f`, his hook `0x11f`, its attack component `0x11d`; after a level load
   `0x71f` / `0x71d`. **We now call `+0xc8` on the joined Link and his items** - flags then match a level load's,
   but the hook still goes through. (Kept: it is a real gap.)
2. **Something else, outside the hook's objects.** A byte-for-byte snapshot of the joined hook, its attack
   component, its collision object (`[comp+0x170]`), its first shape (`[col+0x28]`), that shape's engine shape
   (`[shape+0x18]`, vtable `0x7001c8df60`) and engine actor (`[engine shape+0x28]`, vtable `0x7001c8c2a0`) shows
   **no change at all** across the level load that fixes it.

## Where the hit is lost

- The hook's ray (its Shot update `main+0xd7ecc0`, cast at `main+0xd69f10`, filter `[hook+0x4fc]` =
  `0x810200d5` for every hook) is for surfaces only; enemies are hit through the attack collision.
- Each attack/damage component registers a contact callback (function object, vtables `0x7001b5b150` /
  `0x7001b5b198`, call operator `main+0xd76c0`) that calls the component's contact handler **`main+0xd67d0`**.
- Working case: the handler runs on the hook's attack component **and** on the Octorok's damage component (the
  latter then calls the Octorok's check `main+0x554130` via its virtual `+0x68`).
- Joined case: the handler runs on the hook's component (it gets contacts - with the ground or walls) but
  **never on the Octorok's**: the hook-vs-Octorok pair is never delivered. (Calls on the hook's component are
  otherwise identical in both cases.)

So the collision engine never reports the pair (hook attack shape, enemy damage shape) for a hook made in play.

## The one difference found in the engine shape

The engine shape's filter data (at `+0x50` and a copy at `+0x60`): words `0x21`, an index, `0`, `0x80000001`.
Everything is the same between hooks except the index (`+0x54` / `+0x64`): Player 1's hook 9, Player 2's from
the load 6, Player 2's joined in play `0x69b`/`0x6c2`. It looks like a running count - a hook joined *before* a
level load got `0x235`, kept it through the load, and hit after it - so on its own it may not be the cause.
Overwriting both copies on a joined hook with a small value (9) didn't help, but that may not reach the
engine's internal copy made when the shape was added.

## Ruled out

- Placing the joined Link with the spawn placement's own call (`main+0xd337a0`, Player 1's transform as the spot).
- Re-running the registration steps after activation (virtual `+0x28`, `+0x50`), or `+0x90` / `+0x98`.
- "Link" (the game manager's player, `+0x4b8`) read as the wrong player: each player's own items now give him
  in their updates (`player_item_of`) - right, but not the cause.
- The attack component's `+0xe0` (an early lead: two different objects were being compared).
- The `player:hook` record's list at `+0x20`..`+0x30` (a growing pointer list, not remade by a load).
- Switching the joined Link's items off and on again later (virtual `+0xc0` then `+0xc8`, a hundred frames after
  the join): still goes through. The same done to a hook made at a load: still hits.

## How the co-op fork works around it (2026-11-16)

Every player's Link is made as the level loads (the player creator run for players 2..8 alongside Player 1);
those not playing are benched - parked at y -1000, switched off with their items (virtual `+0xc0`), and their
updates skipped (the Link's, his camera's at `+0x318`, his items'). Joining switches the benched Link on (virtual
`+0xc8`) and puts it beside Player 1; leaving benches it again. Checked: a player joined mid-level this way stuns
the Octorok, also after his benched Link sat through a level change; no load time measurable from the seven extra
Links; a Game Over removes and remakes them. So whatever a load gives actors made during it, a Link made then and
switched on later keeps it. What that is remains the open question below.

## Questions for someone who knows the engine

1. What does a level load do to every existing actor's collision that a spawn in play doesn't? Is there a
   per-level collision/contact-pair table, filter callback or "scene" registration that existing shapes are
   (re)added to at load time - and how do actors spawned in play (enemies, an Octorok's rock) get into it?
2. What is the filter-data index (`+0x54`) - an owner/collision id, and does the pair filter look it up in a
   table sized or filled at level load?
3. Is there a function the game itself uses to register a newly made actor's collision with the current level
   (one a mid-level spawn path calls that the player creator, normally run during a load, never needs)?

## Tools used (in the fork)

- `LARECOMP_FNPROF=<from>:<to>` function-entry counts (`runtime/fnprof.c`), `LARECOMP_FNPROF_BT=<addr>` first
  caller of a function; added temporarily: a filter that logs every function entered with `x0` a given object
  (or a component of it: `[x0+0x48]`), which found the contact handler difference.
- `LARECOMP_WATCH_ADDR` page-protect watch (`runtime/dispatch.c`): works, but collision pages are too busy (the
  game drops to under 1 fps).
- Memory scans for references to the hook's objects before/after a level load.
- `docs/coop.md` ("The hookshot and the boomerang") has the full notes.
