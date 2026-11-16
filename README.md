# LAS internals

A shared map of *The Legend of Zelda: Link's Awakening* (Nintendo Switch) internals for people modding it - through
[LAS Recomp](https://github.com/Peiikai/LASRecomp) and its forks, or otherwise: addresses, struct offsets, what
functions and objects do, and the hook sites mods use. Facts about the program only - no game code, assets, text or
files.

Game version **1.0.1** unless a note says otherwise (1.1.0 uses the same addresses). Addresses are the main module's
virtual addresses as loaded at `0x7000000000` (`main+0x83a470` = `0x700083a470`); the SDK module is at
`0x7004000000`.

## Files

- [`hookshot-drop-in-investigation.md`](hookshot-drop-in-investigation.md) - an open question: a Link made mid-level
  (drop-in co-op) has a hookshot that passes through enemies until the next level load. What we traced, and
  what's still unknown.
- `las-game-internals.md` - the main reference (coming): singletons, the actor system, Link, collision, enemies,
  events, warps, HUD, audio, and every patch and hook site used by the
  [co-op fork](https://github.com/DrHammer79/LASRecomp).

## Adding to it

Add what you find, with where it came from (a function you read, a test you ran, a file in your project) so others
can check it. Mark guesses as guesses. Corrections welcome - open a pull request, or edit directly if you're a
collaborator.
