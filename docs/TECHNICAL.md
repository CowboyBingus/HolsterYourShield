# Holster Your Shield: how it works

Release v1.0 for Steam build 25480438, EXE 1.8.46015.0. Requires Bingus Shared Loader v18 or newer (API 1). Static
research on the decompiled `game.dll`; the chain to the inventory row and the equipment settings was read live on
the ship (2026-10-16). The move itself has not been seen in a mission yet.

## The shield in the game

Both shield backpacks are equipment type 25 (`EquipmentType_BackpackShield`), one-handed, with a hand node and a
back node in their EquipmentComponent; the Directional Shield's barrier is a separate entity its ShieldController
spawns. Equipment type is read from the game's live settings, so any other type-25 backpack is covered too.

The inventory `[game + 0x3326738]` keeps one 48-byte row per Helldiver (map +40 from the avatar's entity id, identity
pointers +64 = the avatar's entity record, rows +80):

| Row offset | Meaning |
| --- | --- |
| +0 / +4 / +8 | primary / sidearm / support weapon (entity ids) |
| +12 | backpack |
| +20 | held carryable: the shield while it is in hand, or an objective; 0 when nothing is held |
| +28 | slot in hand: 1 primary, 2 sidearm, 3 support (0 on the ship) |
| +32 | carry mode: 2 held, 1 put away while still the held item (during some actions), 0 nothing held |

| game.dll | What it does |
| --- | --- |
| `0x9ABC30(inventory, avatar, item)` | sets the held carryable (attachment and animation set); 0 clears it |
| `0x9AC8E0(inventory, avatar, mode)` | sets the carry mode; with nothing held it writes 0; mode 2 with a shield plays the shield's draw event |
| `0xBE20B0(peer, avatar, item, mode)` | sends `rpc_set_held_carryable`; peer -2 = every other player. Its receiver runs the two calls above |
| `0xA95910` | the game's unwield path: with a two-handed weapon it stows the shield with `0x9ABC30(0)`, mode 1 and the message |
| `0xA95A90` | after a wield: with a one-handed weapon, an owned Helldiver, a type-25 backpack and nothing else held it draws the shield (`0x9ABC30(shield)`, mode 2, the message) |

## What the addon does

On a press of Backpack Function (Avatar action pair 2, 37: the processed slot at `[game + 0x3326D20] + 0x150 +
index * 0xA7AEC + 0x1B68 + 37 * 32`, held time at +8) while a type-25 backpack is worn:

| State at the press | The addon |
| --- | --- |
| the shield is the held carryable | toggle on; stow: `0x9ABC30(inv, avatar, 0)`, `0x9AC8E0(inv, avatar, 1)`, `0xBE20B0(-2, avatar, shield, 1)` |
| nothing held, toggle off | toggle on; nothing moves (a two-handed weapon is out) |
| nothing held, toggle on | toggle off; with a one-handed weapon in hand, draw: `0x9ABC30(inv, avatar, shield)`, `0x9AC8E0(inv, avatar, 2)`, `0xBE20B0(-2, avatar, shield, 2)` |
| an objective held | only the toggle flips |

While the toggle is on and the row shows the shield held again (the game drew it), the addon stows it on that
frame. More than 6 re-stows within 120 frames end the toggle. A new avatar entity, another backpack or a guard
pause ends it too.

Before any call the addon finds everything again from the registry roots: the player manager `[game + 0x3326468]`
(+0x84 counts, +0xE8 the local player's record, owned bit; +0x3A8 the avatar's network id), the entity maps of
`[game + 0x346BF98]` (+0xF22EC8 by network id, +0xF1AEB0 by entity id, 24-byte records at +0xF32F18; the avatar's
resource `0x4D1C334D294DFA97` and owned bit), the avatar manager entry (map +0xF8, count +0x6C, entity pointers
+0x110, the same record), the inventory row (its identity pointer must be the avatar's record), and the
backpack's type through the game's equipment getter (`0x5092B0`: a per-entity copy at `[game + 0x3326DC0]` map
+80, records +144; else the shared settings at `[[game + 0x346BF98] + 0xF12BC0]`, 688 slots probed from the
resource modulo 688, 232-byte records after them). For a draw it reads the weapon in hand's one-handed flag (+0xC4)
the same way.

Every move is checked on the next frame against the row: a stow must leave nothing held, a draw must hold the shield
in mode 2. The log line says `done` or `FAILED, the game did not take it` with the row's values.

Once per session: the game.dll and EXE hashes (`bingus_memory.verify_build`), the first 16 bytes of the three
functions, then the function pointers from private typedefs (`hys1_set_item`, `hys1_set_mode`, `hys1_send_held`).
The game's "nothing held" ids (`[game + 0x3483C4C]`, `[game + 0x3483C5C]`; 0 in this build) are read at start.

## Per-frame cost

Calls per frame as `tests/test_mod.lua` pins them. In game a read (ReadProcessMemory) costs about 1-2 us; the
game's functions and the Lua around them are unmeasured in game. No path queries a page or writes memory.

| Path | Calls | In game |
| --- | --- | --- |
| No local player (menus, loading) | none; on every 30th frame 1 view and 2 pointer reads | about 3 us every 30 frames |
| Kept row, another backpack | 1 view | about 1-2 us |
| Shield worn, no press | 2 views | about 2-4 us |
| Every 30th frame with a kept row | + 3 views and 3 pointer reads | about 6-12 us |
| Press that stows, or a re-stow | 25 views, 12 pointer reads, the 3 calls | reads about 37-74 us; calls unmeasured in game |
| Press that draws | 34 views, 15 pointer reads, the 3 calls | reads about 49-98 us; calls unmeasured in game |
| The frame after a move | 2 views (the check uses the row read) | about 2-4 us |

Everything runs interpreted: the addon adds no trace to the LuaJIT code cache the game and every mod share. Idle
frames allocate nothing and create no C types (`tests/test_mod.lua`).

## Tests

`python -B scripts/build.py` runs, in a LuaJIT and in the game's own `lua51.dll`:
- `tests/test_mod.lua`: the addon on `tests/fake_game.lua`, a simulation of the registries above whose game
  functions behave as the decompiled ones (a stow leaves mode 0). Every press case, the directional shield,
  other backpacks, a held key, re-stows and their limit, a move the game does not take, refusals, per-entity
  equipment copies, moved inventory rows, a respawn, the exact calls of every path, no allocation and no C types
  per idle frame, the real adapter's reads and its refusal of a function whose first bytes differ, install.
- `tests/test_hostile.lua`: the Windows names declared first by a neighbour, an update below that raises (the
  pause ends the toggle; the error passes unchanged), a neighbour that calls the chain twice, the shutdown summary.
- `tests/compile_entry.lua` and `tests/test_entry.lua`: the assembled entry compiles, refuses outside the game
  without hooking anything and steps aside for an installed copy.

## Open questions

- The pose and first-person view with a one-handed weapon and the shield on the back.
- The Directional Shield's barrier stays off while on the back (its ShieldController was not traced).
- Vehicles, emplacements and exosuits; ragdoll and death in the middle of a move.
- Two players: what others see after a stow, a draw and a re-stow.
- How often the game draws the shield again in play, and the cost in game.
