# 03. GameUserData and variables

This document covers the `GameUserData` block (version 23): its 100-byte scalar body, the raw fields written between
its members, and every member except `GameConverge` (which is in [doc 04](04-routes-world-hub.md#1-gameconverge)).
It also covers the top-level `GameSchedule` block and the timeline key, because together they define the story
position.

Writer `main+0x71bbe0`, reader `main+0x71d100`. `G` = the GameUserData runtime object; field names marked ★ are the
game's own names (from its named-field export at `main+0x71e600`). Code addresses are `main+0x...` for update 131072.

---

## 1. GameUserData body

Offsets are from the start of the block body (block start + 4, i.e. right after the pointer word). The body is 100
bytes in every real save.

| Body off | Type | Field | Meaning | Status |
|---|---|---|---|---|
| 0 | u8 | version | 23 | confirmed |
| 1 | u8 | `G+0x0C` ★GameMode | **0 Casual, 1 Classic** | confirmed |
| 2 | u8 | `G+0x0D` ★Difficulty | **0 Normal, 1 Hard, 2 Lunatic (Maddening)** | confirmed |
| 3 | u8 | `G+0x0F` | lord / route: 1 Cai, 2 Dietrich, 3 Theodora, 4 Leda, 5 Beltran, 6 Talimun, 7 Orchel, 8 Anatolia, 9 Eshmel | confirmed |
| 4 | u8 | `G+0x3C` | part: 0 prologue, 1, 2, 3 | confirmed |
| 5 | u8 | `G+0x40` | chapter | confirmed |
| 6 | u32 | `G+0x48` | gold | confirmed |
| 10 | u32 | `G+0x4C` | renown | confirmed |
| 14 | u8 | `G+0x44` | army level, snapshotted at chapter start (equals the chapter's GameRecord byte 3) | value confirmed, meaning inferred |
| 15 | u32 | `G+0x50` ★Causation | **Karma Shards** | confirmed |
| 19 | u32 | `G+0x54` | battalion reward points pending: a battalion's Legion.bin `褒賞` is added when it is released; every 100 points become one `IID_戦術書` (Tactics Book), the remainder is kept | mechanism confirmed |
| 23 | u32 | `G+0x58` | lifetime total of the same points | confirmed |
| 27 | u32 | `G+0x08` | **save ID**: must equal the file header's save ID (file 0x67), or Continue rejects the file | confirmed |
| 31 | u32 | `G+0xA90` | current story-line id (= file 0x6B). Set to `++Shareline` at a route start and when play resumes from a Chapter checkpoint | confirmed |
| 35 | u32 | `G+0xA94` ★Shareline | number of story lines started in this save data | confirmed |
| 39 | u32 | `G+0xA98` ★RideSerial | **last mount serial issued**; must stay ≥ the highest mount serial ([doc 02 §4.1](02-units-convoy-mounts.md#41-mount-ids-and-the-serial-counter)) | confirmed |
| 43 | u32 | `G+0xA9C` ★SaveCount | times saved, autosaves included; saturating | confirmed |
| 47 | u16 | `G+0xAA0` ★MakeCount | +1 each time the protagonist creator finishes | confirmed |
| 49 | u8 | `G+0x0E` | **difficulty chosen at new game**. Kept because lowering the difficulty at the Obelisk changes only `G+0x0D` | confirmed |
| 50 | u64 | `G+0x10` | current battlefield: FNV-1 64 of `SCxx_yy`; 0 outside battle | confirmed |
| 58 | u64 | `G+0x38` | FNV-1 64 of the current scene kind (table below) | confirmed |
| 66 | u64 | `G+0x39` | FNV-1 64 of the previous/sub scene kind (NONE in real saves) | confirmed |
| 74 | u8 n + n × u64 | | scene-kind stack, FNV-1 64 per entry; n = 0 in real saves | confirmed |
| (pad to 4) | | | | |
| 76 | object | `G+0x18` | anonymous `{u8 0, u64 key}` (16 bytes with padding) = the **timeline cursor** (§10). The same key is in file 0x49 | confirmed |
| 92 | u64 | `G+0x20` | current SideStory (paralogue) row; 0 unless saved inside a paralogue | confirmed |

The offsets from 76 on assume an empty scene stack (true in every real save).

**Scene kinds** (index = `G+0x38`; the save stores FNV-1 64 of the name): NONE, TITLE, MAP, UNUSED, GMAP, HUB, DUNGEON,
CMDBTL, TIMELINE, DEMO, MOVIE, CHAPTER, AUTOSAVE, MAP_COMPLETE, CMDBTL_COMPLETE, MAKE, MAKE_COMPLETE.

### 1.1 Mode and difficulty

Settled from code **(confirmed)**:

- The new-game commit (`main+0xa1c3d8`) stores the GameMode menu result to `G+0x0C` and the Difficulty menu result
  to both `G+0x0D` and `G+0x0E`.
- The telemetry export (`main+0x1337080`) reports `GameMode` = CASUAL/CLASSIC from `G+0x0C` and `Difficulty` =
  NORMAL/HARD/LUNATIC from `G+0x0D`.
- The save dialog shows `MID_Mode_Casual/Classic` from `G+0x0C` and `MID_Difficulty_Normal/Hard/Lunatic` from `G+0x0D`.
- The Obelisk "lower difficulty" option rewrites only `G+0x0D`.
- Map unit placement (`main+0x7448cc`) sets each unit's `grade` = `G+0x0D`.

Copies of these bytes: file 0x26 (mode) / 0x27 (difficulty), payload[14] / payload[15], GUD body+1 / body+2, every
restart summary and restart GameUserData, body+49 (start difficulty), and each unit's `grade`. See
[editing guide §16](editing-guide.md#16-difficulty-and-mode).

### 1.2 Members and interleaved fields

Objects in file order (= write order):

```
body (100 bytes)
Variable
GameReliance
GameSituation            version byte only
GameConverge             cross-route store; omitted in restart payloads
GameRecord
  u32                    G+0x5C flags (= file header status flags bits 0..10)
GameMiracle
TaskData
  u8 n + n x {u8 0, u64 timeline key}   timeline call stack
CheckPerson
SequenceTime
SectionTime
Storage
  u32 timer A, u32 timer B
member table (11 entries; 10 in restart payloads)
```

| Member | FNV-1 32 | Content | Section |
|---|---|---|---|
| `Variable` | C68947E1 | 13 category maps | §3 |
| `GameReliance` | FE992C20 | 2628 support pairs | §4 |
| `GameSituation` | 3F7E8625 | just `u8 3`; nothing else is written or read | [05](05-misc-blocks.md#7-gamesituation) |
| `GameConverge` | A5D56C94 | routes, cross-route data | [04](04-routes-world-hub.md#1-gameconverge) |
| `GameRecord` | 36BB2FD4 | per-chapter clear records | §8 |
| `GameMiracle` | 426D8EFC | god blessings | §7 |
| `TaskData` | 6F7832F2 | current hub objective | §9.1 |
| `CheckPerson` | 4126266C | person → serial number table; ignored on load | §9.2 |
| `SequenceTime` | E798A625 | playtime per scene kind | §6.1 |
| `SectionTime` | B9F5F339 | playtime per story section | §6.2 |
| `Storage` | B0B6A91C | parked convoy/battalions/mounts | §9.3 |

**Interleaved fields (confirmed).**

| Position | Type | Meaning |
|---|---|---|
| after GameRecord | u32 | `G+0x5C` flags: bit 0 battle start pending, bit 1 battle entered from the world map, bit 2 map completed |
| after TaskData | u8 n + n × `{u8 0, u64 key}` | the timeline **call stack**: the 呼ぶ (call) rows that are waiting for a return (§10). Non-empty only during the Part 1 → Part 2 handover |
| after Storage | u32 + u32 | two stopwatches packed as `u16 hours, u8 minutes, u8 seconds`. **Timer A = playtime since the current part/route started** (0:00:00 at the first chapter of a part; likely the source of RouteInfo clear times). Timer B = 0 in every real save |

---

## 2. Variable categories

Writer `main+0x82f0c0`, reader `main+0x82f1d0` **(confirmed)**.

`Variable` = struct `[ptr | n << 24][u8 version 2][pad]` + member table, one member per **saved** category. Member
hash = FNV-1 32 of the category name. The game has 19 categories:

| # | Category | Saved | Member hash | Content |
|---|---|---|---|---|
| 0 | システム | no | | |
| 1 | ローカル | yes (kind 1) | 8186B481 | battle-map script variables; only in battle saves (e.g. `勝利`, `スコア表示`) |
| 2 | イベント | yes (kind 1) | E5834A3C | hub/dungeon script event variables, self-named strings |
| 3 | 区間 | no | | |
| 4 | 限定 | yes | 972FA7EE | route-local flags (`禁止_`, `機能開放_`, `面識_`, `加入_`, ...), shop stock, gift dates |
| 5 | 日時 | yes | 0802E120 | date-triggered variables |
| 6 | 同期 | yes | 64ABA34F | Variable.bin rows with `変化` = 2 |
| 7 | 中継 | yes | F56D6832 | support conversations watched, hub fast-travel points |
| 8 | 既読 | yes | B631F2C1 | message labels already read (`MID_*`) |
| 9 | 状況 | yes | 4AD575A2 | story/appearance progress, class and skill permissions, paralogue clear flags |
| 10 | 共有 | yes | 2BED8663 | cross-route: clears, boons, Obelisk events, paralogue clears, save-card latches |
| 11 | 回想 | yes | 06C136C7 | recollections |
| 12 | 環境設定 | yes | 58A0BE00 | options (non-default only) |
| 13 | 実績 | yes | 25207C4C | deed/record counters, deed claim states, person bit sets |
| 14 | チュートリアル | yes | E9BC5B65 | `TUTO_*` |
| 15 | ゲーム | no | | |
| 16 | MS限定 | no | | |
| 17 | デバッグ | no | | |
| 18 | 未使用 | no | | |

The saved set is categories 1, 2 and 4..14 (mask `0x7FF6`).

### 2.1 Category map layout

```
[ptr][u8 kind][u16 n] then n entries, padded to 4
kind 0 entry:  u32 IdHash(name) | value
kind 1 entry:  u64 FNV1-64(name) | value | u16 (len + 1) | UTF-8 name + NUL     (ローカル and イベント)
```

- **value: one byte if 0 ≤ v < 16, else `0xFF` + i32** **(confirmed, every entry in real saves follows this)**. The
  reader accepts any byte ≠ `0xFF` as a value, so 16..254 written as one byte loads fine but is non-canonical; the
  game rewrites it in the wide form on the next save.
- **Kind 0 maps store only values that differ from the initial value** (Variable.bin `初期値`, or 0). An absent entry
  means "at its initial value". Deleting an entry resets the variable.
- **Kind 1 maps store every variable** of the category, each with its own name string.

### 2.2 What survives a load

**(confirmed, `main+0x83caf4..0x83cc8c`)**

- **Kind 0:** each u32 is looked up in the registry of known names. **Hashes the game hasn't registered are silently
  dropped.** Registered names are set with the category taken from the save.
- **Kind 1:** a missing name is created on load. Arbitrary string-named event variables survive.
- Rule for editors: a kind-0 name must come from a family the game registers. Anything a real save already contains
  is registered in this game version.

### 2.3 Which category a variable belongs to

- Variable.bin rows: the category is the sheet the row sits in, with two exceptions: rows with `変化` = 2 go to 同期,
  and dated rows go to 日時.
- Non-table families are registered dynamically by the game: 既読 `MID_*`, チュートリアル `TUTO_*`, 中継
  `支援会話_<A>_<B>_<rank>` and hub fast-travel points, 実績 counters and claim states, 回想 recollection data, and
  printf-built families in 限定/状況/共有 such as `面識_/加入_/開示_<PID stem>`.
- Choosing a route resets 限定 and 既読 (route-scoped). 共有, 同期, 回想, 実績 and チュートリアル carry over.

---

## 3. Variable families and value meanings

### 3.1 Common families

| Prefix | Meaning |
|---|---|
| `禁止_` | locked while 1 |
| `機能開放_` | feature unlocked |
| `許可_`, `許可_<lord>篇` | permission; route unlocked |
| `兵種_許可_` / `兵種_表示_` | class tier allowed / shown |
| `使用許可_` | skill use allowed |
| `行動範囲制限_` | movement area limit |
| `進行_` | story or appearance progress |
| `面識_` / `加入_` / `開示_` | person met / joined / profile disclosed |
| `ファストトラベル_` | person fast travel |
| `支援会話_` (中継) | support conversation watched |
| `拠点_FT_` (中継) | hub fast-travel point unlocked |
| `因果報酬_レベル_` / `因果報酬_状態_` (共有) | Boons of Salvation: tier bought / switched off |
| `実績_` / `記録_` (実績) | deed counter / play-record counter |
| `0部/1部/2部(正式)クリア[_<lord>篇]` (共有) | part clears |
| `方尖塔の間_` | Obelisk Chamber events |
| `...外伝_` | paralogue |
| `GMAP_` / `Gmap_` | world map |

### 3.2 Clear variables (共有, confirmed)

Set by timeline command 35: `0部クリア`, `1部クリア` (a **count** of Part 1 clears; boon tiers test it), `2部クリア`,
`2部正式クリア`, `1部クリア_<lord>篇`, `2部クリア_<lord>篇`, `2部正式クリア_<lord>篇`, `許可_<lord>篇`.

### 3.3 Person labels (confirmed)

The game builds 17 labels per person as `<prefix>_<PID stem>`: 面識, 加入, 死亡, 消失, 開示, 生存ロスト, 勧誘,
ローカル_同期_死別退場, 敵対, 行動, 衣装, 引継, 贈答, ファストトラベル, 絆数, 贈数, 勧誘交渉済.

| Family | Category | Values |
|---|---|---|
| `贈答_<person>` | 限定 | **decimal YYYYMMDD** of the last gift to that person (e.g. 14490308); inferred use: limits gifts per period |
| `行動_<person>` | 限定 | 1..3 |
| `贈数_<person>` | 共有 | gift count 1..3 |
| `絆数_<person>` | 共有 | 1..10 |
| others | | 1/2 flags |

### 3.4 Shop stock (限定, confirmed)

- `購入_<shop>_<item stem>` and `交換_<shop>_<item stem>[_<exchange item stem>]...`, where `<shop>` is the shop group
  key (e.g. `帝都ダグシオン_武器屋`) and the item stem is the IID without `IID_`.
- Registered as int variables (max = the row's stock value).
- **Value: bits 0..6 = quantity bought; bit 7 = the stock slot has been listed/initialised.** Remaining stock =
  `在庫数 − (v AND 0x7F)`. So 128 = listed, nothing bought; 129 = one bought (sold out for stock-1 rows). The exact
  trigger that sets bit 7 is inferred.

### 3.5 Notable Deeds and other 実績 entries (confirmed)

| Family | Values | Meaning |
|---|---|---|
| `実績状態_<group>[_N]` | 2, 4 (1 rarely) | deed claim state: **2 = reached/claimable, 4 = claimed**. The first row of a reward group uses the bare group name; later rows add `_N` (1-based). Claiming adds Karma Shards and sets 4 |
| `実績表示_<group>[_N]` | 1 | shown/unlocked in the deed list (inferred) |
| `実績_<counter>` | int ≤ 999999 | deed counters, e.g. `実績_外伝クリア数`, `実績_プレイ時間` |
| `実績_技能<rank>_<weapon>_<w>`, `実績_試験済_<class stem>_<w>` | bit masks | person bit sets keyed by Person serial number: `_0` holds bits 0..31, `_1` bits 32..63 (weapon rank reached / class exam passed) |

### 3.6 Other dynamic families (examples)

| Family | Meaning |
|---|---|
| 限定 `通知_<key>` | a new-action notice was shown |
| 限定 `拠点_開示済_<facility>` | hub facility disclosed |
| 中継 `拠点_FT_<hub>_<area>_<locator>` | hub fast-travel point unlocked (e.g. `拠点_FT_Hub_000_A1_ファストトラベル_闘技場前`) |
| 状況 `Gmap_興行済_1部_<town>` | Part 1 performance done in that town |
| チュートリアル `TUTO_<key>_注目` | tutorial "attention" marker |
| 回想 `回想_<category>_<recall key>_<n>` | recollection: an int holding a packed snapshot taken when the scene was first seen (not decoded). `<category>` is one of 無し, 支援会話, `デモ_<part>_<lord>篇`, `動画_<part>_<lord>篇`. Each is followed in file order by a value-1 partner entry whose name is not known |

### 3.7 Deriving variable names

No name is stored for kind-0 entries, only `IdHash(name)`. To label an entry:

1. Build candidate names from the families above, the Variable.bin key column and table keys (PID stems, item stems,
   shop groups, reward groups, recall keys, ...).
2. Compute `IdHash = FNV-1 64(name) AND 0xFFFFFFFF` and compare.
3. Use the **registry order**: each category map is written in registration order, so an unknown hash sits next to
   its family (person labels, then the shop block, then facility labels, ...).

About 195 kind-0 hashes seen in real saves remain unnamed: most are the value-1 回想 partners, plus a few 限定
facility labels and some 状況 facility-state names ending in a digit.

---

## 4. GameReliance (supports)

Writer `main+0x8ebf60`, reader `main+0x8ec180` **(confirmed)**.

```
[ptr][u8 version 2][u16 n = 2628] n x {u32 FNV1-32("RID_A_RID_B"), u16 points, u16 count, u32 stamp} [1 pad byte]
```

- Every save holds all **2628 pairs**, so editing never adds records. The block is always 31544 bytes.
- The key is FNV-1 32 of the two **Reliance.bin row keys** (`RID_...`, not PIDs) joined by `_`, A before B in row
  order. Unknown hashes are skipped on load.
- **Levels** from points: thresholds 0/400/1400/2400/3200 = level 1 (none), C, B, A, S. The level byte is recomputed
  from points on load.
- **Cap:** points reach 3200 only if both persons have `加入_`, `面識_` or `開示_` set; otherwise they stop one short of
  the next level.
- `count`: an accumulator clamped to [0, 10000], non-zero only mid-chapter; meaning unknown.
- `stamp`: a recency counter (max of the two units' stamps + 1).
- **Conversations watched are not here:** they are 中継 `支援会話_<A>_<B>_<rank>` = 1 plus the 実績
  `実績_支援会話<R>再生数` counters. S-rank conversations exist only for Eshmel pairs and run only in Part 3.

---

## 5. GameSchedule (top-level block)

Writer `main+0x871e30`, reader `main+0x872000` **(confirmed)**. 24 bytes. Offsets from block start + 4:

| Off | Type | Meaning |
|---|---|---|
| 0 | u8 | version 1 |
| 1 | u16 year, u8 month, u8 day, u8 hour, u8 minute, u8 second | **A = current in-game date/time** (also in file 0x32..0x38) |
| 8 | same 7 bytes | **B = time limit**: the date of the next deadline (期限, type 18) timeline row. The clock tick clamps A to B − 1 s, so in-game time can't pass the deadline. 0 at chapter starts |
| 15 | f32 | sub-second accumulator of the clock |
| 19 | u8 | 0; written, skipped by the reader |

---

## 6. Playtime blocks

### 6.1 SequenceTime: playtime per scene kind

Writer `main+0x71eb40`, reader `main+0x720f40` **(confirmed)**.

```
u8 version 0 | u64 FNV1-64(current scene kind) | u16 h, u8 m, u8 s (playtime when that scene was entered) |
u8 n = 17 | n x {u64 FNV1-64(scene kind), u32 packed h:m:s}
```

The "entered" stamp is at member offset +13. **Invariant (every real save):** the sum of the 17 per-scene totals
equals the stamp, to the second.

### 6.2 SectionTime: playtime per story section

Writer `main+0x7200c0`, reader `main+0x7211e0` **(confirmed)**.

```
u8 version 0 | u32 current section index | u16 h, u8 m, u8 s (playtime when the section started) |
u16 slot count = 560 | 560 x slot        slot = 00 (empty) | {u8 minutes | 0x80, u8 seconds, u16 hours}
```

- **Section index = clamp(56 × lord + 14 × part + chapter, 0, 560)**, computed from the GameUserData scalars. For
  example Cai Part 1 Chapter 5 = section 75; the Eshmel prologue is 504 + chapter.
- 560 slots = 10 lords × 56. Slot i = total time spent in section i.
- The stamp is at member offset +9. **Invariant:** the sum of the slots equals the stamp.

---

## 7. GameMiracle (god blessings)

Writer `main+0x918cc0`, reader `main+0x918e40` **(confirmed)**. 88 bytes.

```
u8 version 2 | u16 a (100) | u16 b (0) | u8 n = 7 | n x {u64 FNV1-64("MIR_<god>"), u8 flags, u16 exp}
```

- Order: フォトナ (Fortuna), マーズ, クレール (Credna), カーラ, ジュラ, スミルノス, アウロラ (Aurora).
- `flags` bit 0 = equipped. At most 4 can be equipped, and Fortuna (always 1) counts.
- The level is not stored: it comes from exp against the blessing's three cumulative thresholds in Skill.bin
  (`加護` 経験１/２/３; e.g. Aurora 20/50/100, Credna 15/40/80). Exp is clamped to [0, 経験３] on load.
- `a`/`b` are 100/0 in every real save; meaning unconfirmed.
- The reader zeroes all exp and flags first; unknown ids are skipped.
- Copies: each restart payload; `GameConverge/CommonInfo/GameMiracle` holds the cross-route maximum.
- Not to be confused with the boons `因果報酬_レベル_加護_<god>` (a shop discount).

---

## 8. GameRecord (war record)

Writer `main+0x9ac3c0`, reader `main+0x9ac540` **(confirmed)**. The same class is used in GameUserData and inside each
RouteInfo record.

```
u8 version 7 | i8 order_p1 | i8 order_p2 | u16 n | n x chapter record | u16 m | m x map record
```

- `order_p1` / `order_p2`: in a route record, the **global clear-sequence index** of that route's Part 1 / Part 2
  across all routes and both parts (0-based, `0xFF` = never). `0xFF 0xFF` in GameUserData's own copy.
- One chapter record per cleared chapter, prologue included.
- The map-record list is never populated in real saves (layout from the writer only).

Chapter record:

| Field | Encoding | Meaning | Status |
|---|---|---|---|
| lord, part, chapter | 3 × u8 | | confirmed |
| b3 | u8 | `G+0x44` at chapter start (army level snapshot) | source confirmed, meaning inferred |
| turns | u16 | turn count | inferred |
| map | u32 | IdHash of the battlefield key (e.g. `SC11_05`) | confirmed |
| MVP person | u32 | IdHash of the MVP's PID | confirmed |
| MVP portrait | u32 | IdHash of `PRF_*` | confirmed |
| MVP face | u32 | IdHash of `FID_*` | confirmed |
| MVP name tag | `u8 ver` (0 = none); ver 3: `u16 n, 07 + utf8 + 00, 5 look bytes` | present only when the MVP is the custom-named protagonist. **These are extra copies of the protagonist's name** | confirmed |
| chapter playtime | u16 h, u8 m, u8 s | time spent in the chapter | confirmed |
| units | u8 | player-army units not marked "left the map" | confirmed count |
| flag-0x20 units | u8 | army units with unit flag 0x20 | count confirmed, meaning unknown |

---

## 9. Other members

### 9.1 TaskData (current hub objective)

Writer `main+0x980800`, reader `main+0x980a40` **(confirmed)**.

```
u8 version 1 | u8 active | u64 hub id | u16 (len + 1) message label | u8 n | n x string tag
```

Filled only in hub saves (e.g. hub `帝都ダグシオン`, label `MID_SUP_1KAI_c05_...`, tag `[シナリオ進行受付]`). Chapter
checkpoints have the empty form. This is the objective tracker; side quests are in `SubquestData`.

### 9.2 CheckPerson

`u8 n | n × {u64 FNV1-64(PID), u8 Person serial number}`, n = 73, identical in every save (e.g. `PID_カイ` 1,
`PID_イシュマール` 39, gods 100..105). **The reader skips it entirely** **(confirmed)**: it is a write-only snapshot of
the serial-number table that mount bonds depend on. Recruitment state is `面識_`/`加入_` in the variables.

### 9.3 Storage

A struct with 3 members written by the pool writers: `TransporterData` (`EFBE8231`), `LegionData` (`7644ADD3`),
`RideData` (`09B9F71F`). Normally empty stubs. In a Part 2 save it holds the full 1329-slot Part-1 convoy: the route's
Part-1 convoy is parked here while the War Arc army takes over **(layout confirmed, fill timing inferred)**. Always
empty in restart payloads.

---

## 10. The timeline key (story position)

The file header (0x49), the GameUserData anonymous object (body+76) and every timeline-stack entry hold the same
kind of key **(confirmed in code; every key in real saves resolves to a timeline row)**:

```
key = FNV1-64( "<group>_<type name>_<部>_<章>_<n>" )        UTF-8; a result of 0 is stored as 0xFFFFFFFFFFFFFFFF
```

Built per timeline row by `main+0x891340` with the key-string builder `main+0x8997a0`; registry at `main+0x2231720`;
loader `main+0x898120` (reads `u8`, `u64`; 0 or an unknown key gives a null cursor).

| Part | Rule |
|---|---|
| `<group>` | the row's group (the `#key` column), e.g. `カイ篇`, `序章`, `戦争篇`, `1部クリア` |
| `<type name>` | one of 89 type names indexed by the row's `種類`, e.g. 章題 (chapter title), 拠点 (hub), 戦闘 (battle), 迷宮 (dungeon), 呼ぶ (call), 戻る (return), 期限 (deadline), 保存2. Several names use **half-width katakana** (e.g. `ﾗﾍﾞﾙ`, `ｼﾞｬﾝﾌﾟ`) |
| `<部>`, `<章>` | decimal part and chapter, **inherited**: a row with 部 = 0 or 章 = 0 takes the previous row's value (cascading; the first row of a group keeps its own) |
| `<n>` | 1-based count, walking back while 章 is unchanged, of rows with the same `種類` and an empty `ハッシュキー` |

If a row's `ハッシュキー` is set, its FNV-1 64 is the canonical key instead (no row in update 131072 has one). A legacy
alias with the part written as a raw byte is also registered.

Examples **(confirmed)**:

| Key string | Key |
|---|---|
| `カイ篇_章題_1_5_1` (Cai Part 1 Chapter 5 start) | `0x1C2BF1F1E8FF4E1A` |
| `1部クリア_保存2_0_0_1` (the row that writes `Part_<lord>_2`) | `0x1ACC8881C72B43D6` |

**Meaning.** The key is the **timeline cursor**: the row executing when the save was written. Chapter-start
checkpoint = the chapter's 章題 row; hub/world-map/dungeon autosave = the current 拠点 row; battle save = the 戦闘 row;
DUNGEON/CMDBTL restart = the 迷宮 row. The call stack (§1.2) holds the 呼ぶ rows waiting for a return.

**Story position** = the cursor row + the call stack + GameSchedule date A + the SectionTime index + the scene kind +
the variables (+ HubData, TaskData and RecommendData for hub state). Lord/part/chapter (body+3..+5) are **not** derived
from the key. See [editing guide §17](editing-guide.md#17-story-position).
