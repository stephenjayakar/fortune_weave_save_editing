# Editing guide

How to change things in a Fortune's Weave save safely. Read [doc 01](01-container.md) first: every recipe here
assumes you can unpack the container, walk the object tree with an exact schema, and repack it.

Byte offsets refer to the block docs: unit fields to [doc 02](02-units-convoy-mounts.md), GameUserData fields
(`body+N`) to [doc 03](03-game-user-data-and-variables.md). Recipes marked *untested* follow from the format but were
not checked in game.

---

## 1. General procedure

1. **Close the game** before touching the save folder. The game writes into temporary work folders and then commits,
   so editing while it runs can be overwritten or lost.
2. **Keep a copy of the original file outside the save folder.** On the PC port every file inside the save folder
   counts against the save-data size quota, so don't leave stray copies there.
3. **Unpack:** check the magic at 0x000 and 0x104, read `N` at 0x100, decompress the data document (type 3 = zstd,
   type 1 = raw).
4. **Edit the payload** with an exact, schema-driven reader (never a heuristic walker; see
   [pitfalls §7](known-tool-pitfalls.md#7-heuristic-tree-walkers)).
5. **Size-neutral edits** (overwriting bytes in place) need no further structural work. **Anything that adds or
   removes bytes needs a splice** (§2).
6. **Rehash and repack** (§3).
7. **Verify offline** (§4) before loading in game.
8. **Decide about the restart copies explicitly** (§5).

---

## 2. Splicing: changing the size of anything

Every object stores its end (or member-table) position as an absolute document position, so inserting or removing
bytes at position P shifts everything after it. A correct splice:

1. Inserts or removes the bytes at P (P is a payload offset; document position = P + 32).
2. Adds the size delta to **every pointer word and every member-table position** in the same document whose value
   is greater than the document position of P. That includes the end pointers of all enclosing objects and the root
   member table at the end of the payload.
3. Keeps every object 4-aligned. The simplest way is to keep the delta a multiple of 4 by adjusting the padding at
   the end of the object you grew, re-padding it **minimally** (a new object starts at the next multiple of 4 after
   the previous content).
4. If P is inside an **embedded document** (a restart payload or a HubData mini-save), shift pointers only inside
   that embedded document, then treat the embedded document's growth as an insertion in its parent: update its
   chunk-header size, its `u32 n` length prefix (restart `Data`/`Header` objects, HubData stream lengths), and shift
   the parent's pointers.
5. Rehashes everything that contains P (§3).

You must know which words are pointers. Only an exact parser that follows the schema knows that; data such as support
records or blessing hashes can look like valid pointers.

---

## 3. Rehashing and repacking

Refresh chunk hashes **innermost first**:

1. HubData mini-save documents (in the top-level payload and inside each restart payload)
2. restart `Header` documents and restart `Data` documents
3. the payload hash at file 0x10C (FNV-1 32 of the **uncompressed** payload; 0 → `0xFFFFFFFF`)
4. the file header hash at file 0x008 (FNV-1 32 of `file[0x20:0x100]`)

Then compress the payload (or store it raw with type 1; both load), write the data-document size at 0x108 and
`N = data document length` at 0x100. The game appears not to verify hashes on load, but refresh them anyway.

If you change mode or difficulty, copy the payload's summary values into the file header too (§16).

---

## 4. Verifying an edit

- Re-parse the whole new file with an exact reader: every object must end exactly where its pointer says, in the
  top-level payload and in every embedded document.
- Diff the decoded structure against the original: only the fields you meant to change (and shifted pointers and
  hashes) should differ.
- Run the edit twice on its own output: a correct edit is idempotent.
- Check `GameMiracle` and `GameReliance` after any splice; a naive splice can corrupt a hash inside them
  ([pitfalls §7](known-tool-pitfalls.md#7-heuristic-tree-walkers)).

---

## 5. Restart copies

Each filled restart slot (CHAPTER, BACK, MAP, DUNGEON, CMDBTL) holds a **complete separate payload**: units, mounts,
convoy, variables, supports, blessings, GameRecord, GameUserData scalars. They do **not** contain `GameConverge`.

- An edit that should survive "Restart chapter" or "Restart battle" must be repeated in each filled slot.
- If you don't edit them, using a restart rolls the edit back. That is safe, just surprising.
- A slot is filled only when its `Data` n > 0.

---

## 6. Stats, level, EXP, HP

Size-neutral.

- **Stats:** unit +37 = 11 × s16 **gains** (HP, Str, Mag, Dex, Spd, Lck, Def, Res, Cha, Sight, Mov). Shown stat = gain
  + Person base + Person modifier + class base + the difficulty set chosen by `grade`. To reach a target shown value,
  subtract the table values. Keep Sight/Mov gains at 0 (Mov normally comes from the class).
- **Growth remainders** (+26, 11 bytes): never zero them. For a new unit use `(3 × Person growth%) mod 100` per
  stat, Sight/Mov 0.
- **Level** +59 (u8, 1..99); **EXP** +60 (u32, 100 per level; clamped on load).
- **Current HP is a u8 at +64.** Write exactly one byte. +65 (pending HP), +66 (HP stock) and +67 (max HP stock) are
  separate fields; leave them alone.
- The same person may also appear in restart payloads and, as a different record, in RouteInfo.

---

## 7. Class

- Unit +14 = current JID IdHash.
- The class list in the UnitJob object holds `{u32 JID, u16 value}`: bits 0..9 mastery EXP (≤ the class's
  `成長経験`), 0x400 = reached by class change. The real shape is: the starting class without 0x400; after a change,
  both the old and the new class with 0x400.
- Add the class's master skill to the skill lists as a real class change would: file list 5 (all learned) and, if
  equipped, file list 1 (equipped passives).
- Weapon ranks are in the same object (`{u8 index, u16 exp}`, non-zero ranks only, thresholds 0..2560).
- Rider classes come with a mount (§10).
- **Don't promote past a story class exam.** A chapter whose story step is the exam can soft-lock if the unit is
  already promoted.
- Changing the number of list entries grows the UnitJob object: splice.

---

## 8. Unit inventory

- Item list object: `u8 0 | u8 nCarried (≤ 6) | u8 nStock (≤ 24) | items`.
- Item = `'A' + u32 IID` (5 bytes, default state) or `'a' + u32 IID + 8 bytes` (13 bytes: row index, **uses spent**,
  flags 0x01 equipped / 0x10 locked / 0x20 broken, count bits).
- Forges are separate IIDs (`＋１`..`＋５`). The equipped flag needs the `'a'` form.
- Adding items or switching to the `'a'` form grows the object: splice.

---

## 9. Convoy and food

- `Transporter/TransporterData` = `[ptr][u8 1][u16 1329]` + slots `u16 qty; if qty: UnitItem`.
- **Never append or remove slots.** The count is always 999 regular + 330 stack slots once initialised. A wrong slot
  count corrupts the convoy.
- Regular items: qty 1, any slot 0..998 (holes allowed).
- Stack items (forge materials, gold bullion, all food) go in their **fixed slot**: index 999 + the item's position
  among the stack rows of Item.bin (kind 14, usage ∈ {13, 14, 18, 19, 20, 22}, table order). Qty cap 999.
- Food = usage 20 (`素材` 0 meat, 1 fish, 2 vegetables); skip `IID_試供餌` / `IID_試供種`.
- Don't fill an `n = 0` convoy unless you write all 1329 slots.
- An empty slot is 2 bytes and a filled one 7 or 15, so filling slots grows the object: splice.
- In a War Arc save the Part-1 convoy is parked in `GameUserData/Storage`; edit the one you mean.

---

## 10. Mounts

Record: [doc 02 §2.7](02-units-convoy-mounts.md#27-unitride-the-mount-record).

**Adding a mount to the stable:**

1. Write a UnitRide v6 record into an empty `00` slot of `RidePool/RideData` (the list always has 200 slots): RID
   key, owner 0, `w4 = serial | lord << 24` with `serial` = the highest existing serial + 1 and `lord` = the current
   route, date (decimal YYYYMMDD of the in-game date), no name (`00 00`), up to 5 bonds, a seed (it picks the ability
   and favourite food), b2 = 0.
2. **Raise GameUserData body+39 (`RideSerial`) to at least the highest serial now in use.** Otherwise the game's next
   mount gets a duplicate serial.
3. The record replaces a 1-byte empty slot, so this grows `RideData`: splice.

**Assigning a mount to a unit:** set `owner` = the unit's number (+0x828) in RideData **and** write the
byte-identical record into the unit's mount slot (right after the barrier SkillKey). Unassigning: owner 0 in
RideData and `00` in the unit. Both directions change the unit's size: splice.

Bonds: `{u8 Person serial, u16 order << 12 | points}`; levels at 0/20/50/90/140 points. The mount's ability unlocks
at bond level 3 with the current rider, its "+" form at level 5.

Restart payloads keep their own RidePool and their own `RideSerial`; raise each copy's counter to at least its own
highest serial.

---

## 11. Supports

Size-neutral.

- `GameReliance` has all 2628 pairs `{u32 FNV1-32("RID_A_RID_B"), u16 points, u16 count, u32 stamp}`, keyed by
  Reliance.bin row keys in row order (not PIDs). **Never append records.**
- Points thresholds 0/400/1400/2400/3200 = levels none, C, B, A, S. The game caps at 3200 only when both persons have
  `加入_`/`面識_`/`開示_` set; otherwise one short of the next level. Follow the same rule for unmet pairs.
- To let the hub offer the conversations, leave 中継 `支援会話_<A>_<B>_<rank>` unset. Setting it marks the
  conversation as watched.
- Restart payloads have their own list.

---

## 12. Blessings (GameMiracle)

Size-neutral.

- `u8 2 | u16 100 | u16 0 | u8 7 | 7 × {u64 FNV1-64("MIR_<god>"), u8 flags, u16 exp}`, order フォトナ, マーズ, クレール,
  カーラ, ジュラ, スミルノス, アウロラ.
- flags bit 0 = equipped (at most 4 including Fortuna, whose flag is always 1).
- The level comes from exp against the blessing's thresholds in Skill.bin; set exp to the third threshold for Lv 3
  (e.g. Aurora 100, Credna 80). Exp is clamped on load.
- Edit each restart copy too. `CommonInfo/GameMiracle` is the cross-route maximum; leaving it lower is harmless (it
  is merged by max).

---

## 13. Boons of Salvation

- 共有 `因果報酬_レベル_<key>` = tiers bought; `因果報酬_状態_<key>` = 1 means switched OFF (absent or 0 = ON). Karma
  Shards = GameUserData body+15 (u32).
- Levels are not re-checked on load. In the game, buying a tier requires `1部クリア` ≥ that tier's clear count.
- **`<key>` must be the game's internal short key**, from a table in the executable (`main+0x1ff71c0`) indexed by the
  Reward row's `効果` id. It is **not** always Reward.bin's key column: 14 of the 31 keys differ, and names built from
  the Reward.bin keys are unregistered and **dropped on load**.

The 14 keys that differ (all others are identical to the Reward.bin key):

| 効果 | Use this key | Not this (Reward.bin) |
|---|---|---|
| 7 | `技能経験格` | `技能経験格闘` |
| 8 | `技能経験黒` | `技能経験黒魔法` |
| 9 | `技能経験白` | `技能経験白魔法` |
| 10 | `技能経験歩` | `技能経験歩兵` |
| 11 | `技能経験騎` | `技能経験騎兵` |
| 12 | `技能経験重` | `技能経験重装` |
| 13 | `技能経験飛` | `技能経験飛行` |
| 14 | `技能経験指` | `技能経験指揮` |
| 26 | `加護_マーズ` | `加護消費_マーズ` |
| 27 | `加護_クレール` | `加護消費_クレール` |
| 28 | `加護_カーラ` | `加護消費_カーラ` |
| 29 | `加護_ジュラ` | `加護消費_ジュラ` |
| 30 | `加護_スミルノス` | `加護消費_スミルノス` |
| 31 | `加護_アウロラ` | `加護消費_アウロラ` |

Example: the Black Magic EXP boon at tier 2 is the 共有 entry `IdHash("因果報酬_レベル_技能経験黒") → 2`.

---

## 14. Flags (variables)

- Find the variable's category first ([doc 03 §2](03-game-user-data-and-variables.md#2-variable-categories)). The
  same name in the wrong category map is a different variable.
- **Only registered names survive a load.** Use names that exist in real saves or come from a family the game
  registers. Anything else is silently dropped.
- Write values canonically: one byte if 0 ≤ v < 16, else `0xFF` + i32.
- Kind-0 maps store only non-initial values: setting a value equal to its initial value is harmless but
  non-canonical; deleting the entry resets the variable.
- Adding entries grows the map: splice. Restart payloads have their own `Variable` struct.
- Route-scoped categories (限定, 既読) are reset when a route is chosen; put cross-route state in 共有.

---

## 15. Paralogues

- **On-route paralogue** (its route bit `lord − 1` is set in SideStory.bin `発生するルート`): set its SideStoryData
  entry to state 4, b 1. Real clears of rows with `時刻` ≠ −1 also carry the 7-byte scheduled date (harmless if
  missing, because state 4 is terminal).
- **Every cleared paralogue, on any route:** append its FNV-1 64 SSID to the **cleared-ever list**. This grows the
  block: splice. Off-route entries are zeroed by the game on every load, so the cleared-ever list is the only thing
  that persists a clear across routes.
- Variables of a real clear: 状況 `X外伝_クリア`; 限定 `X外伝_受注済` and `X外伝_受注後の準備開始`; 共有
  `共有_特殊_X外伝_クリア`; 実績 `実績_外伝クリア数` (= the count).
- Save-card icon: 共有 `共有_特殊_第三部で限定参戦可能_X` (the latch the game itself sets in the Obelisk Chamber).
- The header paralogue mask is recomputed on save; don't bother editing it.

---

## 16. Difficulty and mode

Size-neutral.

| Byte | Meaning |
|---|---|
| GUD body+1, payload[14], file 0x26 | **GameMode: 0 Casual, 1 Classic** |
| GUD body+2, payload[15], file 0x27 | **Difficulty: 0 Normal, 1 Hard, 2 Lunatic (Maddening)** |
| GUD body+49 | difficulty chosen at new game |
| unit `grade` (each unit) | difficulty-set index |

A consistent difficulty change sets: body+2, body+49, payload[15], file 0x27, the same bytes in every restart summary
and restart GameUserData, **and every unit's `grade`** (main pool and restart copies). `grade` selects the
`難易度差分` set of each Person/Job row (initial skills, barrier items, potential skills); check whether any of your
units' persons or classes has such a row, because changing `grade` changes those.

- Body+49 must not be below the played difficulty: a route's best difficulty is recorded as the *lower* of body+2
  and body+49.
- The game itself only ever lowers body+2 (Obelisk option) and leaves body+49 alone.
- A mode change sets body+1, payload[14], file 0x26 and the restart copies.

---

## 17. Story position

- **Don't edit lord/part/chapter** (body+3..+5) to move through the story. Story position is the timeline cursor key,
  the call stack, the in-game date, the SectionTime index, the scene kind and hundreds of variables together
  ([doc 03 §10](03-game-user-data-and-variables.md#10-the-timeline-key-story-position)).
- To reach a chapter, start from a **real checkpoint** at that point (`Chapter_<c>_<p>_<n>` or `Part_<c>_2`) and
  transplant the player data onto it: units, convoy, mounts, supports, blessings, cross-route variables (共有, 同期,
  回想, 実績, チュートリアル), `GameConverge`, the SideStoryData cleared-ever list, settings. Keep the checkpoint's own
  限定/既読/状況 variables, GameSchedule, timeline key and stack, SectionTime index, HubData, TaskData.
- The save ID (body+27, file 0x67) must match between the header and GameUserData, and must match the rest of the save
  data, or Continue rejects the file.
- There is no debug chapter select. The timeline key is computable, so small in-chapter cursor moves are conceivable
  (key = FNV-1 64 of the target row's key string, date = that row's date, matching lord/part/chapter, empty stack),
  but this is *untested*. A key that doesn't resolve loads as a null cursor.

---

## 18. Routes (RouteInfo)

Record layout: [doc 04 §1.2](04-routes-world-hub.md#12-the-route-record).

- Best difficulty 0..2; 3 = never cleared.
- A route with only Part 1 cleared has **no units**; units are merged only at the War Arc clear. Units use the full
  unit record format.
- The route's GameRecord: `order_p1`/`order_p2` = global clear-sequence index across all routes and both parts
  (0-based, `0xFF` = never), plus one chapter record per chapter cleared.
- Variables (共有): `1部クリア` (count of Part-1 clears; boon tiers test it), `1部クリア_<lord>篇`,
  `2部クリア_<lord>篇`, `2部正式クリア_<lord>篇`, `許可_<lord>篇`. "View Clear Data" needs `2部クリア_<lord>篇`.
- A real route end also captures CommonInfo (spot states, the 中継 snapshot, the blessing maximum, last lord/part)
  and sets the first lord. A synthetic route record with an empty CommonInfo snapshot is harmless (the next route
  start simply has nothing to merge) but not faithful.
- `Route_<code>` slot files exist only for routes parked part-way.

---

## 19. Map reveal (world map)

Size-neutral.

- For each `GmapSpotMgr` entry: read the `u32` key **before** aligning, then align, then the spot object.
- Set the state byte (spot object +4, runtime +176) to **3**. Leaving `saved` and `flags` at 0 is valid: (3,0,0) is
  common in real saves. Flags bit 0 = arrived; bit 1 = checkpoint passable (kind 4).
- Skip spots whose Gmap.bin attribute has 0x80 (event-only spots), maze nodes, underworld spots, Part 3 spots and
  story-locked spots.
- The game's own reveal also sets a per-spot variable; a byte edit doesn't (no problem has been observed).
- Region maps: Gmap.bin groups map to `IID_<region>地図` items. Hub fast travel = 中継 `拠点_FT_Hub_...` variables;
  person fast travel derives from `面識_<PID stem>`.
- Moving the party: `GmapTroopMgr` troop 0's spot (and probably dest and its flags). *Untested.*

---

## 20. Protagonist name

**Rename size-neutrally unless you splice.** The name is stored as `u16 n | 07 | UTF-8 | 00` (n counts the 07 and
the NUL). Rewrite it in **every** place:

- `GameConverge/CommonInfo` profile
- every GameRecord MVP name tag: GameUserData's GameRecord, each RouteInfo GameRecord, and each restart payload's
  GameRecord
- the protagonist unit's UnitEdit, wherever that unit is in a pool

If the length changes, splice each enclosing object and **re-pad it minimally to 4**. Never keep the old padding and
append more: an object with surplus padding is not what the game writes, and a rename built that way was followed by
a hang after Continue (suspected cause). The name is not in the file header.

The account nickname shown by the PC port is a port setting, not part of the save.

---

## 21. Playtime

Move these together:

- header playtime (file 0x54 and the payload `Header` copy at payload 0x3C)
- the SequenceTime "entered" stamp (member +13) **and** a per-scene total, so that the sum of the 17 totals still
  equals the stamp
- the SectionTime stamp (member +9) **and** section slots, so that the sum of the 560 slots still equals the stamp

Optional:

- **Timer A** (the first u32 after Storage) = playtime since the current part/route started. Move it only if the
  added time is meant to belong to the current part. Timer B stays 0.
- RouteInfo clear times of finished routes should stay below the total.
- Don't jump the 実績 counter `実績_プレイ時間`; leave deeds to the game rather than making play-time deeds claimable at
  once.

---

## 22. Gold, renown, Karma Shards

Size-neutral: GUD body+6 (u32 gold), body+10 (u32 renown), body+15 (u32 Karma Shards). Restart GameUserData copies
are separate.

---

## 23. Safe to leave stale

The game recomputes or ignores these:

- `CheckPerson` (skipped on load)
- the header masks at 0x63/0x65 and all other summary fields (rebuilt on every save)
- support level bytes (from points), weapon-rank letters (from exp)
- item row indexes and battalion row indexes (from the hashes)
- spot `saved` when loading state 2
- mount bond order (rewritten)
- chunk hashes (apparently not checked; refresh them anyway)
