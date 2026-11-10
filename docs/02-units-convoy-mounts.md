# 02. Units, convoy, mounts and battalions

This document covers `UnitPool` and the unit record with every sub-record in it, `Transporter` (the convoy, including
food and forge materials), `RidePool` (mounts and bonds), `LegionPool` (battalions), and every other copy of those
pools (GameUserData `Storage`, GameConverge `Depository`/`Temporary`, the storage sets inside each `RouteInfo`
record, world-map troops, and the restart points).

Offsets in unit tables are relative to the start of the unit record (its pointer word). "rt" names the runtime
`Unit` field the value is loaded into. Code addresses are `main+0x...` for update 131072. Field names come from the
game's own reader and, where noted, from the community decompilation. All layouts below were checked by parsing every
unit in a large set of real saves exactly to each object's end.

Name lookup: u32 ids are IdHash (the low 32 bits of FNV-1 64) and u64 ids are the full FNV-1 64 of the key
(`PID_`, `JID_`, `IID_`, `SID_`, `RID_`, `LID_`). See [doc 01 §7](01-container.md#7-hashes).

---

## 0. Reference tag bytes

Items and skills are introduced by a one-byte tag. Bits 5..6 give the kind, `(tag >> 5) & 3` **(confirmed)**:

| Kind | Meaning | Item tags (`0x41` 'A' / `0x61` 'a') | Skill tags (`0x43` 'C' / `0x63` 'c') |
|---|---|---|---|
| 0, 1 | empty, nothing follows (e.g. `0x21`, `0x23`) | empty slot | empty |
| 2 | u32 IdHash only | `'A'` + IID | `'C'` + SID |
| 3 | u32 IdHash + raw runtime struct | `'a'` + IID + 8 bytes | `'c'` + SID + u32 SkillKey |

Legacy tags with u64 FNV-1 64 keys are still accepted on load, but real saves never contain them.

---

## 1. UnitPool

Reader `main+0x6ed8c0`, writer `main+0x6ed080` **(confirmed)**.

```
UnitPool = [u32 ptr = block end][u8 version 0]
           repeat { u8 force (0xFF = end of list); u8 count; count x unit object }
           [pad to 4]
```

The writer walks forces 0..8 in order and writes a `(force, count)` header only for forces that have units. There
are 10 forces; 9 means "no force".

| Force | Meaning | Status |
|---|---|---|
| 0 | player units deployed on the battle map | confirmed (battle saves only) |
| 1 | enemy units | confirmed |
| 2, 3 | other on-map armies (allies, third parties); "on map" = force < 4 | confirmed |
| 4 | player roster not on the map (hub saves keep the whole roster here) | confirmed |
| 5..7 | more player-side lists (fallen, escaped, ...) | inferred |
| 8 | a special list used in chapter mode | inferred |

Examples: a hub save is `00 | 04 NN units | FF`; a battle save is `00 | 00 n0 units | 01 n1 units | 04 n4 units | FF`;
an empty pool is `00 FF`.

### 1.1 Unit object framing

```
unit object = [u32 ptr = record end][u8 version = 36][u8 flag] ...unit body...
```

- `flag` = 1 when the unit's force is < 4 (it is on a map). When it is 1, an AI object follows the battalion (§2.9).
- The same framing is used everywhere a unit is stored: UnitPool, RouteInfo unit lists, world-map troop unit lists,
  and restart payloads. The body is read by `Unit::Deserialize` (`main+0x6e38e0`).
- Every real save uses record version 36; the reader also accepts older versions.

---

## 2. The unit record

### 2.1 Fixed header, +0..+71

| Off | Size | Field | rt | Values / meaning | Status |
|---|---|---|---|---|---|
| +0 | u32 | pointer word = end of record | | | confirmed |
| +4 | u8 | record version | | 36 | confirmed |
| +5 | u8 | flag: AI object present | | 1 iff force < 4 | confirmed |
| +6 | u32 | `state` bit set | +0x1A0 | §2.2 | confirmed |
| +10 | u32 | PID IdHash | +0x188 (person) | | confirmed |
| +14 | u32 | JID IdHash (current class) | +0x190 (job) | | confirmed |
| +18 | u32 | `random_seed` | +0x1C0 | per-unit RNG state, re-rolled by skill procs | confirmed |
| +22 | u32 | `lottery_seed` | +0x1C4 | seed for the person's drop lottery | confirmed |
| +26 | 11 × u8 | growth remainders | +0x1C8 | order HP, Str, Mag, Dex, Spd, Lck, Def, Res, Cha, Sight, Mov. Set at unit creation to `(3 × Person growth%) mod 100`. **Never zero them** | confirmed (initial value) |
| +37 | 11 × s16 | stat **gains** | +0x1D4 | same order. Shown stat = gain + Person base + Person modifier + class base + the difficulty set chosen by `grade`. Sight and Mov gains are 0 in every real save | confirmed |
| +59 | u8 | level | +0x1F2 | 1..99 | confirmed |
| +60 | u32 | EXP | +0x1F8 | 100 per level; clamped on load to the EXP table | confirmed |
| +64 | u8 | **current HP** | +0x1EC | **one byte** | confirmed |
| +65 | u8 | pending HP | +0x1EE | 0 in every real save | confirmed |
| +66 | u8 | HP stock (extra HP bars) | +0x1F0 | 0, or 1 on some bosses | confirmed |
| +67 | u8 | max HP stock | +0x1F1 | same pattern as +66 | inferred |
| +68 | u8 | pollution | +0x1F3 | usually 0 | confirmed |
| +69 | u8 | burst | +0x1F4 | 0 | confirmed |
| +70 | u8 | corps index | +0x1F5 | 0 | confirmed |
| +71 | pad | alignment for the next object | | | confirmed |

### 2.2 `state` bits (+6)

| Bit | Meaning | Status |
|---|---|---|
| 0x1 | held: cannot act this turn | confirmed |
| 0x4 | immobile | confirmed |
| 0x8 | left the map (dead or retreated) | confirmed |
| 0x40 | unit was generated from Person data during play (every enemy, mid-battle recruits, some guests) | set site confirmed, meaning inferred |
| 0x80 | script-set; seen only on the route lord in battle (forced/fixed sortie) | inferred |
| 0x200 | script-set; always paired with 0x80 on the lord (defeat-condition unit) | inferred |
| 0x8000 | dirty/refresh flag (set when pollution changes and by other writers) | setters confirmed, meaning inferred |
| 0x40000 | dismounted | confirmed |
| 0x80000 | corps leader | confirmed |
| 0x100000 | in a corps | confirmed |
| 0x200000 | seen on some enemies | unknown |
| 0x3000000 | 2-bit HP pose: 1 full HP, 2 damaged (≥ 1/3), 3 critical (< 1/3); switches the idle animation | confirmed |
| 0x10000000 | transient "updating skills" lock; never saved set | confirmed |

Roster units (force 4) normally have state 0.

### 2.3 Variable part, in stream order

| # | Item | Format | rt |
|---|---|---|---|
| 1 | UnitJob object | object: `u8 nRanks`, nRanks × `{u8 index, u16 exp}`, `u8 nClasses`, nClasses × `{u32 JID, u16 value}` (§2.4) | +0x200 |
| 2 | map position | `u8 x, u8 z` (`FF FF` = off map) | +0x248 |
| 3 | second position | `u8, u8`; map coordinates for on-map units, `FF FF` otherwise (previous/turn-start cell, inferred) | +0x24A |
| 4 | `grade` | u8 **difficulty-set index 0..2**: picks one of the three `難易度差分` sets (initial skills, barrier items, potential skills) of the Person/Job row. The game writes it from the Difficulty byte when it places units on a map | +0x270 |
| 5 | item list object | object: `u8 0 (ignored)`, `u8 nCarried ≤ 6`, `u8 nStock ≤ 24`, then nCarried + nStock UnitItems (§2.5) | +0x33C |
| 6 | skill list objects | 6 objects, each `u8 0 (ignored)`, `u16 n`, n × SkillKey. File order = runtime lists 1, 2, 3, 4, 5, 0 (§2.6) | +0x490.. |
| 7 | barrier skill | one SkillKey; `0x23` (empty) in every real save | +0x750 |
| 8 | mount | UnitRide record (§2.7); `00` = none | +0x758 |
| 9 | battalion | UnitLegion record (§2.8); `00` = none | +0x798 |
| 10 | AI object | only if flag = 1 (§2.9) | +0x7B0 |
| 11 | u8 | always 0 | +0x829 |
| 12 | **u8 unit number** | slot index 1..250. Mount owners, battalion owners and pool lookups refer to it. **Must be unique** | +0x828 |
| 13 | class override | u8 present; if 1: u8 + u32 JID. Never present in real saves | +0x198 |
| 14 | u8 hub stamina | 0..3, default 3; training −1, theatre +1, bath/meal facilities add more; clamped (name inferred) | +0x82A |
| 15 | 4 × u8 | always 0 | +0x82B.. |
| 16 | u16 MVP points of the current map | capped at 10000; non-zero only in battle saves | +0x830 |
| 17 | UnitShape | `0x21` = 1×1, nothing follows (§2.10) | +0x250 |
| 18 | extra skill slot | u8 present; if 1: 4 raw bytes + u64 SID key. Never present | +0x7A8 |
| 19 | u8 | always 0 | +0x325 |
| 20 | u8 facing | 0 = keep; 1 → 180°, 2 → 270°, 3 → 225°, 4 → 0°, 6 → 315°, 8 → 90°, 9 → 135°, 12 → 45°. Roster units have 4 | rotation |
| 21 | UnitEdit | custom name and look (§2.11) | +0x838 |
| 22 | counters | battle statistics (§2.12) | +0x860 |
| – | end | pad to 4 = record end | |

Example tail of a mounted roster unit (bytes after the skill lists):

```
23                                       barrier skill: empty
06 603b0dfbc4e9ed48 34 02000001 cf1add00 0000 01 01 6b00 25 00
                                         mount v6 RID_馬, owner = unit 52, w4 = 0x01000002 (serial 2, lord 1),
                                         date 14490319, no name, 1 bond {person 1, order 0, 107 points}, seed 0x25, b2 0
00                                       battalion: none (flag 0, so no AI object)
00 34                                    u8 0, unit number 52
00                                       no class override
03 00 00 00 00 0000                      stamina 3, four zero bytes, MVP points 0
21 00 00 04                              shape 1x1, no extra skill, u8 0, facing 4
00                                       UnitEdit: none
05 06 {...} x6 00000000 00ffff           counters v5, 6 entries, last defeat = none
```

### 2.4 UnitJob: weapon ranks and class list

Reader `main+0x8028e0` **(confirmed)**.

**Ranks.** `{u8 index, u16 exp}`, stored only for non-zero ranks. Index order: 0 Sword, 1 Lance, 2 Axe, 3 Bow,
4 Brawl, 5 Black magic, 6 White magic, 7 Special, 8 Infantry, 9 Rider, 10 Heavy, 11 Flier, 12 Authority. Rank EXP
thresholds from E to S+: 0, 20, 35, 80, 220, 440, 690, 910, 1215, 1510, 1960, 2560. On load, exp is clamped to the last
threshold and the rank letter is rebuilt (it is not stored). The Authority rank is checked against the battalion's
required rank when a battalion is assigned.

**Class list.** `{u32 JID IdHash, u16 value}`. The value is **one u16**:

| Bits | Meaning | Status |
|---|---|---|
| 0..9 | class mastery EXP, from 0 up to the class's Job.bin `成長経験` (15 for Townsfolk, 30 tier 1, 60 tier 2, 100 tier 3) | confirmed |
| 10 (0x400) | class acquired by a class change. On a class change both the old and the new class get it; a unit still in its starting class has no 0x400 | confirmed |
| 11 (0x800) | class exam pending; never saved set | confirmed |

Example: a unit that mastered Townsfolk and then changed class has `(JID_無職, 0x040F)` plus `(new class, exp | 0x400)`.

### 2.5 UnitItem and the item list

Reader `main+0x6eb040` **(confirmed)**.

| Tag | Bytes | Meaning |
|---|---|---|
| `0x41` 'A' | 5 | u32 IID IdHash; default state (0 uses spent, no flags, count 1) |
| `0x61` 'a' | 13 | u32 IID + the 8-byte runtime UnitItem; its stale row index is overwritten from the IID |

The writer emits `'a'` only when one of the 8 bytes differs from the default:

| Byte | Field | Status |
|---|---|---|
| 0..1 | item row index (stale; rebuilt on load) | confirmed |
| 2 | **uses spent** (not uses left) | confirmed |
| 3 | flags: 0x01 equipped, 0x02 discarded/dropped, 0x08 full/exhausted, 0x10 locked, 0x20 broken, 0x40 unknown ("new" marker?) | names confirmed; 0x02, 0x08, 0x40 meanings inferred |
| 4..7 | bits 0..3 count (stock spells hold 2); bits 4..13 a 10-bit value, non-zero only on box items (box-content seed, inferred); bits 14..31 always 0 | layout confirmed |

- **Forges are separate IIDs** (`＋１`..`＋５` suffixes), not a field.
- Carried slots are 0..5; stock slots 6..29 hold spells/tomes and stock items. The reader drops carried entries beyond
  6 and stock entries beyond 24.
- The equipped flag needs the `'a'` form.

### 2.6 Skill lists and SkillKey

Runtime skill key = `index:12 | level:7 | extra:5 | top:8`. `'C'` = SID only; `'c'` = SID + raw u32 key (never seen).
Duplicates in a list are dropped on load **(confirmed)**.

| File order (0-based) | Runtime list | Content (all real saves) |
|---|---|---|
| 0 | 1 | equipped combat arts (戦技), ≤ 5 |
| 1 | 2 | **equipped passive skills**, ≤ 5 |
| 2 | 3 | extra skills granted by scripts or difficulty (e.g. `SID_難易度_弱体`, boss skills); empty for player units |
| 3 | 4 | battle command list: equipped arts + personal spells + weapon-granted arts (derived) |
| 4 | 5 | empty |
| 5 | 0 | **all learned skills** (weapon-level, class/master, personal, arts, magic) |

Runtime lists 6 and 7 are not stored. Person forced skills (`強制スキル`) and bloodmarks are computed at runtime and
are not stored either.

### 2.7 UnitRide: the mount record

Reader `main+0x93efa0`, version 6 **(confirmed)**. The same record is used inline in a unit and in `RideData`.

| Field | Size | Meaning | Status |
|---|---|---|---|
| version | u8 | 6; a single `00` = no mount | confirmed |
| RID | u64 | FNV-1 64 of `RID_...` | confirmed |
| owner | u8 | **unit number** of the rider (§2.3 #12); 0 = in the stable | confirmed |
| w4 | u32 | the mount's unique ID: bits 0..23 = serial, bits 24..27 = the lord route it was obtained on, bits 28..31 = unique-mount number (§4.1) | confirmed |
| date | u32 | in-game acquisition date as decimal YYYYMMDD (14490315 = 1449-03-15); 0 for caught/bred mounts | confirmed |
| name | u16 n + n bytes | 0 = unnamed, else `07 <utf8> 00` | confirmed |
| bonds | u8 n + n × `{u8 person, u16 v}` | person = Person `通し番号` (serial number); `v = order << 12 \| points`. Points 0..4095; bond level by thresholds 0/20/50/90/140. `order` = recency (0 = most recent rider), rewritten by the game. At most 5 bonds; a 6th evicts one | confirmed |
| seed | u8 | picks the mount's ability and favourite food | confirmed |
| b2 | u8 | only copied, never interpreted; 0 in every real save | confirmed |

**A ridden mount exists twice**, byte-identical: inline in the rider's unit record and in `RideData` with `owner` =
that unit's number. Unassigned mounts have owner 0 and no inline copy.

**Ability selection (confirmed).** The ability is chosen per mount instance from the seed:
`ability = list[rng(seed) mod len(list)]`, where `rng` is the game's xorshift128 seeded with
`s0 = seed × 0x6C078965`, `s(i) = 0x6C078965 × (s(i−1) XOR (s(i−1) >> 30)) + i` for i = 1..3, and the first output is
`(t XOR (t >> 8) XOR s3 XOR (s3 >> 19)) AND 0x7FFFFFFF` with `t = s0 XOR (s0 << 11)`. Bond levels 3..4 use the mount's
first-stage skill list and level 5 the second-stage ("+") list at the same index, so the ability unlocks at bond level
3 with the current rider and its "+" form at level 5.

### 2.8 UnitLegion: the battalion record

Reader `main+0x711ed0` **(confirmed)**.

```
u8 present (0 = none) | 8 raw runtime bytes | u64 FNV1-64("LID_...")
```

| Raw byte | Meaning | Status |
|---|---|---|
| 0 | Legion.bin row index (stale; rebuilt from the hash) | confirmed |
| 1 | owner unit number | confirmed |
| 2..3 | only copied or swapped | unknown |
| 4..7 | u32 IdHash of the world-map spot the battalion is dispatched to (exploration dispatch); non-zero protects the slot from eviction | confirmed |

The battalion level is the Legion.bin row's level; there is no per-battalion level, EXP or soldier count in the save.

### 2.9 AI object (flag-1 units only)

Version 9, reader `main+0x120ceb0`. A plain object, so it can be skipped by its end pointer **(confirmed, every
on-map unit in real saves parses exactly)**.

| Field | Type | Values seen |
|---|---|---|
| version | u8 | 9 |
| ai_flags | u32 | player 0 (or 0xD), enemy 0xD / 0x100D |
| b7bc | u8 | player various, enemy 0 (unknown) |
| band | u8 | 0/1/10/20/50/70 |
| ai_active | u8 | 0 (rarely 1) |
| priority | u8 | 0, 110..200 |
| 4 × u8 | | e.g. `[75,50,0,0]` (HP-percentage thresholds, inferred) |
| rage_op | u8 | 0 |
| rage targets | u8 x[8], u8 z[8] | |
| rage_prohibit_turn | u8 | 0 |
| 4 × u8 | | 0 |
| move_limit | u8 | 0/2/3 |
| move_limit_area | s8 × 4 | left, top, right, bottom; −1 = none |
| sequences | 4 × u64 | FNV-1 64 of AI.bin keys: activation `AI_AC_*`, mission `AI_MI_*`, action `AI_AT_*`, move `AI_MV_*`. Player units use `AI_AC_Null` |
| sequence args | 16 × `{u8 0, u8 type, value}` | type 0/1/6/7/8/10: u16 (type 1 = position `x \| z << 8`); type 2: u64 Person ref; 4: u64 Terrain ref; 5: u64 AI ref; 3 and 9: u64 refs; 11: u64 Skill ref |

### 2.10 UnitShape

A tag with `(tag & 0x60) == 0x40`: u8 size, then size² cell bytes (barrier HP per cell). Any other non-zero tag
(`0x21`): 1×1, nothing follows **(confirmed)**.

### 2.11 UnitEdit: custom name and look

Reader `main+0x9885b0` **(confirmed)**.

- u8 version: 0 = none (1 byte); 3 in real saves.
- `u16 n`, then n bytes `07 <utf8> 00` (n counts the 07 and the NUL).
- 5 look bytes: body type (1/2), outfit, hairstyle, skin, hair.
- Version 2 only: 2 more bytes.

Only the protagonist (Eshmel; default name "Eshmel") has one. The same name and look also sit in GameConverge
`CommonInfo` and in GameRecord MVP name tags ([doc 04](04-routes-world-hub.md)).

### 2.12 Counters: battle statistics

Reader `main+0x9bdc00` **(confirmed layout)**.

- u8 version (5); u8 n; n × `{u32 IdHash key, u32 value}`.
- u32 + 3 bytes: the IdHash of a battle-map name plus (lord, part, chapter), default `0, 00 ff ff`. Inferred: where
  the unit was last defeated or retreated.

Keys are IdHash of names such as `BMAP 総出撃回数` (sorties), `BMAP 総戦闘回数` (battles), `BMAP 総撃破回数` (kills),
`BMAP 総特技回数` (art uses), `BMAP 総死亡回数` (deaths), `CMDBTL MVP回数`. Unknown keys are dropped on load.

---

## 3. Unit editing rules

1. The **unit number** (+0x828) must be unique in 1..250. Mount and battalion owners refer to it.
2. A ridden mount is stored twice. Change the inline copy and the RideData copy together.
3. Class-list values are u16: `exp (≤ 成長経験) | 0x400 if acquired by class change`.
4. **HP is a u8 at +64.** Write one byte.
5. Don't zero the growth remainders at +26. For a new unit use `(3 × Person growth%) mod 100` per stat, Sight/Mov 0.
6. `grade` should match the save's Difficulty byte.
7. To add a unit, clone a real player-unit record, then set PID/JID/stats/items with fresh seeds, real growth
   remainders and a unique unit number. The game's join code reuses an existing unit with the same PID.
8. Any size change needs a splice ([editing guide](editing-guide.md#2-splicing-changing-the-size-of-anything)).
9. Restart points embed their own UnitPool/RidePool/Transporter. RouteInfo/Storage/Depository are separate pools,
   not mirrors.

---

## 4. RidePool and RideData

Block reader `main+0x94a720`, list reader `main+0x9492a0` **(confirmed)**.

```
RidePool = struct [u32 ptr][u8 version 3][pad] RideData object, member table {0x09B9F71F "RideData", position}
RideData = [u32 ptr][u8 0 (ignored)][u16 n][n x slot]      slot = 00 (empty) | UnitRide record (§2.7)
```

- n = 200 in every player stable (the cap; "Animal storage is full"). Empty stables written elsewhere by the game
  (Storage, storage sets) use n = 0.
- Every mount the player owns is here, including ridden ones (with owner set).

### 4.1 Mount IDs and the serial counter

Assignment code `main+0x9482f4` **(confirmed)**. When the game creates a mount with ID 0:

```
serial     = (RideSerial + 1) AND 0xFFFFFF ;  RideSerial = serial
lord       = current route lord (GameUserData lord byte) AND 0xF
unique     = Ride.bin row byte +0x6C
w4         = unique == 0 ? serial | lord << 24
                         : unique << 28 | lord << 24      (serial 0)
```

`RideSerial` is GameUserData body+39 ([doc 03](03-game-user-data-and-variables.md#1-gameuserdata-body)). It must stay
**≥ the highest serial** in every RidePool copy. Example: generic mounts obtained on Cai's route have
`w4 = 0x01000000 | serial`; a unique mount number 1 obtained on Leda's route has `w4 = 0x14000000`.

---

## 5. LegionPool and LegionData

Block reader `main+0x946d40`, list reader `main+0x943f60` **(confirmed from code)**.

```
LegionPool = struct [u32 ptr][u8 version 3][pad] LegionData object, member {0x7644ADD3 "LegionData"}
LegionData = [u32 ptr][u8 0 (ignored)][u16 n][n x slot]    slot = 00 | 01 + 8 raw bytes + u64 LID   (§2.8)
```

- Capacity 300.
- When the list is full, the slot evicted is the one with the lowest priority:
  `row | (Legion row +0x33) << 10 | 0x4000 if owner ≠ 0 | 0x8000 if raw u32 @4 ≠ 0`.
- No real save examined owns a player battalion, so this list layout is confirmed from code and from inline enemy
  records only.

---

## 6. Transporter and TransporterData

Reader `main+0x8a43e0` **(confirmed)**.

```
Transporter     = struct [u32 ptr][u8 version 0][pad] TransporterData object, member {0xEFBE8231 "TransporterData"}
TransporterData = [u32 ptr][u8 version 1][u16 n][n x slot]     slot = u16 qty; if qty != 0: UnitItem (§2.5)
```

- **Regular slots: 999**, positional, holes allowed (slot i < 999 goes to runtime slot i). Regular items always
  have qty 1.
- **Fixed stack slots: one per stack item.** A stack item is an Item.bin row with `種類` (kind) = 14 and `用途` (usage)
  ∈ {13, 14, 18, 19, 20, 22}. The writer emits the 999 regular slots, then one slot per stack row in table order. In
  update 131072 there are 330 stack rows, so **n = 1329** in every initialised convoy.
- On load the reader moves a stack item into its own fixed slot by item lookup, whatever its file position.
- Stack quantity cap: **999**.
- Stack slots hold forge materials, gold bullion (金塊) and all food ingredients. Food = usage 20, with `素材`
  0 meat, 1 fish, 2 vegetables/plants; skip the tutorial freebies `IID_試供餌` / `IID_試供種`.
- **n = 0** (`01 00 00`) means the convoy has never been initialised (before it is unlocked). Don't add items to an
  n = 0 convoy without writing all 1329 slots.
- An empty slot is `00 00`; a filled slot is 7 bytes (`'A'` form) or 15 bytes (`'a'` form), so filling slots grows
  the object.
- On load, regular slots lose flag bits 0..2 (equipped/discarded); uses, locked/broken/full and the 10-bit value are
  kept.

---

## 7. Every copy of the pools

All copies use the same TransporterData / RideData / LegionData objects as named members of a small struct.

| Copy | Where | Framing | Meaning | Status |
|---|---|---|---|---|
| main pools | blocks `Transporter`, `RidePool`, `LegionPool` | §4..§6 | the active army | confirmed |
| `GameUserData/Storage` | GameUserData member | struct without a lead byte; member order Transporter, Legion, Ride | the Part-1 army's convoy, parked while the War Arc is played (seen holding a full convoy in a Part 2 save) | layout confirmed, fill timing inferred |
| `GameConverge/Depository` | GameConverge member | object wrapping `{u8 0; TransporterData; RideData; LegionData}` | the Drifting Reliquary, shared across routes. The War Arc merge moves the live pools into it | confirmed |
| `GameConverge/Temporary` | GameConverge member | same as Depository | purpose unknown; always empty | layout confirmed |
| RouteInfo set A | in each route record | `{u8 0; TransporterData; RideData; LegionData}` | snapshot of the convoy/mounts/battalions taken at that route's first War Arc merge | confirmed (copy) |
| RouteInfo set B | in each route record (route version ≥ 11) | same | pending deliveries for that route: drained into the live pools and cleared by `main+0x8fb140`; no code that fills it was found | drain confirmed, fill unknown |
| world-map troops | `GmapTroopMgr` | full unit records | enemy/ally troop members | confirmed |
| restart points | embedded payloads | full UnitPool / RidePool / LegionPool / Transporter | snapshots for "Restart chapter/battle" | confirmed |

The RouteInfo record itself is described in [doc 04 §1.2](04-routes-world-hub.md#12-the-route-record).
