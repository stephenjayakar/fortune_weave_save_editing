# Known tool pitfalls

Mistakes that save tools for this game have made, or that are easy to make. Each entry says what goes wrong and what
to do instead. None of them are specific to one tool; check any editor you use against this list.

---

## 1. HP is one byte at unit +64

**Pitfall.** Reading or writing current HP as a u16 or u32 at unit +64.

**Effect.** A u16 write clobbers +65 (pending HP); a u32 write also clobbers +66 (HP stock) and +67 (max HP stock).
These bytes are 0 in normal player units, so the damage is often invisible until it isn't.

**Do this.** Write exactly one byte at +64. Leave +65..+67 alone.

## 2. Unit +66 is not "max HP"

**Pitfall.** Treating +66 as maximum HP. Maximum HP is computed from stats; it is not stored.

**Effect.** Editing "max HP" there writes the HP stock (extra HP bars, normally only on bosses).

**Do this.** Change max HP through the HP stat gain at +37.

## 3. Mode and difficulty labels are swapped in many tools

**Pitfall.** Labelling GameUserData body+1 / payload[14] / file 0x26 "difficulty" and body+2 / payload[15] /
file 0x27 "mode".

**Correct values (confirmed in code).** body+1 = **GameMode** (0 Casual, 1 Classic); body+2 = **Difficulty**
(0 Normal, 1 Hard, 2 Lunatic/Maddening); body+49 = difficulty chosen at new game.

**Do this.** Read such tools' labels the other way round. A consistent difficulty change also has to update body+49,
the restart copies and every unit's `grade` ([editing guide §16](editing-guide.md#16-difficulty-and-mode)).

## 4. The real save date is at file 0x58

**Pitfall.** Reading the real-clock save date at summary +86 (file 0x76).

**Effect.** File 0x76 is the u64 job key of Eshmel's class, so the "saved at" date comes out wrong or empty.

**Do this.** The date is at summary +0x38 = file 0x58: `u16 year, u8 month, day, hour, minute, second`.

## 5. Boon keys come from the game's internal short-key table

**Pitfall.** Building `因果報酬_レベル_<key>` / `因果報酬_状態_<key>` names from Reward.bin's key column.

**Effect.** 14 of the 31 keys differ from the game's own keys (e.g. `技能経験黒魔法` vs the game's `技能経験黒`,
`加護消費_マーズ` vs `加護_マーズ`). Unregistered names are **silently dropped on load**, so those boons disappear.

**Do this.** Use the game's keys ([editing guide §13](editing-guide.md#13-boons-of-salvation)).

## 6. Rename padding and alignment

**Pitfall.** When renaming the protagonist, keeping the old padding and appending extra bytes to fix the size
(e.g. 6 pad bytes in CommonInfo where 2 are correct).

**Effect.** Objects with surplus padding are not what the game writes; such a build hung on a black screen after
Continue (suspected cause). Renaming only some copies leaves inconsistent names.

**Do this.** Rename size-neutrally, or splice and re-pad each enclosing object minimally to 4. Rewrite every copy:
CommonInfo profile, all GameRecord MVP name tags (main, per route, per restart payload) and the protagonist's
UnitEdit ([editing guide §20](editing-guide.md#20-protagonist-name)).

## 7. Heuristic tree walkers

Anonymous child objects carry no marker, so a walker without the schema has to guess which u32 values are pointer
words. Known failures:

- **A false anonymous object swallows a named member.** A candidate starting just before `GameUserData/Variable` can
  cover it, and `Variable` then disappears from the tree. A walker must at least refuse anonymous candidates that cover
  one of their parent's member starts.
- **Flat record lists look like objects.** `GameReliance` is a flat list of 12-byte records; once the variable maps
  grow, a record word can equal a real object end and be taken for an object. A `GameMiracle` hash can do the same.
  Ordinary play that grows the variable maps can trigger this.
- **Deep false chains** in large HubData blobs and DUNGEON restart payloads.

**Effect.** Splices get refused or, worse, shift a word that is not a pointer, e.g. corrupting a blessing id or a
support key.

**Do this.** Parse with an exact, schema-driven reader. At minimum treat `GameReliance` and `GameMiracle` as leaves,
and check both after every splice.

## 8. Spot keys: read before aligning

**Pitfall.** Aligning to 4 before reading each `GmapSpotMgr` key.

**Effect.** The first key (`帝都ダグシオン`, `0x621A6FD3`) sits unaligned at block offset +7 and is misread as
`0x00621A6F`.

**Do this.** Read the u32 key immediately after the previous byte, then align, then read the spot object.

## 9. Duplicated mount copies and the serial counter

**Pitfall.** Editing a ridden mount in only one place, or adding mounts without touching the serial counter.

**Effect.** A ridden mount exists twice, byte-identical: inline in the rider's unit record and in `RideData` with
`owner` = the unit number. Diverging copies are not a state the game produces. New mounts with serials above
GameUserData body+39 (`RideSerial`) mean the game's next mount reuses a serial.

**Do this.** Edit both copies together. After adding mounts, set body+39 to at least the highest serial, in the main
GameUserData and in each restart copy (for that copy's own pool).

## 10. Mount `w4` layout

**Pitfall.** Reading mount `w4` as "byte 0 = serial, byte 3 = flag".

**Correct layout (confirmed in code).** Bits 0..23 = serial (24-bit), bits 24..27 = the lord route the mount was
obtained on, bits 28..31 = unique-mount number (serial 0 for unique mounts). Byte 3 = 0x01 just means "obtained on
Cai's route".

## 11. Route-bitmask paralogue zeroing

**Pitfall.** Marking paralogues cleared only through their per-route SideStoryData entries.

**Effect.** On every load (and every map/hub sequence start) the game zeroes the state of every paralogue that isn't
on the current lord's route. Clears set only there vanish on the next save.

**Do this.** Add each cleared paralogue's SSID to the **cleared-ever list**, which is never zeroed
([editing guide §15](editing-guide.md#15-paralogues)).

## 12. Variables are dropped if not registered

**Pitfall.** Writing kind-0 variables with invented or misspelled names, or in the wrong category.

**Effect.** On load, kind-0 hashes the game hasn't registered are silently skipped. The edit "takes" in the file but
never reaches the game.

**Do this.** Use names from real saves or from families the game registers, in the right category
([doc 03 §2.2](03-game-user-data-and-variables.md#22-what-survives-a-load)).

## 13. Non-canonical variable values

**Pitfall.** Writing values 16..254 as a single byte.

**Effect.** It loads (the reader accepts any byte ≠ `0xFF`) but differs from what the game writes; the next save
rewrites it.

**Do this.** One byte only for 0..15, otherwise `0xFF` + i32.

## 14. Restart payloads exclude GameConverge

**Pitfall.** Assuming a restart payload is a full mirror of the save, or forgetting that it exists.

**Effect.** Restart payloads have no `GameConverge` (routes, CommonInfo, Reliquary), an 8-byte `GameRestartData` and
an empty `Storage`, but their own copies of everything else. Edits made only to the top-level payload are rolled back
by "Restart chapter/battle". Code that expects `GameConverge` in every payload fails on them.

**Do this.** Edit the filled restart slots explicitly, or accept the rollback.

## 15. Convoy slot count

**Pitfall.** Appending or removing convoy slots, or putting stack items in arbitrary slots.

**Effect.** The convoy must have exactly 999 regular + 330 stack slots once initialised; a wrong slot count corrupts
it. Stack items get re-homed to their fixed slot on load anyway.

**Do this.** Keep n = 1329; write stack items into their fixed slot; don't initialise an `n = 0` convoy partially.

## 16. Raw data documents

**Pitfall.** Assuming the data document is always zstd (type 3).

**Effect.** The game stores data raw (type 1) whenever compression would not shrink it; tools that only handle type
3 fail on such files.

**Do this.** Accept both types. Writing type 3 always is fine.

## 17. Unit detection and force lists

**Pitfall.** Recognising unit records by "version 36 followed by a zero byte", or reading only the first UnitPool
force list.

**Effect.** On-map units have flag byte 1 (and an AI object), so battle saves and battle restart copies are
mis-walked. Enemy and roster lists are skipped.

**Do this.** Parse UnitPool as `repeat {u8 force, u8 count, units}` until `0xFF`, and parse each unit with its flag.

## 18. Class list values and stat counts

**Pitfall.** Treating class-list entries as `(JID, a, b)` byte pairs, or reading 9 stats / 9 growth bytes.

**Correct layout.** Class-list values are one u16 (`exp | 0x400`), and there are **11** stats and **11** growth
remainders (Sight and Mov follow Cha).

## 19. Deed claim values

**Pitfall.** Treating `実績状態_*` values 1..3 as claimable.

**Correct values.** 2 = reached/claimable, 4 = claimed (1 appears only rarely).

## 20. `Data` is not a root block

**Pitfall.** Listing `Data` (or expecting `HubSeq` always) among the payload root blocks.

**Correct.** `Data` is the member of a restart slot. The optional root blocks are `MapSeq`, `HubSeq` and `GmapSeq`,
plus `CmdBtlSeq` in CMDBTL restart payloads only.

## 21. Zeroed growth remainders and stale `grade`

**Pitfall.** Creating units with zeroed growth remainders (+26), or changing difficulty without updating `grade`.

**Do this.** New units: `(3 × Person growth%) mod 100` per stat. A difficulty change: set every unit's `grade` to the
new Difficulty value.
