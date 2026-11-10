# 01. Container, encoding, slot files and restart points

This document covers everything outside the individual data blocks:

- the slot files and their names
- the outer container (every byte from file offset `0x000` to `0x124`) and the `Global` file
- the chunk ("document") header and compression
- the stream/object encoding that every payload uses
- the hash functions
- the payload root and its blocks
- restart points (`GameRestartData`)
- an index of every node in the object tree

All integers are little-endian. Code addresses are offsets into the game's main executable for update 1.1.1
(version 131072), written `main+0x...`. Status markers are explained in the [README](../README.md#status-markers).

---

## 1. Slot files

A save folder (`000/`) holds one file per slot. The file name comes from the slot type
(name builder `main+0x87c0e0`) **(confirmed)**:

| Type | File name | Notes |
|---|---|---|
| 0 | (none) | |
| 1 | `Global` | settings; no 0x100-byte header (§5) |
| 2 | `Auto` | the autosave / suspend slot; the file "Continue" loads |
| 3 | `Manual1` | |
| 4 | `Route_<code>` | a route parked part-way through Part 1, used by "Resume" |
| 5 | `Route2_<code>` | the same for Part 2 |
| 6 | `Part_<code>_<part>` | the War Arc start checkpoint (`Part_<code>_2`), written at the end of Part 1 |
| 7 | `Chapter_<code>_<part>_<chapter>` | chapter-start checkpoints |
| 8 | `Debug` | |
| 9 | `KeyLog` | |

Lord codes (by lord number): 0 `Nan`, 1 `Kai`, 2 `Dit`, 3 `Seo`, 4 `Led`, 5 `Bel`, 6 `Tal`, 7 `Orh`, 8 `Anh`, 9 `Ism`.
The lord numbers are the route numbers used throughout the save: 1 Cai, 2 Dietrich, 3 Theodora, 4 Leda, 5 Beltran,
6 Talimun, 7 Orchel, 8 Anatolia, 9 Eshmel.

Facts about slot files **(confirmed)**:

- Every save writes `Global` first (`SaveSlot`, `main+0x87ff50`), then the requested slot.
- `Chapter_*` and `Part_*` files never contain a sequence block (§7) and all of their restart slots are empty.
- `Chapter_<lord>_1_1` has an empty `UnitPool`.
- `Auto` and `Route_*` files carry whichever sequence block was active when they were written.
- Save files written by the Switch 2 game, by emulators and by the fefw-recomp PC port use the same format and are
  interchangeable.

---

## 2. Slot file layout

Writer `main+0x87faa0`, reader `main+0x87f4c0` **(confirmed)**.

```
0x000  header document  (always 0x100 bytes: 32-byte chunk header + 224-byte summary object + padding)
0x100  u32 N            length of the data document = file length - 0x104
0x104  data document    32-byte chunk header + payload (zstd-compressed, or raw)
```

- The writer always writes 0x100 header bytes, then `N`, then N bytes.
- The reader reads the 0x100-byte header only if the file is at least 0x100 bytes long, then `N` and N bytes. It
  decompresses and checks both magics.
- Nothing is encrypted.

---

## 3. Chunk ("document") header

Every document (the file header, the data document, `Global`, and every embedded document) starts with this
32-byte header, written when a depth-0 object closes (`main+0x65d6c0`) **(confirmed)**:

| Offset | Type | Value |
|---|---|---|
| +0x00 | u32 | magic `0xCDEFBCDD` |
| +0x04 | u32 | data size: the bytes after this header (the compressed size if compressed) |
| +0x08 | u32 | FNV-1 32 of the **uncompressed** data; a result of 0 is stored as `0xFFFFFFFF` |
| +0x0C | u32 | type: 1 = raw, 3 = zstd (raw \| 2) |
| +0x10 | 16 bytes | zero |

**Compression (confirmed).** The writer compresses type-1 data with zstd and keeps the result **only if it is
smaller**; otherwise the data stays raw with type 1. So a data document of type 1 is legal at file level and readers
must accept it. The header document (224 bytes of summary) is never compressed.

**Hash checks on load (inferred: none).** The only FNV-1 32 loop over a stream buffer in the executable belongs to
the writer (`main+0x65e984`); the header reader and the file loader check only the magic. This has not been tested
in game, so editors should still refresh every hash after an edit, innermost documents first (§9).

### 3.1 Data-document bytes 0x104..0x124

| File offset | Size | Field |
|---|---|---|
| 0x104 | 4 | magic `CDEFBCDD` |
| 0x108 | 4 | compressed size (or raw size if type 1) |
| 0x10C | 4 | FNV-1 32 of the uncompressed payload |
| 0x110 | 4 | type 3 (zstd) or 1 (raw) |
| 0x114 | 16 | zero |
| 0x124 | ... | zstd frame (or raw payload) |

---

## 4. The header document and the summary object

The header struct is filled by `main+0x87cc00` and serialised by `main+0x87cfb0` as a **version 15** summary object.
The reader (`main+0x87d630`) gates fields by version. `G` below is the GameUserData runtime object; `body+N` refers to
the GameUserData block body ([doc 03](03-game-user-data-and-variables.md#1-gameuserdata-body)).

### 4.1 Every byte of the file header (version 15)

| File off | Size | Field | Meaning / source | Status |
|---|---|---|---|---|
| 0x000 | 4 | magic | `CDEFBCDD` | confirmed |
| 0x004 | 4 | size | 0xE0 (224) = file 0x20..0x100 | confirmed |
| 0x008 | 4 | hash | FNV-1 32 of `file[0x20:0x100]` (0 stored as `FFFFFFFF`) | confirmed |
| 0x00C | 4 | type | 1 (raw) | confirmed |
| 0x010 | 16 | zero | | confirmed |
| 0x020 | 4 | object pointer word | `0x00000080`: no named members, object ends at document position 0x80 | confirmed |
| 0x024 | 1 | version | 15 | confirmed |
| 0x025 | 1 | format sub-revision | constant 2 (0 in an older game build); written together with the version, no reader found | confirmed value, meaning inferred |
| 0x026 | 1 | **GameMode** | `G+0x0C` = body+1: **0 Casual, 1 Classic** | confirmed |
| 0x027 | 1 | **Difficulty** | `G+0x0D` = body+2: **0 Normal, 1 Hard, 2 Lunatic (Maddening)** | confirmed |
| 0x028 | 1 | lord | `G+0x0F` (1..9) | confirmed |
| 0x029 | 1 | part | `G+0x3C` (0 prologue, 1, 2, 3) | confirmed |
| 0x02A | 1 | chapter | `G+0x40` | confirmed |
| 0x02B | 2 | battle turn | equals `MapSeq/Situation` turn in battle saves, 0 elsewhere | confirmed |
| 0x02D | 4 | status flags | §4.2 | confirmed |
| 0x031 | 1 | scene kind | `G+0x38`: 2 battle, 4 world map, 5 hub/base, 6 dungeon, 7 command battle, 8 Part checkpoint, 11 chapter start | confirmed |
| 0x032 | 2 | in-game year | copied from the GameSchedule clock | confirmed |
| 0x034 | 1 | in-game month | | confirmed |
| 0x035 | 1 | in-game day | | confirmed |
| 0x036 | 1 | in-game hour | | confirmed |
| 0x037 | 1 | in-game minute | | confirmed |
| 0x038 | 1 | in-game seconds | sixth field of the clock date | confirmed |
| 0x039 | 8 | battlefield key | FNV-1 64 of the battlefield row key, e.g. `FNV1-64("SC13_00") = 0xEF80E0F9E5467712`; 0 outside battle (= body+50) | confirmed |
| 0x041 | 3 | zero | alignment | confirmed |
| 0x044 | 4 | pointer word | anonymous object, ends at document position 0x54 | confirmed |
| 0x048 | 1 | 0 | | confirmed |
| 0x049 | 8 | timeline key | the timeline cursor row ([doc 03 §10](03-game-user-data-and-variables.md#10-the-timeline-key-story-position)); equals the anonymous object in the GameUserData body | confirmed |
| 0x051 | 3 | zero | | confirmed |
| 0x054 | 2 | playtime hours | | confirmed |
| 0x056 | 1 | playtime minutes | | confirmed |
| 0x057 | 1 | playtime seconds | | confirmed |
| 0x058 | 2 | real-clock year | wall-clock date of the save | confirmed |
| 0x05A | 1 | month | | confirmed |
| 0x05B | 1 | day | | confirmed |
| 0x05C | 1 | hour | | confirmed |
| 0x05D | 1 | minute | | confirmed |
| 0x05E | 1 | second | | confirmed |
| 0x05F | 4 | location | IdHash of the current world-map spot (0 = none) | confirmed |
| 0x063 | 2 | Part-2-clear mask | bit L set if variable `2部クリア_<L>篇` ≠ 0 (L = lord 1..8) | confirmed |
| 0x065 | 2 | paralogue mask | bit L set if lord L's paralogue clear flag (`X外伝_クリア`) ≠ 0 | confirmed |
| 0x067 | 4 | save ID | `G+0x08` = body+27; must equal the loaded GameUserData save ID | confirmed |
| 0x06B | 4 | story-line id | `G+0xA90` = body+31 | confirmed |
| 0x06F | 1 | party size | number of units in the first UnitPool list | confirmed |
| 0x070 | 4 | SaveCount | `G+0xA9C` = body+43 | confirmed |
| 0x074 | 2 | MakeCount | `G+0xAA0` = body+47 | confirmed |
| 0x076 | 8 | Eshmel's class | FNV-1 64 job key of Eshmel's unit (e.g. `JID_ガーディアン`), 0 if he has none | confirmed |
| 0x07E | 1 | Eshmel's gender for the card | Person `性別` of his person row (1 male, 2 female); if that is 3 (player-chosen), `max(1, UnitEdit look byte 0)`; 0 if there is no such unit | confirmed |
| 0x07F | 1 | zero | aligns the object end to 0x80 | confirmed |
| 0x080 | 0x80 | zero | padding up to 0x100 | confirmed |

The game rebuilds **all** summary fields on every save (the two masks from variables). Editing the summary changes
only what the Continue card shows until the next save. The masks are never read by the card.

### 4.2 Status flags (file 0x02D)

| Bit | Meaning | Status |
|---|---|---|
| 0 | battle start pending: set just before the battle-start autosave and the MAP restart point, cleared when turn 1 begins | confirmed |
| 1 | battle entered from the world map | confirmed |
| 2 | map completed | confirmed |
| 3..10 | other `G+0x5C` flags (setters known, not named) | unknown |
| 25 | a CHAPTER restart point exists | confirmed |
| 27 | variable `拠点_所在_方尖塔の間` (currently in the Obelisk Chamber) | confirmed |
| 28 | variable `拠点_経由_方尖塔の間` (has passed through the Obelisk Chamber) | confirmed |
| 30 | the slot is rejected (`CheckSlot` returns 3; Continue refuses it) | confirmed |

Bits 0..10 are the GameUserData `G+0x5C` flags word, which is also stored as the u32 after `GameRecord` in the
GameUserData block. The game may later OR status bits into *other* slot files' headers in place (only the 0x100-byte
header is rewritten).

### 4.3 The payload `Header` block

The first payload block, `Header` (FNV-1 32 `0x624F70B2`), holds the **same summary object**, byte-identical except
for the inner anonymous object's pointer word, which is relocated **(confirmed)**. It has no version byte. It is
followed by zero padding up to document position 0x100, so it is always 220 bytes.

Because the summary sits at payload+8 (after the root pointer word and the `Header` pointer word), a summary field at
file offset `F` is at payload offset `F − 0x18`. For example GameMode is at payload[14] and Difficulty at payload[15].

### 4.4 Slot status

`CheckSlot` (`main+0x87ed50`) reads only the header **(confirmed)** and returns:

- 0: no file
- 3: status flag bit 30 set
- 2: the header save ID differs from the current one ("other save data")
- Route slots: 1 if the lord matches the current lord and the part is non-zero, otherwise 4

The Continue card reads only the `Auto` header (lord/part/chapter, location, dates, playtime) and refuses it if bit 30
is set or the save ID doesn't match.

---

## 5. The `Global` file

Writer `main+0x8b22f0`, reader `main+0x8b2490` **(confirmed)**. `Global` has no 0x100-byte header region:
`[u32 N][document]`.

```
0x00  u32 N = 0x3C (60)
0x04  chunk header: magic, size 0x1C, FNV-1 32, type 1, 16 x 0
0x24  root object pointer word 0x01000034 (1 member, member table at document position 0x34)
0x28  "Global" member object pointer word (ends at document position 0x34)
0x2C  u8 version = 4
0x2D  u8 A
0x2E  u8 B
0x2F  u8 C
0x30  u8 D
0x31  u8 E
0x32  u8 constant 2
0x33  u32 F
0x37  u8 pad
0x38  member table: {0x2951B00C = FNV1-32("Global"), 0x00000024}
```

| Byte | Meaning | Status |
|---|---|---|
| A | index of the last used Manual slot; −1 (`0xFF`) when the last slot wasn't a Manual slot | confirmed |
| B | slot type of the last loaded/written slot (2 Auto, 3 Manual, 4 Route, 6 Part, 7 Chapter, ...). Resuming from a Chapter slot (B = 7) starts a new story line | confirmed |
| C | system language index: 0 Japanese, 1 US English, 2 US French, 3 US Spanish, 4 EU English, 5 EU French, 6 EU Spanish, 7 EU German, 8 EU Italian, 9 Traditional Chinese, 10 Simplified Chinese, 11 Korean. Written, but skipped by the reader and re-derived from the system each boot | confirmed |
| D | voice language: 0 Japanese, 1 English | confirmed |
| E | insect-phobia setting | confirmed |
| F | bitmask of title-screen notices already shown (`1 << notice kind`); 24 = kinds 3 and 4 seen. Which notices those kinds are is not identified | mechanism confirmed |

An older game build wrote version 3: a 56-byte document (60-byte file) that stops after the constant byte.

---

## 6. Stream and object encoding

All saves are written through one stream class **(confirmed)**. Begin object `main+0x65dfe0`, end object
`main+0x65e520`, begin read `main+0x65ec20`, end read `main+0x65f880`.

### 6.1 Positions

Positions are absolute offsets into the document buffer, and that buffer **starts with the 32-byte chunk header**.
So every pointer stored in a payload is "payload offset + 32" (called the *document position* in these docs).
Embedded documents (restart payloads, HubData sub-documents) have their own coordinate system that starts at their
own chunk header.

### 6.2 Objects

Every object starts with a u32 **pointer word**:

- low 24 bits: an absolute document position
- high byte: the number of **named** members

| Member count | The pointer gives | After the content |
|---|---|---|
| 0 | the end of the object | nothing |
| > 0 | the position of the member table | a member table `{u32 FNV1-32(name), u32 position} × count` |

- Objects are 4-aligned. The writer zero-pads to 4 before an object and at its end. **Alignment padding is not
  guaranteed to be zero on disk** (a real save has `0x0d` in `UnitPool` padding), so never treat padding as data.
- A **named** child (a member) appears in its parent's member table *and* inline in the parent's content.
- An **anonymous** child (hash 0) appears inline only. Nothing in the format marks it, so a reader without the schema
  can only guess where it is (see [known pitfalls](known-tool-pitfalls.md#7-heuristic-tree-walkers)).
- The member table order is the writer's hash-map order and has no meaning. Readers look members up by hash.
- A parent may write raw bytes **between** member objects (GameUserData does it three times).
- An end-object call may pad up to a minimum absolute size (the payload `Header` is padded to document position 0x100).

### 6.3 Reading rules (forward compatibility)

- A reader looks each member up by hash. A missing member returns "not found" and the game keeps its defaults.
- After reading an object, the reader jumps to the object's recorded end. Unknown members and unread trailing bytes
  are skipped.
- Because everything is length-delimited by absolute pointers, **growing any object shifts every later pointer** in
  the same document. See [the editing guide](editing-guide.md#2-splicing-changing-the-size-of-anything).

### 6.4 Primitives

| Primitive | Encoding |
|---|---|
| integers | u8/u16/u32/u64 little-endian, unaligned unless inside a new object |
| string | `u16 (len + 1)`, the bytes, a NUL. An empty string is `u16 0`. Not aligned |
| data reference | u64 key of a game-data row = FNV-1 64 of the row key; 0 = null |
| anonymous data-reference object | an object `{u8 0, u64 key}` |
| custom name (protagonist, mounts) | a string whose first byte is the marker `0x07`: `u16 n`, then `07`, UTF-8 name, `00` (n counts the 07 and the NUL) |

### 6.5 Version bytes

Every payload root block except `Header` starts with a u8 version written by the save framework. The version byte is
not a field of the block. Nested members carry a version only if their own serializer writes one; `CommonInfo`,
`Depository` and `Temporary`, for example, start directly with an anonymous object. Versions in update 131072:

| Block | Ver | Block | Ver | Block | Ver |
|---|---|---|---|---|---|
| GameSound | 7 | GameRestartData | 3 | UnitPool | 0 |
| GameSchedule | 1 | SubquestData | 4 | RidePool | 3 |
| GameUserData | 23 | SideStoryData | 4 | LegionPool | 3 |
| HubData | 21 | Calendar | 6 | Transporter | 0 |
| GmapSpotMgr | 20 | RecommendData | 0 | MapSeq | 0 |
| GmapTroopMgr | 7 | Random | 0 | HubSeq | 0 |
| GameGmapData | 12 | | | GmapSeq | 2 |
| | | | | CmdBtlSeq | 0 |

Record versions: unit 36, mount (UnitRide) 6, unit counters 5, UnitEdit 3, RouteInfo record 16, CommonInfo 12,
GameRecord 7, GameMiracle 2, GameReliance 2, Variable 2, troop 24, spot attachment 9.

---

## 7. Hashes

| Use | Function | Notes |
|---|---|---|
| member names | **FNV-1 32** of the UTF-8 name: basis `0x811C9DC5`, prime `0x01000193`, *multiply then XOR* | `main+0x7ac540`. It is FNV-1, not FNV-1a (a comment in the decompilation says FNV-1a; that is wrong) |
| data keys | **FNV-1 64** of the UTF-8 row key: basis `0xCBF29CE484222325`, prime `0x100000001B3`, multiply then XOR | `main+0x7ac700`. Used for `PID_`, `JID_`, `IID_`, `SID_`, `RID_`, `LID_`, `SSID_`, `MIR_` keys, scene names, `SCxx_yy` battlefields, timeline keys |
| IdHash | the **low 32 bits** of FNV-1 64 | used for u32 ids: unit PID/JID, items, skills, kind-0 variables, spot keys, GameRecord refs, counter keys |
| chunk hash | FNV-1 32 over the document's data (uncompressed) | 0 is stored as `0xFFFFFFFF` |

FNV-1, for reference:

```
h = basis
for each byte b of the UTF-8 string:
    h = (h * prime) mod 2^bits
    h = h XOR b
```

Example values (all **confirmed**):

| String | Hash |
|---|---|
| FNV1-32 `Global` | `0x2951B00C` |
| FNV1-32 `Header` | `0x624F70B2` |
| FNV1-32 `GameUserData` | `0x887CA386` |
| FNV1-32 `RideData` | `0x09B9F71F` |
| FNV1-64 `SC13_00` | `0xEF80E0F9E5467712` |
| FNV1-64 `SSID_カイ外伝` | `0xA26AEDB9FDCA69D0` |
| IdHash `帝都ダグシオン` (spot key) | `0x621A6FD3` |

**Deriving names.** No name table is stored in the save. To label a hash, hash candidate strings and compare: member
names come from the serializer class names, data keys from the game's data tables (`PID_*`, `IID_*`, ...), and
variable names from known families ([doc 03 §3.7](03-game-user-data-and-variables.md#37-deriving-variable-names)).

---

## 8. The payload root

The payload is one anonymous root object **(confirmed)**:

- it starts at payload+0 (its pointer word's high byte is the member count, 18..20);
- its member table is the very last thing in the payload, so the payload length = the table end;
- its members are the 18 core blocks, always written in this order, followed by at most one sequence block.

| # | Block | FNV-1 32 | Version | Contents | Doc |
|---|---|---|---|---|---|
| 1 | `Header` | 624F70B2 | – | summary object (§4.3) | 01 |
| 2 | `GameSound` | A83FDECC | 7 | music/ambience state to restore | [05](05-misc-blocks.md#6-gamesound-v7) |
| 3 | `GameSchedule` | 78B1DA94 | 1 | in-game clock and the next deadline | [03](03-game-user-data-and-variables.md#5-gameschedule-top-level-block) |
| 4 | `GameUserData` | 887CA386 | 23 | scalars, variables, supports, routes, records, ... | [03](03-game-user-data-and-variables.md) |
| 5 | `HubData` | FC7F1B42 | 21 | hub state + up to 3 embedded mini-saves | [04](04-routes-world-hub.md#3-hubdata-and-hubseq) |
| 6 | `GmapSpotMgr` | 73F72B18 | 20 | the 864 world-map spots | [04](04-routes-world-hub.md#21-gmapspotmgr-v20) |
| 7 | `GmapTroopMgr` | 784CE518 | 7 | world-map troops (troop 0 = the party) | [04](04-routes-world-hub.md#22-gmaptroopmgr-v7) |
| 8 | `GameGmapData` | FBA11788 | 12 | world-map situation, chatter played, event queue | [04](04-routes-world-hub.md#23-gamegmapdata-v12) |
| 9 | `GameRestartData` | D723A28C | 3 | 8 restart-point slots (§9) | 01 |
| 10 | `SubquestData` | 77B3CC39 | 4 | side quests | [05](05-misc-blocks.md#2-subquestdata-v4) |
| 11 | `SideStoryData` | EBD9EFBB | 4 | paralogues | [04](04-routes-world-hub.md#4-sidestorydata-paralogues) |
| 12 | `Calendar` | 5283171F | 6 | calendar reservations | [05](05-misc-blocks.md#3-calendar-v6) |
| 13 | `RecommendData` | D1521821 | 0 | hub "recommended activities" | [05](05-misc-blocks.md#4-recommenddata-v0) |
| 14 | `Random` | 5B6AB250 | 0 | global xorshift128 state | [05](05-misc-blocks.md#5-random-v0) |
| 15 | `UnitPool` | 69C819E9 | 0 | units by force list | [02](02-units-convoy-mounts.md#1-unitpool) |
| 16 | `RidePool` | 92C570B9 | 3 | 200 mount slots | [02](02-units-convoy-mounts.md#4-ridepool-and-ridedata) |
| 17 | `LegionPool` | EB3036B5 | 3 | battalions | [02](02-units-convoy-mounts.md#5-legionpool-and-legiondata) |
| 18 | `Transporter` | F07FF6D9 | 0 | convoy | [02](02-units-convoy-mounts.md#6-transporter-and-transporterdata) |
| 19 | one sequence block (optional) | | | see below | |

**Sequence blocks.** The game registers one extra serializer only while a sequence is running and appends it as the
last root block **(confirmed)**:

| Block | FNV-1 32 | When present | Contents |
|---|---|---|---|
| `MapSeq` | 33BD31C2 | saved in battle | battle suspend data, up to ~360 KB ([05](05-misc-blocks.md#1-mapseq-battle-suspend-data)) |
| `HubSeq` | 78AE2B2F | saved in the hub | an empty 8-byte marker (`[ptr][00][pad 3]`) |
| `GmapSeq` | 81F21E47 | saved on the world map | world-map cursor and camera ([05](05-misc-blocks.md#9-gmapseq-v2)) |
| `CmdBtlSeq` | 7503ED94 | only inside CMDBTL restart payloads, after `GmapSeq` or `HubSeq` | command-battle resume data ([05](05-misc-blocks.md#8-cmdbtlseq-v0)) |

Chapter and Part checkpoints are written at sequence transitions, so they have none. `Data` is **not** a root block:
it is a member of each restart slot (§9).

---

## 9. Restart points: `GameRestartData`

Writer `main+0x8b5480`, reader `main+0x8b5780` **(confirmed)**. The block is a struct with 8 slot members. The member
key is FNV-1 32 of the slot name:

| # | Slot | FNV-1 32 | Captured at | Scene of its payload |
|---|---|---|---|---|
| 1 | CHAPTER | 8C8807E8 | chapter start; used by "Restart chapter" (confirmed) | 11 |
| 2 | CHECK1 | A23565E0 | never seen filled | |
| 3 | CHECK2 | A23565E3 | never seen filled | |
| 4 | BACK | 9FD3E608 | a hub state (inferred) | 5 |
| 5 | CHECK3 | A23565E2 | never seen filled | |
| 6 | MAP | 1E2FC0A5 | battle start; used by "Restart battle" (confirmed) | 2 |
| 7 | DUNGEON | 0DB29669 | dungeon entry (inferred) | 6 |
| 8 | CMDBTL | 1475B227 | command-battle start (inferred) | 7 |

(Slot 0, `NONE`, FNV-1 32 `B15A4F9B`, is never written.)

```
GameRestartData   [ptr][u8 3][pad]  8 x slot object  member table {FNV1-32(slot name), position} x 8
  slot object     struct with 2 members: Header, Data
    Header        [ptr][u32 n][n bytes]   n = 0, or 256 = a complete summary document
                                           (32-byte chunk header + 224-byte summary, same layout as file 0x000..0x0FF)
    Data          [ptr][u32 n][n bytes]   n = 0, or a raw (type 1) payload document: chunk header + full payload
```

Member hashes: `Header` = `624F70B2`, `Data` = `D755BB1D`.

**Empty slots (confirmed).** `Header n = 0, Data n = 0` = never allocated (all Chapter files). `Header n = 256` of
zeros with `Data n = 0` = allocated but cleared. **Treat a slot as filled only when Data n > 0.** An empty
`GameRestartData` (8 empty slots) is 360 bytes.

**Restart payloads** are complete payloads of their own, with the same 18 core blocks. They differ from a top-level
payload in these ways **(confirmed)**:

- Their `GameRestartData` is 8 bytes (version 3, no members): restart points never nest.
- Their `GameUserData` has **no `GameConverge`** member (10 members instead of 11). The GameUserData writer skips it
  while restart data is being written, so restarting never rolls back cross-route data.
- Their `GameUserData/Storage` is always the empty form.
- They carry the sequence block of the moment they were captured: MAP has `MapSeq`; CMDBTL has `GmapSeq` or `HubSeq`
  plus `CmdBtlSeq`; BACK has `HubSeq`; DUNGEON has `HubSeq` (and a large HubData); CHAPTER has none.

Everything else in a restart payload (units, mounts, convoy, variables, supports, blessings, GameRecord) is a
**separate full copy**. An edit that should survive a restart must be repeated in each filled slot.

**Hashes.** Each slot's Header document and Data document has its own chunk hash, and HubData sub-documents inside a
restart payload have theirs. Refresh innermost first: HubData sub-documents, then restart Header/Data documents, then
the payload hash at file 0x10C, then the file header hash at 0x008.

---

## 10. Tree index

Every node seen in real saves, with its member hash. `[]` = an anonymous child object (no member-table entry).
`<lord>篇` = one member per route, keyed by FNV-1 32 of the route name (`カイ篇` ... `イシュマール篇`). Status:
**decoded** = layout and the meaning an editor needs are known; **partial** = layout known, some meanings open.

| Path | FNV-1 32 | Meaning | Status | Doc |
|---|---|---|---|---|
| `<root>` | – | 18 core blocks + 0..2 sequence blocks; member table at payload end | decoded | 01 |
| `/Header` | 624F70B2 | summary object, no version byte | decoded | 01 |
| `/GameSound` | A83FDECC | BGM/ambience state | decoded (meanings inferred) | 05 |
| `/GameSchedule` | 78B1DA94 | in-game clock, deadline, sub-second accumulator | decoded | 03 |
| `/GameUserData` | 887CA386 | 100-byte body + 11 members + 3 interleaved fields | decoded | 03 |
| `/GameUserData/[]` | – | `{u8 0, u64 key}` timeline cursor | decoded | 03 |
| `/GameUserData/Variable` | C68947E1 | the 13 saved variable categories | decoded | 03 |
| `.../Variable/ローカル` | 8186B481 | battle-map script variables (battle saves only) | decoded | 03 |
| `.../Variable/イベント` | E5834A3C | hub/dungeon script event variables | decoded | 03 |
| `.../Variable/限定` | 972FA7EE | route-local flags, shop stock, gift dates | decoded | 03 |
| `.../Variable/日時` | 0802E120 | date-triggered variables | decoded | 03 |
| `.../Variable/同期` | 64ABA34F | synchronised variables | decoded | 03 |
| `.../Variable/中継` | F56D6832 | supports watched, hub fast-travel points | decoded | 03 |
| `.../Variable/既読` | B631F2C1 | message labels already read | decoded | 03 |
| `.../Variable/状況` | 4AD575A2 | story progress, class/skill permissions | decoded | 03 |
| `.../Variable/共有` | 2BED8663 | cross-route: clears, boons, Obelisk, paralogues | decoded | 03 |
| `.../Variable/回想` | 06C136C7 | recollections | partial | 03 |
| `.../Variable/環境設定` | 58A0BE00 | options (non-default only) | decoded | 03 |
| `.../Variable/実績` | 25207C4C | deed/record counters and claim states | decoded | 03 |
| `.../Variable/チュートリアル` | E9BC5B65 | tutorials seen | decoded | 03 |
| `/GameUserData/GameReliance` | FE992C20 | all 2628 support pairs | decoded | 03 |
| `/GameUserData/GameSituation` | 3F7E8625 | version byte only | decoded | 05 |
| `/GameUserData/GameConverge` | A5D56C94 | cross-route store; absent from restart payloads | decoded | 04 |
| `.../GameConverge/RouteInfo` | 72218468 | 9 route records | decoded | 04 |
| `.../RouteInfo/<lord>篇` | FNV(`<lord>篇`) | route record v16 | decoded | 04 |
| `.../<lord>篇/[]` | – | recorded unit records + the two storage sets | decoded | 02, 04 |
| `.../<lord>篇/[]/TransporterData` | EFBE8231 | set convoy | decoded | 02 |
| `.../<lord>篇/[]/RideData` | 09B9F71F | set mounts | decoded | 02 |
| `.../<lord>篇/[]/LegionData` | 7644ADD3 | set battalions | decoded | 02 |
| `.../<lord>篇/GameRecord` | 36BB2FD4 | per-route chapter records | decoded | 03, 04 |
| `.../GameConverge/CommonInfo` | 79B0BBD2 | wraps one struct, no version byte | decoded | 04 |
| `.../CommonInfo/[]` | – | v12: spot snapshot, relay snapshot, protagonist profile | decoded | 04 |
| `.../CommonInfo/[]/GameUnitRecord` | B425772A | per-person counters + route mask | partial | 04 |
| `.../CommonInfo/[]/GameMiracle` | 426D8EFC | cross-route maximum blessing EXP | decoded | 04 |
| `.../CommonInfo/[]/GameOverCount` | 748C5182 | game overs per battle map | decoded | 04 |
| `.../GameConverge/Depository` | FBD85F53 | Drifting Reliquary storage set | decoded | 04 |
| `.../GameConverge/Temporary` | 3801F62A | same shape as Depository; purpose unknown | partial | 04 |
| `.../Depository/[]`, `.../Temporary/[]` | – | `{u8 0; TransporterData; RideData; LegionData}` | decoded | 02, 04 |
| `/GameUserData/GameRecord` | 36BB2FD4 | one record per cleared chapter | decoded | 03 |
| `/GameUserData/GameMiracle` | 426D8EFC | 7 god blessings | decoded | 03 |
| `/GameUserData/TaskData` | 6F7832F2 | current hub objective | decoded | 03 |
| `/GameUserData/CheckPerson` | 4126266C | PID → Person serial snapshot; ignored on load | decoded | 03 |
| `/GameUserData/SequenceTime` | E798A625 | playtime per scene kind | decoded | 03 |
| `/GameUserData/SectionTime` | B9F5F339 | playtime per story section | decoded | 03 |
| `/GameUserData/Storage` | B0B6A91C | parked convoy/mounts/battalions | decoded | 03 |
| `.../Storage/TransporterData`, `LegionData`, `RideData` | EFBE8231, 7644ADD3, 09B9F71F | | decoded | 02 |
| `/HubData` | FC7F1B42 | hub state | decoded | 04 |
| `/HubData/[]` | – | position, escape point, state lists, item sets, sub-documents | decoded | 04 |
| `/HubData/[]/[]` | – | object-state records, item-set entries | partial (hash formula open) | 04 |
| `/GmapSpotMgr` | 73F72B18 | 864 spots + tail lists | decoded | 04 |
| `/GmapSpotMgr/[]` | – | spot object | decoded | 04 |
| `/GmapSpotMgr/[]/[]` | – | animal / farm / attachment sub-objects | decoded | 04 |
| `/GmapTroopMgr` | 784CE518 | troops | decoded | 04 |
| `/GmapTroopMgr/[]` | – | troop v24 | partial | 04 |
| `/GmapTroopMgr/[]/[]` | – | full unit records | decoded | 02 |
| `/GameGmapData` | FBA11788 | world-map situation etc. | decoded | 04 |
| `/GameRestartData` | D723A28C | 8 restart slots | decoded | 01 |
| `/GameRestartData/<slot>` | (slot name) | `{Header, Data}` | decoded | 01 |
| `/GameRestartData/<slot>/Header` | 624F70B2 | `u32 n` + summary document | decoded | 01 |
| `/GameRestartData/<slot>/Data` | D755BB1D | `u32 n` + raw payload document | decoded | 01 |
| `/SubquestData` | 77B3CC39 | side quests | decoded | 05 |
| `/SideStoryData` | EBD9EFBB | paralogues | decoded | 04 |
| `/Calendar` | 5283171F | reservations, SSID lists | decoded (purpose partly open) | 05 |
| `/RecommendData` | D1521821 | `RCID_*` list | decoded | 05 |
| `/Random` + `/Random/[]` | 5B6AB250 | `{u8 0, u32 s0..s3}` | decoded | 05 |
| `/UnitPool` | 69C819E9 | units by force | decoded | 02 |
| `/UnitPool/[]` | – | unit record v36 | decoded | 02 |
| `/RidePool` + `/RidePool/RideData` | 92C570B9, 09B9F71F | 200 mount slots | decoded | 02 |
| `/LegionPool` + `/LegionPool/LegionData` | EB3036B5, 7644ADD3 | battalion list | decoded | 02 |
| `/Transporter` + `/Transporter/TransporterData` | F07FF6D9, EFBE8231 | 999 + 330 convoy slots | decoded | 02 |
| `/GmapSeq` | 81F21E47 | world-map sequence marker | decoded | 05 |
| `/GmapSeq/GmapCursor` | 3D0EA394 | `u8 2, f32 x, y, z` | decoded | 05 |
| `/GmapSeq/GmapCamera` | 7218293F | `u8 0, f32 x, y, z, u32 0` | decoded | 05 |
| `/MapSeq` | 33BD31C2 | battle suspend data | decoded | 05 |
| `/MapSeq/Situation` | F59699CB | battle-map state | decoded | 05 |
| `/MapSeq/Camera` | B804D7B0 | battle camera | decoded | 05 |
| `/MapSeq/Cursor` | F78654AF | cursor cell | decoded | 05 |
| `/MapSeq/Image` | 20B1D94C | changed terrain cells | decoded | 05 |
| `/MapSeq/Trick` | 4C2C1662 | map gimmicks | decoded | 05 |
| `/MapSeq/AIValueFlags` | 6BA115EB | always empty | partial | 05 |
| `/MapSeq/AIValuePosLists` | CE0C58B7 | always empty | partial | 05 |
| `/MapSeq/MapObjectManager` | 4ECE5371 | map-object states | decoded | 05 |
| `/MapSeq/MapEffect` | 13FDD09E | persistent map effects | decoded | 05 |
| `/MapSeq/MapHistory` | EDCAA2DB | battle Rewind history | decoded | 05 |
| `.../MapHistory/MapHistoryMapObjMgr` | C93CB89C | recorded map objects | decoded | 05 |
| `.../MapHistory/MapHistoryRelianceMgr` | 0A30C96E | battle-start support snapshot | decoded | 05 |
| `.../MapHistory/MapHistoryUnitMgr` | EBC9DD2D | unit-exists map | decoded | 05 |
| `.../MapHistory/MapHistoryVariableMgr` | 5CEE1B03 | variable baseline at battle start | decoded | 05 |
| `.../MapHistory/MapHistoryNewTypeMgr` | A195ED3F | always empty | decoded | 05 |
| `/HubSeq` | 78AE2B2F | empty 8-byte marker | decoded | 01 |
| `R/CmdBtlSeq` (restart payloads only) | 7503ED94 | command-battle resume object | decoded | 05 |

The names `MapObjectManager` (`4ECE5371`) and `MapHistoryMapObjMgr` (`C93CB89C`) have no string in the executable;
their hashes are compile-time constants and the names were recovered by matching hashes **(inferred, strong)**. All
other member hashes match strings in the executable.

### 10.1 Sizes

- **Fixed size:** `Header` 220 bytes; `GameSchedule` 24; `GameReliance` 31544 (all 2628 pairs); `CheckPerson` 664;
  `GameMiracle` 88; `Random` 32; empty `LegionPool` 24; `GmapSpotMgr` about 15 KB (864 spots).
- **Grow with progress:** `GameUserData` 37..72 KB (mostly the variable maps); `UnitPool` 8 B..10 KB; `Transporter`
  24 B (never initialised) to ~3.7 KB; `RidePool`; `GmapTroopMgr` up to ~57 KB (it holds full unit records);
  `HubData` 0.2..17 KB (about 150 KB inside DUNGEON restart payloads); `GameRecord`; `SectionTime`.
- **Depend on where the save was made:** `MapSeq` 1..360 KB, the sequence blocks, and `GameRestartData` 360 B..650 KB.
