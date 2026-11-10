# 04. Routes, world map, hub and paralogues

This document covers:

1. `GameUserData/GameConverge`: route records, cross-route data, the Drifting Reliquary
2. the world map: `GmapSpotMgr`, `GmapTroopMgr`, `GameGmapData`
3. `HubData` (with its three embedded mini-saves) and `HubSeq`
4. `SideStoryData` (paralogues)
5. the save card and the header masks

Code addresses are `main+0x...` for update 131072. Offsets written `+N` on runtime objects (e.g. "spot +176") are the
game's in-memory field offsets; they are used here as field names, and the file position of each field is given in
the layout tables.

---

## 1. GameConverge

GameConverge (version 6, member of GameUserData, FNV-1 32 `A5D56C94`) is the cross-route store. Writer
`main+0x8f5300`. It is **absent from restart payloads**, so restarting never rolls back cross-route data
**(confirmed)**.

### 1.1 Framing

`[ptr][u8 version 6]` followed by four members:

| Member | FNV-1 32 | Read if version | Content |
|---|---|---|---|
| `RouteInfo` | 72218468 | ≥ 3 | one record per route |
| `CommonInfo` | 79B0BBD2 | ≥ 4 | data shared by all routes |
| `Depository` | FBD85F53 | ≥ 5 | the Drifting Reliquary |
| `Temporary` | 3801F62A | ≥ 6 | same shape as Depository; purpose unknown |

### 1.2 The route record

`RouteInfo` is a struct with 9 members keyed by FNV-1 32 of the route name: `カイ篇`, `ディートリヒ篇`, `セオドラ篇`,
`レダ篇`, `ベルトラン篇`, `タリムーン篇`, `オルヘル篇`, `アナトリア篇`, `イシュマール篇`. Each route record is a struct with
one named member (its `GameRecord`), so its pointer word has count 1. Per-route writer `main+0x8fe0e0`.

Body, in write order (objects are 4-aligned; bare bytes are not):

| # | Field | Encoding | Meaning | Status |
|---|---|---|---|---|
| 1 | version | u8 16 | | confirmed |
| 2 | n | u16 | number of recorded units | confirmed |
| 3 | units | n × unit object `[ptr][u8 36][u8 flag][unit body]` | full unit records in UnitPool format ([doc 02](02-units-convoy-mounts.md#2-the-unit-record)) | confirmed |
| 4 | gold | u32 | gold carried over; at each merge it becomes max(stored, current gold) | confirmed |
| 5 | merges, Part 1 | u8 | +1 (capped at 100) each time this route's units are recorded with part ≠ 2 | confirmed |
| 6 | merges, Part 2 | u8 | the same for part 2 | confirmed |
| 7 | clears, Part 1 | u8 | +1 (capped at 100) at the route end when part ≠ 2 | confirmed |
| 8 | clears, Part 2 | u8 | the same for part 2 | confirmed |
| 9 | best difficulty, Part 1 | u8 | 0..2; **3 = never cleared**. At the route end it becomes the higher of the stored value and the *lower* of the run's Difficulty (body+2) and start difficulty (body+49) | confirmed |
| 10 | best difficulty, Part 2 | u8 | | confirmed |
| 11 | clear playtime, Part 1 | u16 h, u8 m, u8 s | playtime counter at the route end | confirmed |
| 12 | clear playtime, Part 2 | u16 h, u8 m, u8 s | | confirmed |
| 13 | set A | anonymous struct `{u8 0; TransporterData; RideData; LegionData}` | convoy/mounts/battalions recorded with the units (reset and filled at the route's first merge) | role confirmed |
| 14 | set B | same (route version ≥ 11) | pending deliveries: drained into the live pools by `main+0x8fb140`; no filler found | drain confirmed |
| m | GameRecord | named member (route version ≥ 15) | per-route war record ([doc 03 §8](03-game-user-data-and-variables.md#8-gamerecord-war-record)) | confirmed |

Rules **(confirmed)**:

- Units are merged into the route record **only at the War Arc clear** (unit merge at `main+0x8f1d40`, timeline
  command 46). A route with only Part 1 cleared has no units.
- The clear fields are written at the route end (`main+0x8f2920`, timeline type 58 "ルート終了").
- On load, recorded units whose person doesn't resolve are dropped.
- "View Clear Data" opens only if `2部クリア_<lord>篇` is set, and lists the route's recorded units.

Route GameRecord: `order_p1`/`order_p2` are global clear-sequence indexes. A first clear gets the number of
non-negative order slots over all routes and both parts (0-based); a re-clear moves the route to the last place and
decrements the higher ones.

### 1.3 CommonInfo (version 12)

`CommonInfo` has no version byte of its own; it wraps one anonymous struct (version 12) with three named members.

| # | Field | Encoding | Meaning | Status |
|---|---|---|---|---|
| 1 | version | u8 12 | | confirmed |
| 2 | spots | u16 n + n × `{u32 IdHash(spot key), u8 state, u8 flags}` (6 bytes each, unaligned) | **cross-route world-map knowledge**: every spot's state (+176) and flags (+178), captured at the route end. At a route start each spot gets `state = max(current, saved)` (except attribute-0x80 spots; it also sets the spot's reveal variable) and `flags = flags OR (saved AND NOT 2)` (the checkpoint-passable bit is dropped). Unknown keys are removed on load | confirmed |
| 3 | relay vars | u16 n + n × `{u32 IdHash(name), value}` (value encoded as in variable maps) | a snapshot of the **中継 variable category** (hub fast-travel points, support conversations watched), written back into the variables at a route start. Unknown keys are dropped on load | confirmed |
| 4 | profile | `u8 3; u16 n; 07 + utf8 + 00; u8 b0..b3; u8 b4`, or a single `u8 0` | **the protagonist's custom name and appearance** (the default name is `\x07Eshmel`). b0 = body type (1/2, inferred); b1..b4 other creator choices | layout confirmed |
| 5 | b152 | u8 | always 0 | unknown |
| 6 | b156 | u8 | always 0 | unknown |
| 7 | last route lord | u8 | set at the route end, cleared later (a pending "route finished" notice, inferred) | confirmed |
| 8 | last route part | u8 | part at the route end | confirmed |
| 9 | first lord | u8 | the first lord route ever started (0 in prologue-only saves) | confirmed |
| m | `GameUnitRecord` | member | per-person counters (§1.4) | confirmed layout |
| m | `GameMiracle` | member, same format as GameUserData's | **cross-route blessing EXP maximum**, merged per blessing at route end/start. Not a zero snapshot | confirmed |
| m | `GameOverCount` | member: `u16 n + n × {u32 IdHash(map), u16 count}` | game overs per battle map across routes | confirmed |

### 1.4 GameUnitRecord

`u8 0; u16 n`, then n records **(layout confirmed)**:

| Field | Encoding | Meaning |
|---|---|---|
| person | u32 IdHash(PID) | |
| counter A | `u8 kind`; kind 0 → `u32 Part1, u32 Part2` | per-part pair (battles, inferred) |
| counter B | same | victories/kills (inferred) |
| counter C | same | a third per-unit count (unknown) |
| route mask | u16 | bit L set for routes on which the person joined as an outside recruit (inferred) |

### 1.5 Depository and Temporary

- **Depository** (the Drifting Reliquary): an object wrapping `{u8 0; TransporterData; RideData; LegionData}`. At the
  War Arc merge the party's convoy stacks, mounts and battalions are moved into it; the Obelisk Chamber reliquary
  sells them back **(confirmed)**. Empty in every real save examined.
- **Temporary**: same layout, written and read like Depository, always empty. No reader of its contents was found;
  purpose unknown.

---

## 2. World map

### 2.1 GmapSpotMgr (v20)

Manager writer `main+0x135ed40`, reader `main+0x135f020`; per-spot writer `main+0x1352970`, reader `main+0x1352bc0`
**(confirmed)**.

| Offset | Type | Meaning |
|---|---|---|
| +0 | u32 | pointer word (block end) |
| +4 | u8 | manager version = 20 |
| +5 | u16 | n = 864 spots |
| +7 | n × entry | `u32 key` written **immediately** after the previous byte (it may be unaligned), **then** pad to 4, then the spot object |
| tail | u8 | event-block flag: set by timeline type 40, cleared by type 41; while set it also suppresses the neighbour auto-reveal. 0 in saves |
| | u32 | a spot key with no writer apart from the loader (vestigial); always 0 |
| | u16 n + n × u32 | queue of spots to reveal (transient; always empty in saves) |
| | u16 n + n × u32 | queue of hidden spots adjacent to the party (transient; always empty in saves) |

**Spot key = IdHash of the node name.** The order is exactly the node table order of `GameData/Gmap/Data.bin`.
Node names are either spot names from Gmap.bin (towns, fields, branch points `分岐点...`) or route waypoints named
`<spotA>_<spotB>_<n>`. Entry 0 is `帝都ダグシオン` with key `0x621A6FD3`, stored unaligned at block offset +7.
**Read the key first, then align.**

Node kinds (Data.bin; equal to the Gmap.bin `種類` where the row exists):

| Kind | Meaning | Kind | Meaning |
|---|---|---|---|
| 0 | capital / city | 7 | landmark / shrine |
| 1 | town / village / port | 8 | fort (no nodes) |
| 2 | temple | 9 | dungeon |
| 3 | carriage station | 10 | farmable land (always has the farm sub-object) |
| 4 | checkpoint (関所) | 11 | mid-spot |
| 5 | field (exploration dispatch target) | 12 | branch point |
| 6 | animal field (has the animal sub-object) | 13 | route waypoint `A_B_n` |

Kinds 0..4 and 7 are named from the spot names (inferred); the others are confirmed in code or by their names.

#### Spot object (version 20)

| Field | Type | Runtime field | Meaning | Status |
|---|---|---|---|---|
| pointer word | u32 | | object end | confirmed |
| state | u8 | +176 | visibility: 1 hidden, 2 drawn route, 3 revealed | confirmed |
| saved | u8 | +177 | event-block restore value: 0 or 3 | confirmed |
| flags | u8 | +178 | bit 0 arrived/visited; bit 1 checkpoint passable (kind 4); bit 2 reconstructed (Part 3) | bits 0..1 confirmed, bit 2 inferred |
| hasAnimal | u8 | | 1 = animal sub-object follows (kind 6 only) | confirmed |
| animal | `u8 4`, then an object `{u64 bait IID, u8 state, u64 target}` | | lure state | confirmed |
| hasFarm | u8 | | 1 = farm sub-object follows (kind 10 only) | confirmed |
| farm | object `{u8 1, u64 planted IID}` | | crop; 0 = empty plot | confirmed |
| hasAttach | u8 | | 1 = a timed world-map action is in progress | confirmed |
| attachment | below | | | confirmed |
| recruiting | u8 | | recruiting in progress | confirmed |
| pad | | | to 4 | |

So in a plain spot the state byte is at object+4, saved at +5 and flags at +6. Typical object sizes (with the pointer
word): 12 bytes plain, 32 for a farm plot, 40 for an animal field.

**State values (confirmed):**

- **1 = hidden.** Set at spot init and by the timeline type 37 reset (which also clears flags and recruiting and
  deletes the attachment, but leaves `saved` alone).
- **3 = revealed.** Written by timeline type 38 ("spot list") and by the exploration reveal. Both skip spots whose
  Gmap.bin attribute has bit 0x80 (event-only spots) and, when the spot has a data row, also set a per-row variable.
- **2 = drawn route.** Written by timeline type 39 (and at an event-block start) on route waypoints, overriding 3.
  The game treats any state ≠ 1 as shown/selectable; some checks (farm, unrevealed-neighbour checks) need exactly 3.
- **saved = 3** marks "was revealed when a story map-event block began" and lingers afterwards; it has no gameplay
  effect outside a block. The reader forces `saved = 3` when it loads state 2.
- **flags:** bit 0 is set when the party reaches the spot (map colour "arrival"); bit 1 makes a checkpoint passable
  (`kind != 4 || flags & 2`); bit 2 is a Part 3 "reconstructed" marker.
- Valid combinations seen in real saves include (1,0,0), (3,0,0), (3,0,1), (3,3,0), (3,3,1), (3,3,3) and (3,0,3) on
  checkpoints, (2,3,0) and (2,3,1) on waypoints, (1,3,0).

#### Spot attachments

The reader takes `u8 kind`, constructs the attachment, then reads `u8 version` (9).

```
common head:  u8 kind | u8 version 9 | u64 time 1 | u64 time 2 | u8 +0x29 | u8 +0x28
GmapTime:     year << 32 | month << 24 | day << 18 | hour << 12 | minute << 6 | second
```

- **time 1** = the start / last-progress time; **time 2** = the due time (the attachment is finished when the clock
  reaches it) **(confirmed)**.
- `+0x28` = done/collected flag for completion kinds 2 and 5; `+0x29` unknown.

| Kind | Action | Extra bytes after the head | Status |
|---|---|---|---|
| 0 | Animal lure (おびき寄せる) | none | confirmed |
| 1 | Farm (畑を作る) | u8 growth stage; u8 n = harvest stocks pending; u8 boosted-harvest flag (inferred); u64 fertiliser IID. The stock items themselves are not saved: they are re-rolled on load | layout confirmed |
| 2 | none (the reader builds nothing) | | confirmed |
| 3 | Recruit (兵士募集) | u8 (recruits already taken, inferred) | layout confirmed |
| 4 | Dispatch: explore / dungeon / fort | u8 n; n × u64 FNV-1 64 keys of the dispatched persons; u8 progress counter (inferred) | layout confirmed |
| 5 | Reconstruction (復興) | u8 elapsed work (inferred) | layout confirmed |

Kinds 3..5 have never been seen in a real save; their layouts come from code.

### 2.2 GmapTroopMgr (v7)

Writer `main+0x1377fe0`; troop writer `main+0x136ab00` **(confirmed)**.

```
[ptr][u8 version 7][u8 troop count] troop objects ... | u8 (always 255) | u8 n + n x u8 (always empty)
```

**Troop object (version 24).** Troop 0 is always the **player's party**.

| Field | Type | Meaning | Status |
|---|---|---|---|
| version | u8 | 24 | confirmed |
| clan | u64 | a hash reached through the troop's data row; 0 for the party and lords | unresolved |
| force | u64 | FNV-1 64 of the Gmap faction (勢力) row | confirmed |
| leader | u64 | FNV-1 64 PID of the leader, for roaming lord troops | confirmed |
| spot | u32 | IdHash of the current spot. For troop 0 this is **where the party stands** | confirmed |
| dest | u32 | IdHash of the destination / last target spot | inferred |
| route | u8 n + n × u32 | queued path (always empty) | layout confirmed |
| u8 (+0x50) | u8 | 0 for the party, 5..35 for clans | unknown |
| 2 × u8 | | always 0 | unknown |
| corps | u8 | corps index; 255 = none | inferred |
| u16 (+0x220) | u16 | flags: party 0 / 0x90, clans 0 / 0x200 | unknown |
| PID | u64 | the leader PID again for lord troops, else 0 | confirmed |
| units A | `u8 0, u8 n, n × {u8 36, u8 flag, unit body}` | member units, full UnitPool-format records | confirmed |
| units B | same | second unit list (reinforcements, inferred) | layout confirmed |
| time | u64 GmapTime | always 0 | layout |
| u8 (+0x200) | u8 | 0/2 (2 on lord troops) | unknown |
| offset | 3 × f32 | display offset x, y, z | inferred |
| spot2 | u32 | a third spot ref, always 0 | layout |
| u8 (+0x222) | u8 | 0/2/6 | unknown |
| time | u64 GmapTime | always 0 | layout |

### 2.3 GameGmapData (v12)

Writer `main+0x1387d60` **(confirmed; every real save decodes to the end)**.

| # | Field | Encoding | Meaning | Status |
|---|---|---|---|---|
| 0 | version | u8 12 | | confirmed |
| 1 | const | u8 1 | skipped by the reader | confirmed |
| 2 | miasma manager | u8 n; n × entry; u8 latch; u8 m + m × u64 | Part 3 miasma (瘴気); entry = `u8 1, u64 miasma row key, u8 k + k × u64, u8, u8 j + j × u32 covered spot IdHashes, u16 (remaining HP, inferred), u8 (cycle counter, inferred)`. Empty outside Part 3 | layout confirmed |
| 3 | lure pending | u8 | animal-lure encounter pending | confirmed |
| 4 | camera zoom | f32 | world-map camera zoom, clamped to [0, 1] on load | confirmed |
| 5 | GmapSituation | `u8 12`, u8 lord, u8 chapter, u8 re-init flag, u8 Part-3-init flag, u8 cached `門破壊期間`, u8 cached `Gmap_ダグシオン耐久値`, `u16 n + n × u64` chatter played | lord/chapter the map was last initialised for (a difference re-initialises the world map); the chatter list holds FNV-1 64 of already-played `MID_Gmap_Fix_*` lines | confirmed |
| 6 | spot pins | u8 0, u8 n, n × u64 | world-map spot pins (low 32 bits = spot IdHash); cap from variable `Gmap_スポットピン_最大数`, default 5 | confirmed |
| 7 | BladeResonance (刃鳴り) | u8 1, u8 power, u8 counter, u64 target row | Part 3 blade-resonance gauge | layout confirmed, counter/target inferred |
| 8 | event queue | u8 1, u8 index, u8 n, n × string, u8 | pending world-map event commands (e.g. `会話/MID_.../`, `ウェイト/1/`); non-empty only in checkpoints saved just before a map event | confirmed |
| 9 | trailer | u8 0 | | confirmed |

In the world-map code, "part byte == 3" checks mean **Part 3** (the field is the GameUserData part byte).

---

## 3. HubData and HubSeq

`HubData` (version 21, FNV-1 32 `FC7F1B42`): write `main+0xdd01e0`, read `main+0xdd1d60` **(confirmed)**. Strings
are `u16 (len+1)` + bytes + NUL (or `00 00`), not aligned. Objects are 4-aligned. Offsets in the "field" column are
the runtime `GameHubData` offsets.

### 3.1 Layout (write order)

| # | Field | Encoding | Meaning | Status |
|---|---|---|---|---|
| 0 | version | u8 21 | | confirmed |
| 1 | +16 | string | empty in every save; meaning unknown | layout confirmed |
| 2 | +720..+724 | object `{u8 0; 5 × u8}` | +720 = hub camera distance step 1..3 (default 3); +721..+724 always 0 | confirmed |
| 3 | +40 | u8 | "has saved player position" (1 = use the position struct) | confirmed |
| 4 | position struct | object `{u8 1; string map; string locator; u8 mode (2); string; f32 x, y, z; f32 yaw (radians); u8 camValid; f32 camA; f32 camB}` | player location at save time, e.g. map `Hub_000`, locator `A1` | confirmed |
| 5 | +160 | u8 | an escape/return point is stored | confirmed |
| 6 | escape point | object `{u8 0; f32 x, y, z; f32 angle}` | | confirmed |
| 7 | +208 | string | escape point label, e.g. `Hub_000_A1` | inferred |
| 8 | +232 | u16 n + n strings | reserved enter-event script (name + args), run on the next hub entry and cleared | confirmed |
| 9 | +256 | string | **hub BGM override label**; the literal `null` means silence | confirmed |
| 10 | +280 | string | dispos-ID override label | setter confirmed, name inferred |
| 11 | +304 | u8 | facilities enabled (exit-to-world-map needs 1); always 1 | confirmed |
| 12 | +308 | u8 | unit count allowed in hub/dungeon battles; always 5 | use confirmed |
| 13 | +312 | u64 | 32-bit random value that changes every save (hub random-placement seed, inferred) | inferred |
| 14 | +320 | u32 | **hub day stamp** `year << 20 \| month << 14 \| day << 8` of the last hub refresh | confirmed |
| 15 | +324 | u32 | the same stamp + time-of-day zone 0..3 (morning/day/evening/night); refreshes NPC schedules | confirmed |
| 16..18 | character state lists 0..2 | each object `{u8 0; u16 n; n × u64}` | global hashes of hub characters (handled field enemies, encounter NPCs) | confirmed |
| 19..21 | object state lists 0..2 | each object `{u8 0; u16 n; n × object {u8 0; u64 hash; u8 exists; u32 done}}` (20-byte records) | list 0 = persistent hub objects (chests, walls, switches); list 1 = dungeon pickups; list 2 = respawning destroyables/ore (seen in dungeon command-battle restarts). `exists = 0` destroys the object on spawn; `done = 1` = opened/used | confirmed |
| 22 | +872 | u16 n + n × object `{u8 0; string name; item set}` | named item sets (a person's stored equipment, inferred); item set = `{u8 0; u8 n1; u8 n2; slots}` with slots `'!'` empty, `'A'` + IID, `'a'` + IID + 8 bytes | layout confirmed |
| 23 | +328 | u64 bitfield | bit 0 encounter BGM continues, bit 1 save menu disabled, bit 2 one unit in dungeon, bit 3 inside a dungeon run, bit 4 autosave latch, bit 5 command battle started from the hub/dungeon | bits 0..2 and 5 confirmed, 3..4 inferred names |
| 24 | +336 | u32 len + bytes | **mini-save A** (§3.2) | confirmed |
| 25 | +464 | u32 len + bytes | **mini-save B** | confirmed |
| 26 | +592 | u32 len + bytes | **mini-save C** | confirmed |
| 27 | pending encounters | object `{u8 0; u64; u16 n; n × u64}` | characters awaiting a post-battle move into list 0 or 1 | confirmed |

**The u64 "global hashes"** in the character lists, object lists and mini-saves behave like FNV-1 64 of
digit-suffixed strings, but their construction is unknown. This only matters for synthesising hub state.

### 3.2 The three embedded mini-saves

Each is `u32 len` followed by a **type-1 document** (its own 32-byte chunk header and FNV-1 32 hash) with an
anonymous root object. Pointer words are relative to that document. They are rewritten only when the hub is live at
save time; mini-save C has its own rewrite path, so it can appear in world-map and battle saves where A and B are
empty **(confirmed)**.

**A: party and dungeon state.**

```
root{ u8 0,
  obj{ u8 1,
       u16 n, n x obj{u8 0, u64 globalHash, f32 x, f32 y, f32 z, f32 yaw},          field characters of the current dungeon map
       obj{ u8 1, u16 capacity, u16 count,
            capacity x obj{u8 1, f32 x, f32 y, f32 z, f32 yaw, f32 distance, u8} } },  the player's movement trail
  obj{ u8 0, u16 n, n x obj{u8 0, u64 hash, 3 x vec3, string, u8, string, string} }   hub ObjectManager records (never populated)
}
```

The trail capacity is 200 in hubs and 5000 in dungeons (which is why DUNGEON restart payloads are ~140 KB larger).

**B: NPC arrangement.**

```
root{ u8 0, obj{ u8 0, u16 capacity = 196, u16 count,
                 196 x obj{ u8 1, string PID, f32 x, f32 y, f32 z, f32 yaw, string locator, u64 hash } } }
```

Slots beyond `count` are empty.

**C: random gather points.**

```
root{ u8 0, obj{ u8 0, u16 n,
                 n x obj{ u8 0, string area, string lotteryTag, u16 m,
                          m x obj{ u8 0, u64 IID, u64 pointHash, u8 collected } } } }
```

One entry per active Hub.bin random-placement row: `area` (e.g. `Hub_000_A1`), lottery tag (e.g.
`抽選_1部ダグシオン_淡水魚`), and per placement the rolled item, the locator hash and `collected` = 1 once picked.

### 3.3 HubSeq

`HubSeq` (FNV-1 32 `78AE2B2F`) is an **empty marker**: its serializer's read and write functions do nothing, so the
block is 8 bytes `[ptr][00][pad 3]` **(confirmed)**. It only records that the hub sequence was running at save time.

---

## 4. SideStoryData (paralogues)

Writer `main+0x12da040`, reader `main+0x12da4c0` **(confirmed)**.

```
[ptr][u8 version 4][u32 n = 9]
n x entry          {u64 FNV1-64("SSID_<name>外伝"), u8 state, u8 b, u8 hasExtra} (+7 bytes if hasExtra)
u32 m              list 2: m x {u16 y, u8 mo, u8 d, u8 h, u8 mi, u8 s, u64 SSID}   (15 bytes each; always empty)
u32 k              cleared-ever list: k x u64 SSID
```

- The 9 entries are the SideStory.bin rows, in table order (e.g. `SSID_カイ外伝` = `0xA26AEDB9FDCA69D0`).
- `extra` = `u16 year, u8 month, day, hour, minute, second`: the scheduled event time, set only for rows whose
  `時刻` ≠ −1.
- `b` is a 0/1 flag; every real clear has b = 1 (inferred: "battle won / objective done").

| State | Meaning | Status |
|---|---|---|
| 0 | not offered | confirmed |
| 1 | offered | confirmed |
| 2 | accepted (sets the row's `X外伝_受注済` flag) | confirmed |
| 3 | done, reward pending | confirmed |
| 4 | completed, reward received (terminal) | confirmed |
| 5 | missed / expired (terminal) | confirmed |

**Route zeroing (confirmed).** On every load and at every map/hub sequence start, each entry whose route bit
`(発生するルート >> (lord − 1)) AND 1` is 0 for the current lord has its state, b and hasExtra cleared. The
**cleared-ever list is never touched**: it is what persists a paralogue clear across routes. Eshmel (lord 9) has no
bit, so no paralogue is on his route. The route bits come from the `発生するルート` column of SideStory.bin (bit 0 =
Cai's route).

Related variables of a real clear: 状況 `X外伝_クリア`; 限定 `X外伝_受注済` and `X外伝_受注後の準備開始`; 共有
`共有_特殊_X外伝_クリア`; 実績 `実績_外伝クリア数`.

---

## 5. The save card and the header masks

**The card (confirmed).** The save dialog (`main+0x16db9e0`) is used for Save, Suspend and Continue.

- Save/Suspend: lord, part and chapter come from the running game; the date from the in-game clock; the playtime
  from the playtime counter (clamped to 9999:59:59).
- Continue (`main+0x16db6a0`): reads only the `Auto` header and aborts if status bit 30 is set or the header save ID
  doesn't match.
- The pixel-art character icons are **per-character route-progress icons**, read from the variables of the loaded
  save for characters 1..8 (lords; Eshmel has none). First match wins:
  1. `2部正式クリア_X篇` ≠ 0 → Part 2 cleared icon
  2. `2部クリア_X篇` ≥ 1 → Part 2 cleared (skipped) icon
  3. 共有 `共有_特殊_第三部で限定参戦可能_X` ≠ 0 → bright Part 1 icon
  4. `1部クリア_X篇` ≥ 1 → greyed Part 1 icon
- `3部クリア` adds an Eshmel icon. Icons pack left.
- Rule 3's variable is a latch the game sets itself in the Obelisk Chamber when that character's paralogue is cleared
  and their Part 2 is not.

**The masks (confirmed).** File 0x63 = bit L set when `2部クリア_<L>篇` ≠ 0; file 0x65 = bit L set when the lord's
paralogue row flag `X外伝_クリア` ≠ 0 (only lords have bits; non-lord paralogues don't appear). Both are recomputed on
every save and never read by the card.
