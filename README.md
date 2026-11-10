# Fortune's Weave save editing

A reference for the save file format of **Fire Emblem: Fortune's Weave**, game update **1.1.1 (version 131072)**:
what every offset, field, block and hash means, and how to edit a save without breaking it.

The format is the same for Switch 2 save data, emulators and the fefw-recomp PC port; files are interchangeable.
This repository contains documentation only: no tools, no game data, no save files.

## Contents

| Document | Covers |
|---|---|
| [docs/01-container.md](docs/01-container.md) | slot files, the 0x000..0x124 container and header summary, `Global`, chunk headers, the stream/object encoding, FNV hashes, the payload root, restart points, an index of every tree node |
| [docs/02-units-convoy-mounts.md](docs/02-units-convoy-mounts.md) | `UnitPool` and the unit record, items, skills, mounts (`RidePool`), battalions (`LegionPool`), the convoy (`Transporter`), every copy of those pools |
| [docs/03-game-user-data-and-variables.md](docs/03-game-user-data-and-variables.md) | `GameUserData`: scalar body, mode/difficulty, variables (flags), supports, `GameSchedule`, playtime blocks, blessings, war record, the timeline (story position) key |
| [docs/04-routes-world-hub.md](docs/04-routes-world-hub.md) | `GameConverge` (route records, cross-route data, Reliquary), the world map, `HubData` and its mini-saves, paralogues, the save card |
| [docs/05-misc-blocks.md](docs/05-misc-blocks.md) | battle suspend data (`MapSeq`, incl. the Rewind history), `SubquestData`, `Calendar`, `RecommendData`, `Random`, `GameSound`, `GameSituation`, `CmdBtlSeq`, `GmapSeq` |
| [docs/editing-guide.md](docs/editing-guide.md) | the cookbook: splicing, rehashing, restart copies, and recipes for stats, classes, items, convoy, mounts, supports, blessings, boons, flags, paralogues, difficulty, story position, routes, map reveal, renaming, playtime |
| [docs/known-tool-pitfalls.md](docs/known-tool-pitfalls.md) | mistakes save tools have made or easily make, and what to do instead |
| [REFERENCES.md](REFERENCES.md) | credits |

## Status markers

- **confirmed**: verified in the game's code, or consistent across every real save examined with exact parsing.
- **inferred**: consistent with the code and data, but not proven.
- **unknown**: the layout is known, the meaning is not.

Code addresses are written `main+0x...`: offsets into the game's main executable for update 131072. They identify
the reader/writer functions the facts come from; they are not needed for editing.

## File anatomy at a glance

All integers are little-endian. A *document position* is a payload offset + 32 (pointer words count the 32-byte chunk
header).

```
SLOT FILE  000/<name>   Auto, Manual1, Chapter_<c>_<p>_<n>, Part_<c>_<p>, Route_<c>, Route2_<c>
0x000 +-- header document (always 0x100 bytes)
      |   0x000 chunk header: u32 magic CDEFBCDD | u32 size 0xE0 | u32 FNV1-32(file[0x20:0x100]) | u32 type 1 | 16 x 00
      |   0x020 summary object, version 15:
      |         0x26 GameMode (0 Casual, 1 Classic) | 0x27 Difficulty (0 Normal, 1 Hard, 2 Lunatic)
      |         0x28 lord | 0x29 part | 0x2A chapter | 0x2B u16 battle turn | 0x2D u32 status flags | 0x31 scene kind
      |         0x32 in-game date y,m,d,h,mi,s | 0x39 u64 battlefield key | 0x49 u64 timeline key
      |         0x54 playtime u16 h, u8 m, u8 s | 0x58 real-clock date | 0x5F u32 spot IdHash
      |         0x63 Part-2-clear mask | 0x65 paralogue mask | 0x67 save ID | 0x6B story-line id | 0x6F party size
      |         0x70 SaveCount | 0x74 MakeCount | 0x76 u64 Eshmel's class | 0x7E Eshmel's gender for the card
0x100 +-- u32 N = file length - 0x104
0x104 +-- data document chunk header: magic | size | FNV1-32 of the UNCOMPRESSED payload | type 3 (zstd) or 1 (raw)
0x124 +-- zstd frame -> PAYLOAD

PAYLOAD = anonymous root object; its member table {FNV1-32(name), position} is the last thing in the payload
  Header          summary object again (no version byte); payload offset = file offset - 0x18
  GameSound       v7   music/ambience to restore
  GameSchedule    v1   in-game clock, next deadline
  GameUserData    v23  100-byte body (mode, difficulty, gold, Karma, save ID, mount serial, ...), then:
                         Variable (13 maps) | GameReliance | GameSituation | GameConverge {RouteInfo, CommonInfo,
                         Depository, Temporary} | GameRecord | u32 flags | GameMiracle | TaskData | timeline stack |
                         CheckPerson | SequenceTime | SectionTime | Storage | u32 timer A, u32 timer B
  HubData         v21  hub state + up to 3 embedded mini-save documents
  GmapSpotMgr     v20  864 world-map spots
  GmapTroopMgr    v7   world-map troops (troop 0 = the party), with full unit records
  GameGmapData    v12  world-map situation, chatter played, event queue
  GameRestartData v3   8 restart slots, each {Header: summary document, Data: a complete payload document}
  SubquestData    v4   side quests
  SideStoryData   v4   paralogues + cleared-ever list
  Calendar        v6
  RecommendData   v0
  Random          v0   global xorshift128 state
  UnitPool        v0   units by force list
  RidePool        v3   200 mount slots
  LegionPool      v3   battalions
  Transporter     v0   convoy: 999 regular + 330 fixed stack slots
  [MapSeq | HubSeq | GmapSeq]   at most one sequence block, depending on where the save was made
  member table

GLOBAL  000/Global   [u32 N][document] (no 0x100 header): version 4, last slot index/type, system language,
                     voice language, insect phobia, title-notice mask
```

Hashes: member names are **FNV-1 32**; data keys are **FNV-1 64**; u32 ids ("IdHash") are the low 32 bits of FNV-1
64. All use multiply-then-XOR (FNV-1, not FNV-1a).

## The rules that matter most when editing

1. Growing anything shifts every later pointer in that document: use a real splice, then refresh every chunk hash,
   innermost first.
2. Restart points are complete separate payloads (without `GameConverge`); edit them too or accept that a restart
   rolls the edit back.
3. Current HP is one byte at unit +64.
4. Body+1 is GameMode and body+2 is Difficulty (many tools have these swapped).
5. Kind-0 variables the game hasn't registered are dropped on load; boons must use the game's short keys.
6. A ridden mount exists twice; keep `RideSerial` (GameUserData body+39) at or above the highest mount serial.
7. Paralogue clears persist across routes only through the SideStoryData cleared-ever list.
8. Story position is not lord/part/chapter; start from a real checkpoint.

## Status and coverage

- **Tree nodes:** of the 105 distinct top-level nodes found in real saves, **98 are decoded and 7 partial; none is
  unknown**. All restart-payload-only nodes (`CmdBtlSeq`) are decoded.
- **Member-name hashes:** all of them resolve to names.
- **Spot keys:** all 864 world-map node keys are derivable from the node table. **Timeline keys:** every key found in
  real saves resolves to a timeline row.
- **Variables:** all saved categories are decoded. About 195 kind-0 variable hashes seen in real saves remain
  unnamed (mostly the value-1 partners of 回想 recollection entries).
- **Partial nodes:** `Variable/回想` (partner names, packed snapshot values), `CommonInfo/GameUnitRecord` (counter
  meanings), `GameConverge/Temporary` (purpose), HubData object records (the u64 hub hash formula), world-map troops
  (a few fields), `MapSeq/AIValueFlags` and `AIValuePosLists` (always empty).
- **Layout known from code only** (never seen populated in a real save): player battalions, named mounts, route units
  written by the game, RouteInfo storage-set data, Reliquary contents, paralogue list 2, GameRecord map records,
  dispatch/recruit/reconstruction attachments, Part 3 miasma and blade-resonance state, calendar reservations, hub
  ObjectManager records.

### Open questions

- Whether the game checks chunk hashes on load: the code suggests it doesn't, but this hasn't been tested in game,
  so the docs recommend refreshing them.
- War Arc data paths: what fills RouteInfo storage set B, exactly when `Storage` is filled and emptied, what
  `Temporary` is for.
- The u64 "global hash" construction used in HubData and its mini-saves.
- Small leftovers listed as *unknown* in each document (a few header/flag bits, unit state bits 0x80/0x200/0x200000,
  battle counters, calendar purposes, GameSound semantics).

Corrections and additions are welcome.

## Credits and license

This work builds on two community projects; see [REFERENCES.md](REFERENCES.md). The documentation is licensed under
[CC BY 4.0](LICENSE).

Fire Emblem and Fortune's Weave are trademarks of their respective owners. This is an unofficial, fan-made
reference and is not affiliated with or endorsed by them.
