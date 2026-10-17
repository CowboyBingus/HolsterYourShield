# Holster Your Shield v1.0

Put your shield on your back without giving up your one-handed weapon. With the SH-20 Ballistic Shield or the
SH-51 Directional Shield worn, press **Backpack Function** and the shield goes onto your back while you keep your
pistol, SMG or other one-handed weapon in hand. Press it again to take the shield back into your hand.

**Status:** new in v1.0 and not yet played: it is built on the game's own code (the calls the game makes when you
switch to a two-handed weapon), checked offline against a simulation of the game's memory, and loaded live, but
the move itself has not been seen in a mission yet. Every press and its result go to the log (see below).

## What changes

- **Backpack Function with a shield in hand:** the shield goes onto your back, exactly as when you switch to a
  two-handed weapon, and stays there. When the game would bring it back (switching to a one-handed weapon,
  dropping an objective, the end of some actions), the mod puts it back on your back on the next frame.
- **Press again:** the shield returns to your hand if a one-handed weapon is in it and you carry nothing else.
  With a two-handed weapon out it simply returns the next time you take a one-handed weapon.
- **With the shield already on your back** (a two-handed weapon out), a press keeps it on your back when you
  switch to a one-handed weapon.
- **Carrying an objective:** the objective is never touched; a press only decides where the shield goes
  afterwards.
- The choice ends when you die, change backpacks, or if the game draws the shield seven times within two seconds
  (the mod then leaves it to the game rather than fight it).
- Nothing else changes. A Ballistic Shield on your back still blocks shots from behind, as in the base game. A
  Directional Shield on your back projects no barrier, as in the base game. Other backpacks are never touched,
  and the Backpack Function key keeps its normal use with them (the Supply Pack).

The key is the game's own Backpack Function action, so your rebinding and controller layout apply. There are no
options.

## Multiplayer

The mod tells the other players with the same message the game sends when you switch weapons, so they see your
shield where it is. Only your own Helldiver is ever changed. Whether everything looks right on other players'
screens has not been tested with a squad yet.

## Requirements and install

- Helldivers 2 Steam build **25480438** (EXE 1.8.46015.0). The mod checks the game.dll and EXE hashes and the
  first bytes of each game function it calls, and does nothing on any other build.
- [Bingus Shared Loader v18 or newer](https://github.com/CowboyBingus/BingusSharedLoader/releases/latest).

Import the release ZIP into Arsenal or HD2MM together with the loader, enable and deploy both, then restart the
game. Keep the shared loader as the winning Wwise startup replacement. The log is
`%LOCALAPPDATA%\CowboyBingus\Helldivers2\Logs\HolsterYourShield.log`: it names each press, each move and whether
the game took it, every refusal with its reason, and a summary when the game closes.

## How it works

With a shield in hand, the game holds it as your Helldiver's "held carryable" (the same slot an objective uses).
The game moves it with two of its own functions, one for the held item and one for the carry mode, and tells the
other players with one network message. The mod calls those same three functions, with the same values the game
uses when you switch to a two-handed weapon (to stow it) or to a one-handed weapon (to draw it). It writes no
memory itself. [docs/TECHNICAL.md](docs/TECHNICAL.md) has the addresses and the checks.

## Cost

Per frame: one memory read of your inventory (24 bytes), plus one read of the Backpack Function key while you
wear a shield. Every 30th frame about six more reads confirm the kept addresses still belong to you. A press first
finds everything again from the game's registries (about 40 to 50 reads), then makes three calls. No page queries, no
writes, no allocation per frame, and nothing compiled (the code stays interpreted). Not yet measured in game.

## Build and test

`python -B scripts/build.py` runs every test in a LuaJIT and in the game's own `lua51.dll`, then builds
`releases/Holster-Your-Shield-v1.0.zip`. The tests run the mod on a simulation of the game's memory: every press
case, re-stows and their limit, moved inventory rows, a respawn, refusals, the exact memory reads and calls of
every per-frame path, no allocation per idle frame, and a hostile shared Lua state.

## License

Zero-Clause BSD (0BSD): use, copy, modify and distribute for any purpose, with no conditions. See `LICENSE`.

**AI disclosure:** Claude Opus 5.5 assisted with research, implementation, tests, documentation and artwork.
