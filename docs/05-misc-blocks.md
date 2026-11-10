# 05. Battle suspend data and the remaining blocks

This document covers:

1. `MapSeq`: battle suspend data, including the Rewind history
2. `SubquestData`
3. `Calendar`
4. `RecommendData`
5. `Random`
6. `GameSound`
7. `GameUserData/GameSituation`
8. `CmdBtlSeq` (restart payloads only)
9. `GmapSeq`

Code addresses are `main+0x...` for update 131072. Every member layout starts after its 4-byte pointer word. These
layouts were checked by parsing every real save that contains the block exactly to each object's end.

---

## 1. MapSeq: battle suspend data

`MapSeq` (version 0, FNV-1 32 `33BD31C2`) is present only in saves made during a battle (and in MAP restart
payloads). Writer `main+0x72bba0`, reader `main+0x72bef0` **(confirmed)**. It is a struct with 10 members, written in
this order:

| # | Member | FNV-1 32 | Content |
|---|---|---|---|
| 1 | `Situation` | F59699CB | battle-map state (§1.1) |
| 2 | `Camera` | B804D7B0 | camera (§1.2) |
| 3 | `Cursor` | F78654AF | cursor cell |
| 4 | `Image` | 20B1D94C | changed terrain cells |
| 5 | `Trick` | 4C2C1662 | map gimmicks (§1.3) |
| 6 | `AIValueFlags` | 6BA115EB | script AI values (always empty) |
| 7 | `AIValuePosLists` | CE0C58B7 | script AI position lists (always empty) |
| 8 | `MapObjectManager` | 4ECE5371 | map-object states (§1.4) |
| 9 | `MapEffect` | 13FDD09E | persistent map effects |
| 10 | `MapHistory` | EDCAA2DB | the Rewind history (§1.5); written only while history recording is active |

Members 1..5 get a framework version byte; members 6..10 write their own version byte.

### 1.1 Situation (v12)

| Field | Type | Meaning | Status |
|---|---|---|---|
| version | u8 12 | | confirmed |
| force controllers | 4 × u8 | controller of forces 0..3: 1 player, 2 AI | confirmed |
| flags | u32 | 0x100 danger-zone display, 0x200 fog of war (`索敵中`), 0x800 raining (`雨天中`), 0x1000 in the underworld (`冥界中`), 0x1E rule flags; 0x4/0x400/0x2000 seen but unknown | named bits confirmed |
| current force | u8 | the phase being played | confirmed |
| u8 | | 0 | |
| turn | u16 | **turn number** (= file header 0x2B) | confirmed |
| blessing uses | u8 | Invoke-Blessing uses left on this map; `0xFF` = not yet initialised | confirmed |
| 2 × u8 | | rule bytes | layout |
| records[2] | 2 × `{string name; 4 × {string detail, u16 value}}` | win and lose condition texts: `MID_WINRULE_<map>_n` and `MID_LOSERULE_<map>`, with per-argument details (value `0xFFFF` = none) | confirmed |
| counters | u16, u8, u8, u16, u8, u8 | per-battle counters; 0 at map start | layout only |
| BonusExpList | anonymous object `{u8 2, u8 n, n × {u32, u32, u32}}` | bonus-EXP queue (at most 100); always empty | confirmed |
| honor EXP | u32 | | confirmed |

The same code registers the battle-local (ローカル) variables such as `勝利`, `敗北`, `継続`, `スコア表示`.

### 1.2 Camera, Cursor, Image

- **Camera** (v1): `u8 0, f32 A, f32 B, 4 × u8 0`. A is 90.0 or 180.0, B 97..138 in real saves (yaw in degrees and
  distance, inferred).
- **Cursor** (v0): `u8 x, u8 z`, the map cursor cell **(confirmed)**.
- **Image** (v0): `u8 width, u8 height, u16 n`, then n × `{u8 layer (0 base, 1 overlay), u8 x, u8 z, u64 Terrain
  ref}`: one entry per cell whose terrain differs from the map's original **(layout confirmed)**.

### 1.3 Trick (v4): map gimmicks

`u16 n`, then n inline entities (no object framing) **(confirmed)**:

| Field | Type | Meaning |
|---|---|---|
| version | u8 8 | |
| type | u8 | 1 destructible terrain, 2 area effect (e.g. `TID_瘴気`), 3 control device, 5 treasure chest, 13 target terrain, 15 warp-forbidden zone |
| x, z, w, h | 4 × u8 | rectangle |
| cells | u8 n + n × (u8 x, u8 z) | |
| life, max life | 2 × u8 | HP of destructible tricks |
| u8 | | |
| force | u8 | 9 = none, 0 = player |
| status | u8 | 9 = none |
| 2 × u8 | | |
| flags | u16 | 0x01 enabled, 0x10 disabled, 0x04/0x08 "done" (meaning by type) |
| angle | s16 | |
| item, legion, terrain | 3 × u32 IdHash | Item / Legion / Terrain (`TID_*`) rows |
| target kind | u8 | 1 or 2: a u32 follows; 3: u64 Person ref; 4: u64 Skill ref; 0: nothing |
| owner | u8 | |

### 1.4 MapObjectManager (v1) and MapEffect

```
MapObjectManager:  u16 n | n x {u32 id, u8 active} | n x {u32 id, u8 state}
MapEffect:         u8 0 | u16 n | n x {string effect name, u8 x, u8 z}
```

- **id = FNV-1 32 of the map-object instance name** in the map layout, e.g. `Bg_Terrain_A000_001`,
  `Obj_Fence_A000_003` **(confirmed)**. `active` = the object's flag 1; `state` = its object state (2 = broken/"done",
  inferred).
- MapEffect example: `UI_BG_ExitZone` at a cell.
- `AIValueFlags` = `u8 0, u16 n × {u32, u32}, u8 0`; `AIValuePosLists` = `u8 0, u8 nLists × {u8 n × {u8 x, u8 z,
  u8 v}}`. Both are always empty; entry meaning unknown.

### 1.5 MapHistory (v3): the battle Rewind history

| Field | Type | Meaning | Status |
|---|---|---|---|
| version | u8 3 | | confirmed |
| recording | u8 | 1 while recording (top-level saves), 0 in restart copies | inferred |
| u8 | | 0 | |
| u32 | | 0x10000 / 0x10402 | unknown |
| slots | 100 × `{u16 n, n × {u8 type, record}}` | **one slot per turn** (slot t = turn t; slot 0 unused), records in step order | confirmed |
| rewind points | u32 | number of DataAll + DataDiff records | confirmed |
| latest turn | u16 | | confirmed |
| u16 | | 0 | |
| temp | u16 n + n × `{u8 type, u8 0, u8 0, u8 unit, u8 flag}` | units added/deleted in the current step; always empty | confirmed |
| log | LogInfo | the current log entry | confirmed |
| members | 5 named members | below | confirmed |

**Record types:**

| Type | Record | Body |
|---|---|---|
| 0 | DataAll (full snapshot; first record of each turn) | header; `u16 nKeys`; `u32 bufSize`; nKeys × key; buffer; LogInfo |
| 1 | DataDiff (one step) | header; `u16 n`; n × `{u8 0, key, u16 c, c × {u8 2, u8 kind, u32 a, u32 b, u32 size, data}}`; LogInfo. Binary patches against the previous record for that key (kind 1 = replace b bytes at offset a, inferred) |
| 2 | DataAdditionalUnit | header; key; key.size bytes |
| 3 | DataDeletedUnit | header; `u8 unit, u8` |
| 4 | DataAdditionalNewType | as type 2 |

- **Header:** `u8 version, u8 0, u16 turn, u8 phase (0 player, 1 enemy), u16 step`.
- **Key:** `u8 1, u32 offset, u32 size, u8 op, u8 index`. `op` names the sub-serializer whose mini-stream the payload
  holds: 0 Unit (index = unit number), 1 UnitMgr, 2 Trick, 3 Variable (index = category), 4 AI, 5 Random, 6 Reliance,
  7 Transporter, 8 BonusExp, 9 UserData, 10 Rule, 11 Terrain, 12 Miracle, 13 MapObj, 14 Rain, 15 Effect, 16 Sound,
  17 AIValue, 18 RidePool, 19 LegionPool, 20 HonorExp.
- **LogInfo** (41 bytes): `u8 2, u8 kind, u8 unit, u8 x, u8 z, u32 value, u64 name0, u64 name1, u64 name2, u64 name3`.
  kind: 1 phase start, 2 move/wait, 3 attack, 4 heal, 5 assist, 6 arts, 7 special, 8 gambit, 9 rage, 10 item,
  11 escape, 12 chest, 13 miracle, 14 skill, 15 revenge kill, 255 general. name0 = actor PID, name1 = target (PID,
  IID, SID or TID), name2 = actor, name3 = actor's JID (all FNV-1 64). x, z = the actor's position.

**Members:**

| Member | FNV-1 32 | Layout | Meaning |
|---|---|---|---|
| `MapHistoryMapObjMgr` (v1) | C93CB89C | `u16 n, n × {string name, u8, u8, u8}, u16` | recorded map objects |
| `MapHistoryRelianceMgr` (v3) | 0A30C96E | `u16 n, n × {u64 PID A, u64 PID B, u16 points, u16 count, u32 stamp}` | support snapshot of the deployed pairs at battle start |
| `MapHistoryUnitMgr` | EBC9DD2D | `u8 0, u8[256]` | per unit number: 1 = exists on the map |
| `MapHistoryVariableMgr` (v11) | 5CEE1B03 | `u8 11, u8 flags`; 3 × `{u32 n, n × {u64 key, u32 value}}`; 12 × `{u8 category, u8 flag, u32 n, n × u64 key}` | **variable baseline at battle start**: list 0 = variables with other values, list 1 = value 1, list 2 = value 0; the 12 groups list, per category, the variables changed during the battle |
| `MapHistoryNewTypeMgr` | A195ED3F | `u8 0, u16 n, n × DataAdditionalNewType` | always empty |

---

## 2. SubquestData (v4)

Writer `main+0x1282620` **(confirmed)**.

```
u8 version 4
u32 n                     n x {u32 IdHash(SQID_...), u8 lord, u8 state, u8 flag}       (7 bytes each)
u32 n + n x u64           shared list: FNV-1 64 of SQIDs cleared on any route
10 x per-lord slot        {u32 n + n x u64 list, u32 mask}   (slot L = lord L, L = 0..9)
pad to 4
```

- Records are per (sub-quest, lord); n is 928 in update 131072. Records whose row is null are written with id 0 and
  are always 0/0/0.
- **state:** the same scheme as paralogues: 0 not offered, 1 offered, 2 accepted, 3 report pending, 4 done, 5 missed
  (inferred from the shared state machine).
- **flag:** bit 0 = "new quest" notice already shown (inferred); **bit 1 = auto-completed by the cross-route rule
  below (confirmed)**; bit 2 cleared together with bit 1 on load for ineligible rows (unknown).
- **Per-lord slot** (written at route start): the list is a snapshot of the shared list taken when route L started;
  the mask has bit L set for each `1部クリア_<L>篇` ≠ 0. A record of the current lord that is still state 0, whose row
  allows it, and whose SQID (or parent SQID) is in the snapshot is auto-completed (state 4, flag bit 1, no reward).
- Slots that were never written hold **uninitialised memory** (odd large u32 values with bits above 9 set). Leave
  them alone.
- A route start also resets every record of the new lord (state and flag = 0).

---

## 3. Calendar (v6)

Writer `main+0x13045c0`, reader `main+0x1305120` **(layout confirmed, purposes partly inferred)**.

| # | Encoding | Meaning |
|---|---|---|
| 1 | u8 version 6 | |
| 2 | u8 n (0..3), n × `{u16 y, u8 mo, u8 d, u8 flag}` | up to 3 day reservations on the calendar; each must be a future day with a calendar entry, and is freed once the date has passed. What is reserved is not identified |
| 3 | u8 | unknown |
| 4 | u8 n + n × u64 | SSIDs (paralogue events already shown on the calendar, inferred) |
| 5 | u8 n + n × u64 | always empty |
| 6 | u8 n + n × u64 | always empty |
| 7 | u16 y, u8 mo, u8 d | a date (last day the calendar notify process handled, inferred) |
| 8 | u8 n + n × u64 | always empty |

---

## 4. RecommendData (v0)

`u8 0, u32 n, n × u32` **(confirmed)**. Each u32 is the IdHash of an `RCID_<LORD>_<chapter>_...` row (or 0 for a
null row): the hub's "recommended activities" list for the current chapter. n is 0 at chapter start, then 12.

---

## 5. Random (v0)

`u8 0, pad`, then an anonymous object `{u8 0, u32 s0, s1, s2, s3}`: the global **xorshift128** state (seeding
multiplier `0x6C078965`) **(confirmed)**. Editing it only reseeds later random rolls.

---

## 6. GameSound (v7)

Writer `main+0x817920` **(field order confirmed; meanings inferred)**. Strings are `u16 (len+1)`, bytes, NUL, or
`u16 0`.

| # | Encoding | Meaning (inferred) |
|---|---|---|
| 1 | u8 7 | version |
| 2 | string | current BGM cue (e.g. `BGM_G_MAP1`) |
| 3 | string | BGM queued under the current one |
| 4 | string | current ambience set (e.g. `Gmp_000`) |
| 5 | 5 × u8 | play-state flags |
| 6 | 32 × string | per-slot BGM table |
| 7 | 16 × `{u8, string}` | 16 channels: volume step (10) + cue |
| 8 | 45 × u8, u8, u8 | unused indices (`0xFF`) |
| 9 | string, string | BGM to resume after the current scene (e.g. `BGM_Hub5`) |
| 10 | u8; s32; u8; string; u8; u8; u8; string | saved BGM record 1: playback position and track/section |
| 11 | s32; u8; string; u8; u8; u8; string | saved BGM/ambience record 2 |
| 12 | 8 × u32 | unknown |

This block only restores the sound state. Leave it alone.

---

## 7. GameSituation

A GameUserData member (FNV-1 32 `3F7E8625`) with **no content**: the writer emits one constant version byte `03` and
the reader reads nothing. The 8 bytes on disk are the pointer word, `03` and 3 padding bytes **(confirmed)**. The
live battle situation is in `MapSeq/Situation`.

---

## 8. CmdBtlSeq (v0)

Present only in CMDBTL restart payloads: the resume data for a command battle (clan skirmish) **(confirmed)**. It
holds one anonymous object:

| Field | Type | Meaning |
|---|---|---|
| version | u8 5 | |
| field | string | command-battle field key, e.g. `Field_帝都ダグシオン下水道1` |
| string | | empty |
| 2 × u8 | | the second is written only if the first is set (unknown) |
| u8 | | 0 |
| 2 × u16 | | unknown |
| units | u16 n + n × object `{u8 0, u64 PID ref, u16 n1 + n1 × u64 ref, u16 n2 + n2 × u64 ref, u8, u8, u8 side, u8, u8, u32}` | combatants; side 0 player, 1 enemy, 2 reinforcement (inferred) |
| string | | empty |
| labels | u16 n + n × string | enemy band names (e.g. `MID_Clan_Bandits`) |
| 2 × u64 | data refs | 0 |
| u64 | | unknown |

---

## 9. GmapSeq (v2)

Present only in saves made on the world map **(confirmed)**. A struct with two members:

| Member | FNV-1 32 | Layout |
|---|---|---|
| `GmapCursor` (v2) | 3D0EA394 | `u8 2, f32 x, f32 y, f32 z` |
| `GmapCamera` (v0) | 7218293F | `u8 0, f32 x, f32 y, f32 z, u32 0` |
