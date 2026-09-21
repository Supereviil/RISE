# `svr_state_0.stt` — format map

Reverse engineered from `Radiation Island` (Steam), save version **6**.
Samples: `samples/svr_state_0.A.stt` (4815 bytes), `samples/svr_state_0.B.stt` (4793 bytes).

**Multi-island player state:** later saves add `svr_state_2.stt` / `svr_state_4.stt`
etc. Each file is a full save with its **own** player entity. Inventories/stats/position
diverge per island; unlock lists and capacity tend to match. Editors must **not**
auto-sync player fields across siblings — that is real format behaviour, not staleness.

Every offset below is a byte offset into sample A. Anything marked **CONFIRMED** was
verified by a full-file structural walk that consumes the file with no
desynchronisation; anything marked **THEORY** is a best reading that has not yet been
proven by a controlled before/after diff.

---

## 1. Engine and container format

The game is **not** Unity. There are no `.assets`, `.bundle`, `level*` or
`globalgamemanagers` files, no `il2cpp`/`Mono` runtime, and no `UnityPlayer.dll`.
What is there instead:

| Evidence | Meaning |
| --- | --- |
| `win32/` and `data/` split | Atypical Games' in-house "Skyline" engine |
| `binsh/`, `comsh/` shader dirs | custom shader pipeline, not Unity's |
| `data/configs/*.cfg`, `*.lst` | custom config/asset lists |
| `zlib1.dll` present | zlib is available, but is **not** used on the save |

So there is no Unity binary serialisation, no `PlayerPrefs`, no JSON and no YAML
anywhere in the save path.

**The save is a raw, byte-packed, little-endian dump of the live object graph.**

- No compression. Entropy is low and ASCII ids are plainly visible.
- No encryption.
- **No alignment padding.** Fields sit immediately after one another; a `float32`
  routinely starts on an odd offset. This is the single most important property of the
  format — any parser that assumes 4-byte alignment will desynchronise immediately.
- **No magic number.** The file opens with `06 00 00 00`, the version.
- **No checksum, no stored file length, no stored record count for the file as a
  whole.** This is confirmed by construction: sample B is 22 bytes shorter than
  sample A and both load, and the game accepts files whose length changed.
  Consequence: many regions may grow or shrink safely. **Exception:** changing
  the player backpack or hotbar *record count* reliably breaks load — see
  §10. Same-count edits (quantity, in-place id swaps) remain safe.

### String encoding — `PStr`

One primitive is used for every string:

```
uint8  L          // total length INCLUDING the NUL
byte   chars[L-1] // ASCII
                  // the last of those bytes is 0x00
```

So `'res.flint'` is stored as `0A 72 65 73 2E 66 6C 69 6E 74 00`.

**An empty string has two distinct encodings and the game uses both:**

| Bytes | `L` | Decodes to | Used for |
| --- | --- | --- | --- |
| `00` | 0 | `''` | unnamed world objects (ground pickups) |
| `01 00` | 1 | `''` | empty equipment slots, unnamed spawners |

This is the single nastiest trap in the format. Both forms decode to the same empty
string, so a writer that normalises empties to `L = 0` produces a file that is one byte
short per bare equipment slot — and because everything downstream is byte-packed, that
shifts the whole tail. It was caught only by byte-exact round-trip comparison, which
failed at `0x0CF2`, precisely the empty feet slot. It bit a second time in the spawner
list, where a save with two unnamed spawners came out exactly 2 bytes short. The raw
length byte is therefore carried through the model (`TRIObject.InstNameRawLen`,
`TRIEquipSlot.ItemIdRawLen`, `TRISpawner.NameRawLen`) and replayed on write.

### Item identifiers are strings, not integers

Items are referenced by namespaced ASCII id — `res.flint`, `wep.bow`,
`cloth.wool.hat`, `food.medicine`, `dpl.inventorybox`, `build.part.pianno`. There is
no integer item table in the save. The namespace prefix is a reliable category:

| Prefix | Category |
| --- | --- |
| `res.` | raw resource / crafting material |
| `wep.` | weapon |
| `ammo.` | ammunition |
| `food.` | food, water, medicine |
| `cloth.` | wearable |
| `tool.` | tool |
| `build.` / `dpl.` | buildable / deployable |

`data/configs/items.lst` lists asset names and is useful as a source of *candidate*
ids, but the ids the save uses are runtime ids and the two lists are not identical.
`RISE.ItemDB.pas` therefore separates **confirmed** ids (observed in a real save) from
**candidates** (inferred), and grows the confirmed set automatically as saves are
loaded.

Naming guesses from `items.lst` are sometimes wrong even when they look consistent.
Organic pickups confirmed `wep.fact.axe` / `wep.fact.pickaxe`; the earlier catalogue
spellings `wep.factoryaxe` / `wep.factorypickaxe` do **not** match real save data and
remain candidates only.

### Condition / durability (CONFIRMED split)

| Kind | Behaviour | Confirmed maxima / notes |
| --- | --- | --- |
| Gathering tools (axes, pickaxes) | real durability bar | `wep.stoneaxe` = 20.0; `wep.ironaxe` = 60.0; `wep.titaniumaxe` = 100.0 |
| Melee (top-tier) | real durability bar | `wep.katana` ≈ 200.0 (new top-tier melee class) |
| Ranged / combat (`wep.piratepistol`, `wep.bow`, `wep.arbalete`, `wep.medievalrifle`) | dummy durability float | Condition stays `1.0` across every sample regardless of use |
| Clothing (uniform / strong / leather / wool / radsuit) | **true 0–100 durability** (CONFIRMED dynamic wear) | freshly equipped = 100; worn sample: uniform shoes = **59.0** |
| Special-variant (`wep.flameclawclub`) | **not durability** | Condition holds a variant-specific effect float; flagged `SpecialVariant` in `RISE.ItemDB` until more samples decode it |
| Resources / food / ammo | stack marker | typically `1.0` |

`HasDurability = False` in `RISE.ItemDB` marks non-tracking weapons for UI display
(`N/A`). `SpecialVariant = True` marks items whose Condition float must not be
treated as wear (UI shows `….… (variant)`; `RepairEverything` skips them). A
previously shipped bug in `RepairEverything` used a flat `20.0` fallback for every
`ikWeapon` without a registered max, which could *lower* a titanium axe from ~100
to 20; that fallback is now an unknown sentinel (`-1`) and repair never decreases
condition.

### A note on `.cfg` obfuscation

`data/configs/*.cfg` and the `.str` localisation files are obfuscated with a
byte-position-dependent transform — not a fixed-key XOR, which was ruled out by
brute force over bit permutations combined with XOR masks (`analysis/crack_cfg.py`).
Cracking them was abandoned as unnecessary: the save itself carries plaintext ids, so
the item catalogue can be built empirically. This is the one deliberate gap in the
analysis.

---

## 2. Top-level layout

```
0x0000  Header                    fixed, 559 bytes
0x022F  World object list         variable, self-terminating
        Terminator                9 bytes
        Spawner list              int32 count + variable
        Player entity             variable
        Tail region               everything else, preserved verbatim
```

Only the header has fixed offsets. Everything after `0x022F` moves when content
changes, so a real editor **must** parse rather than poke fixed addresses. The offsets
quoted for later sections below are the values *in sample A* and are given for
orientation only.

---

## 3. Header — `0x000 .. 0x22E` (CONFIRMED)

| Offset | Type | Sample A | Meaning |
| --- | --- | --- | --- |
| `0x000` | `int32` | 6 | save version |
| `0x004` | `int32` | 0 | unknown, constant |
| `0x008` | `int32` | 1 | unknown, constant (slot index?) |
| `0x00C` | `float32` | 502.186 | **play time, seconds** |
| `0x010` | `byte[523]` | mostly 0 | **progression bitfield** |
| `0x21B` | `float32` | -1451.82 | **player X** |
| `0x21F` | `float32` | 9.99 | **player Y** |
| `0x223` | `float32` | -2428.80 | **player Z** |
| `0x227` | `int32` | 0 | **count** of the vec3 list below |
| `0x22B` | `vec3[count]` | — | 12 bytes per entry |
| — | `int32` | 0 | header tail |
| — | — | — | object list starts (`0x22F` only when count = 0) |

### The header is variable length

`0x227` is **not** 8 bytes of padding — it is a counted `vec3` list:

```
int32   Count
vec3    Entries[Count]        // 12 bytes each
int32   Tail
```

Sample A has `Count = 0`, which makes the block look exactly like 8 fixed bytes and
is how it was originally misread. A later save from the same playthrough has
`Count = 1`, which pushes the object list from `0x22F` to `0x23B`. Reading the block as
fixed put the spawner count inside the list and desynchronised everything after it.

So **the only truly fixed offsets are `0x000` through `0x226`.** `RI_OFS_OBJECTS` is
kept as a constant for the `Count = 0` case but code should use
`TRISaveFile.ObjectsOffset`.

### The progression bitfield — `0x010 .. 0x21A`

523 bytes (4184 bits), overwhelmingly zero on a fresh-ish save. In sample A only three
bytes are non-zero:

| Offset | Value | Bits |
| --- | --- | --- |
| `0x014` | `0x20` | `00100000` |
| `0x038` | `0x40` | `01000000` |
| `0x047` | `0x01` | `00000001` |

This region holds **permanent unlocks** (map areas, story pages, completed tasks,
crafting recipes) **and** some **transient status flags** that set and clear during
play. That the region is the progression / status bitfield is CONFIRMED by size,
position and sparse-bitfield character; most individual bit meanings remain unmapped.

Mapped bits so far — each claimed from exactly one clean isolated before/after
`fdiff`, so kept as **THEORY**, not confirmed:

#### Permanent unlock bits (theory)

| Absolute bit | Meaning (theory) | Pair audit (2026-08-11) |
| --- | --- | --- |
| 0 | crafted first backpack (`recipe.inventorybox`) | **Unverifiable from archive** — no preserved before/after pair that isolates this bit. Closest archived delta is contaminated (see below). |
| 352 | discovered a map area | **Unverifiable from archive** — no preserved before/after pair; never appears as a lone progression delta in retained `fdiff_output*.txt`. |

#### Transient status flags (theory) — not permanent unlocks

| Absolute bit | Meaning (theory) | Pair audit (2026-08-11) |
| --- | --- | --- |
| 1 | sleep-related status (e.g. “well-rested”) — **sets and clears** with recent sleep activity; do **not** treat as a permanent unlock | **Unverifiable from archive** — no preserved before/after pair; no archived `fdiff` shows bit 1 toggling in isolation. |

Bit index = byte-index-into-flags × 8 + bit-within-byte (LSB = bit 0 of that byte).
So bit 0/1 live in the first flags byte at `0x010`; bit 352 is byte index 44
(`0x010 + 44`).

#### Multi-island pair audit (do not promote these bits without retest)

Multi-island saves (`svr_state_0` / `_2` / `_4`) each carry their **own** copy of this
bitfield; copies can diverge by a few bytes. Any before/after that accidentally
compared different island suffixes would be invalid for bit mapping.

What the retained `fdiff_output*.txt` files actually used:

| Retained pair | A | B | Same island suffix? | Notes |
| --- | --- | --- | --- | --- |
| `fdiff_output.txt` | `svr_state_0_1.stt` | `svr_state_0.stt` | **Yes** (`_0` lineage) | Progression **unchanged** |
| `fdiff_output_bleed*.txt` | `svr_state_0_1.stt` | `svr_state_0_bleed.stt` | **Yes** | Progression unchanged; unlock list multi-id churn |
| `fdiff_output_bleed_tight.txt` | `svr_state_container.stt` | `svr_state_0_bleed.stt` | **Likely yes** — both header `0x004 = 0` (pre-multi-island idx); filename `container` has no `_N` | Only archived progression delta: `0x010` `0x01→0x00` (**bit 0 clear**). Contaminated (inventory/unlocks/world also change) — **not** a clean craft isolation |
| `fdiff_output_fall.txt` / `*_healed.txt` / `*_fracture_healed.txt` | `svr_state_0*` | `svr_state_0*` | **Yes** | Progression unchanged; unlock lists change in bulk |
| `fdiff_machete.txt` | `svr_state_0.stt` | `svr_state_0-bad.stt` | **Yes** | Editor experiment, not bit mapping |

**No retained pair uses `svr_state_2` or `svr_state_4`.** No retained pair documents an
isolated bit-1 or bit-352 toggle. Treat bits 0 / 1 / 352 as **operator-reported
theory pending retest on a single explicit `svr_state_N` suffix** (copy → one
action → quit → `fdiff` same `N`).

Further bits should still be mapped one at a time: save, unlock one thing, save,
`RISECmd fdiff`. `RISE.HexDiff.DiffProgressionFlags` reports per-byte bit
breakdowns. Setting the whole region to `0xFF` (`SetAllProgressionFlags`) is
available but likely to trip the game's own consistency checks.

---

## 4. World object list — from `0x022F` (CONFIRMED)

An array of records, terminated by `int32 TypeId = 0`.

```
TRIObject:
  int32   TypeId       // 1 = placed object, 3 = ground pickup
                       // 0 = end of list (peek only, do not consume - see below)
  uint16  ClassTag     // 7 for placed, 0x0101 / 0x0103 for pickups
  PStr    InstName     // 'chest_133', or empty for unnamed pickups
  PStr    ClassName    // 'dpl.inventorybox', 'res.flint', ...
  uint8   LeadByte     // always 0 in all samples
  float32 Pos[3]
  float32 Rot[4]       // quaternion
  int32   NetId        // 50137, 50140, ... sequential network id
  // then, ONLY if ClassName = 'dpl.inventorybox' AND ClassTag = 7:
  uint8   MarkerA      // 2
  uint8   MarkerB      // 5
  uint16  ItemCount
  uint16  InvFlags
  TRIItem items[ItemCount]
  // then, ALWAYS if ClassName = 'dpl.furnace':
  PStr    InputSlot    // "none" when empty
  uint16  InputQty     // on disk ONLY when InputSlot <> "none"
  PStr    FuelSlot     // "none" when empty; real item id when occupied
  uint16  FuelQty      // on disk ONLY when FuelSlot <> "none"
  uint16  StoredCount
  // each stored slot:
  PStr    ItemId       // "none" = unused slot
  uint16  Quantity     // omitted on disk when ItemId = "none"
  float32 FurnaceHeat  // THEORY: heat/burn-progress (16.0 idle, 4.0 mid-use)
  int32   FurnaceStatus // THEORY: status flag or timer (observed 0)
```

The fuel slot is a `(PStr, uint16)` pair when occupied. When `FuelSlot` is
`"none"`, the quantity word is omitted (logical qty = 0) — confirmed on the
idle-smelt sample. Stored `"none"` slots are likewise a bare PStr. An occupied
sample (`res.wood`×3 fuel, `food.steak`×18 + `"none"` store, heat 4.0) was
pinned by working backward from the next object's verified header.

World-chest inventory is keyed by **class name + ClassTag = 7**. Player-placed
`dpl.inventorybox` deployables use ClassTag `0x0107` and have no item blob.
`dpl.sleepingbed` is a `TypeId = 1` placed object with no item list, so keying
off `TypeId` alone desynchronises the parser.

A used furnace always carries the smelt/store trailer after `NetId`; skipping it
desyncs the object list and the spawner count. Idle `dpl.campfire` samples have
**no** trailer after `NetId`; whether a used campfire shares the furnace layout
is unconfirmed — do not assume it does.

`InvFlags` was initially missed, which made `chest_136` appear to have
`ItemCount = 65537` when read as an `int32`. Splitting the field into
`uint16 ItemCount; uint16 InvFlags` resolved it and made every container in both
samples parse consistently.

Sample A holds 25 objects: 11 `dpl.inventorybox`, 1 `dpl.sleepingbed`, 13 ground
pickups. Sample B holds 24, of which 10 are containers.

### `TRIItem` — the universal stack record (CONFIRMED)

```
TRIItem:
  uint16  Slot         // meaning depends on which array — see below
  uint16  Quantity
  byte    Mid[16]      // all zero in every sample; probably a reserved GUID
  float32 Condition    // 1.0 for resources, 0..100 for gear, durability for tools
  PStr    ItemId
  uint16  Extra        // PRESENT IN SOME ARRAYS ONLY - see below
```

**Backpack `Slot` is a column, not a unique index (CONFIRMED).** On the hotbar,
`Slot` is the bar position (unique). In the **backpack** it is only the column
index in a fixed **6-wide grid** (`0..5`); the row is implied by array order and
is not stored. An 18-item backpack (two capacity upgrades applied) from a real
non-edited save showed `0–5` repeated exactly three times. Repeated backpack
`Slot` values across a large backpack are therefore **expected**, not evidence of
corruption or misalignment — `Validate` must not treat a repeated `Slot` as
invalid on its own. Uniqueness, if enforced at all, is only within a single grid
row (`Capacity div 6` rows).

Working theory (not confirmed math): `BackpackFlags[5]` (`backpack_up0`..`backpack_up4`)
are five sequential capacity-upgrade tiers, each adding one 6-wide row (roughly
12 → 18 → 24 → 30 → 40). Only the 12 → 18 transition has been observed so far.

The trailing `Extra` field is the format's one real trap. It is present for
container items and for player **backpack** items, and absent for player **hotbar**
items. Getting this wrong shifts everything after the player inventory by two bytes
per item.

### List terminator

The list ends when the next `int32` reads 0. That zero is **not a field of its own** —
it is the first four bytes of a **zero pad** (5 or 9 bytes) that sits between the last
object and the spawner count. So a reader should peek the `TypeId`, and on zero rewind
and consume the pad, not 4 + pad. Getting this wrong puts the spawner count in the
middle of the pad. Early samples (A/B) use a **9-byte** pad; saves that contain
`burried_*` entries use a **5-byte** pad. Readers should try 9 then 5, validating that
player `SizeField = 56` follows the spawner list. In sample A the pad is
`0x0A59 .. 0x0A61` and the spawner count is at `0x0A62`.

---

## 5. Spawner lists — `0x0A62` in sample A (CONFIRMED)

One or more consecutive counted lists follow the zero pad. Readers keep reading
lists until the next `int32` is player `SizeField = 56` (optional trailing zeros
before that are preserved as post-pad).

```
int32 SpawnerCount
TRISpawner (default / animalspawner_* / empty name):
  PStr  Name           // 'animalspawner_371', or EMPTY as 01 00 (see section 1)
  int32 A
  int32 B              // respawn timer or population?

TRISpawner (name starts with 'burried', 'treasure', or 'chest'):
  PStr  Name           // 'burried_022', 'treasure_001', 'chest_islands_002' — name only, no A/B
```

Sample A has one list of 2 animal spawners. Later saves often have a `burried_*`,
`treasure_*`, or `chest_*` list (name-only entries) followed by another list of animal or
empty-name spawners.

---

## 6. Player entity — `0x0A9C` in sample A

The most complex block, and the one with the most residual uncertainty. Structure
(CONFIRMED — it parses end to end and lands exactly on the equipment count):

```
int32   SizeField        // 56
byte    Preamble[20]     // 1 + 4 + 15
byte    SixFlags[6]      // 01 01 01 01 01 01
byte    PreTransform[30]
float32 Pos1[3]          // == header position   CONFIRMED
byte    Matrix[60]       // 15 floats, orientation
float32 Pos2[3]          // == header position, second copy   CONFIRMED
byte    Misc[28]         // 7 floats, unidentified
byte    Stats[20]        // 5 floats  <-- STAT CANDIDATES, see below

// backpack: explicitly counted, items DO carry Extra
uint8   InvA             // 6
uint8   InvB             // 1
uint16  Count            // 4
uint16  InvFlags         // 0
TRIItem Backpack[Count]  // with Extra

// hotbar: NO count field found, items do NOT carry Extra
TRIItem Hotbar[]         // read by probing until a record fails to validate

// status effects: four counted lists, all empty when nothing is active
byte    TrailPre[10]     // 01 00 00 00 00 00 | selected-slot uint16 | 07 00
uint16  EffCount         //   repeated 4 times
  uint32  Id
  float32 Value
byte    TrailPost[6]     // first uint16 = InfectionActive — see § Infection
// when infected (InfectionActive = 1):
float32 InfectionSeverity
byte    InfectionPad[4]

uint16  EqU1             // 1
uint16  EqU2             // 0
uint16  EquipCount       // 5
TRIEquipSlot Equipment[EquipCount]

float32 PostStat
int32   UnlockCount
int32   UnlockIds[UnlockCount]  // ascending crafting / recipe ids
int32   Capacity         // backpack cell count (6-wide grid)   CONFIRMED field
byte    BackpackFlags[5] // backpack_up0..up4; THEORY: one 6-wide row per flag
```

### Unlock ids — observed but unmapped

`UnlockIds` are ascending integer recipe / crafting unlock identifiers. Meanings are
not yet mapped to recipe names. Newly observed ids (present in real saves, identity
unknown):

| Unlock id | Status |
| --- | --- |
| 15, 18, 24, 25, 28 | observed but not yet mapped |

Record further ids here as they appear; map them with isolated craft → quit →
`RISECmd fdiff` pairs the same way as progression bits.

**Pair audit:** retained `fdiff` unlock sections only show **multi-id** add/remove
batches across long play spans (same `_0` lineage files as above). There is **no**
archived single-craft before/after that maps one unlock id → one recipe name.
Across live `svr_state_0` / `_2` / `_4`, the unlock list and capacity / backpack
upgrade flags are currently **identical** (account-level shared), while inventories
and stats diverge — do not infer unlock mappings from cross-island diffs.

### Infection — `InfectionActive` / `InfectionSeverity` (CONFIRMED)

Previously `TrailPost[6]` was documented as an all-zero constant. It is not.

| Field | Type | Meaning |
| --- | --- | --- |
| `TrailPost` first `uint16` | `InfectionActive` | `0` = uninfected, `1` = infected |
| `InfectionSeverity` | `float32` | severity while active (≈0.9195 ≈ ~85% in-game UI) |
| `InfectionPad` | `byte[4]` | reserved / padding; zero in samples so far |

**CONFIRMED** via a matched infected → medkit-in-game → post-cure pair:

1. Infected save: `InfectionActive = 1`, `InfectionSeverity ≈ 0.9195`, pad zero.
2. Player used a medkit in-game and quit (no editor involvement).
3. Post-heal save: active flag returned to `0`, severity region zeroed; nothing else
   in the trailer was disturbed.

Two independent samples, one showing the field appear and correctly clear in response
to a controlled in-game action — this is confirmed, not theory.

**Pair audit:** the specific before/after filenames for that infection test were **not
retained** in the repo or current slot folder. A later `svr_state_4.stt_BKP` still
shows infection active (`idx@0x004 = 4`), which is consistent with per-island player
state but is **not** the documented cure pair. Reconfirm on one explicit suffix if
promoting related trailer fields further.

On disk the editor treats the 8-byte severity+pad block as present when
`InfectionActive ≠ 0` (and presence-tracks write-back so uninfected round-trips stay
byte-identical). Model fields always exist and default to `False` / `0.0`.

#### Why infected saves originally failed to open (`Implausible effect count 2560`)

The hotbar has **no count field**; it is probe-based (see below). Before the infection
block was modelled, that probe could walk past the real hotbar into the trailer:

- a denormal / near-zero float decoded from trailer bytes still passed a naive
  `Condition > 0` check;
- a byte from the severity region coincidentally looked like a plausible `PStr`
  length, so the probe accepted a fake “item” and desynchronised;
- the effect-list `uint16` counts were then read from garbage (e.g. `2560`), and
  `Validate`'s sanity guard correctly refused the parse.

This is a **general fragility of the probe-based hotbar reader**, not
infection-specific — any future data placed after the hotbar could trigger the same
misread class. Mitigation in the current parser: stop the probe when the next bytes
look like `TrailPre` (`… FF FF 07 00`), and reject near-zero conditions /
out-of-range slot–qty in `TryReadItemAt`.

### Two variable-length blocks that look fixed

Both of the blocks above were first modelled as fixed-size fields, and both
were wrong:

- The status-effect block is exactly 24 bytes when no effect is active, which
  is what every early sample had. It grows by 8 bytes per active effect.
- The unlock list held exactly one entry in the first save examined, so
  `UnlockCount` and `UnlockIds[0]` were read as two unrelated int32 fields
  named `PostA1` / `PostA2`. A save with 13 unlocks pushed `Capacity` 48 bytes
  out of position.

Measured spans between the end of the hotbar and the equipment block:

| span | L1 | L2 | L3 | L4 | records |
|---|---|---|---|---|---|
| 24 | 0 | 0 | 0 | 0 | none (no active effect) |
| 32 | 1 | 0 | 0 | 0 | `{id=0, 0.5}` — bleeding |
| 40 | 1 | 0 | 0 | 1 | `{id=0, 1.0}`, `{id=3, 0.3884}` |

**Neither error was caught by round-trip verification.** Bytes the reader does
not interpret land in the opaque tail and are written straight back, so a
block read at the wrong offset still reproduces the file byte for byte. The
misalignment only became visible as nonsense field *values* — `Capacity` of
1752461164 and backpack flags reading `2e776f6f6c`, which is ASCII `.wool`
from the middle of a `cloth.wool.*` string.

This is why `TRISaveFile.Validate` exists as a separate gate: it range-checks
fields with known-narrow values (equipment slots ascending from 0, capacity
1..1000, boolean upgrade flags, ascending unlock ids). Run both gates. Passing
round-trip alone does not mean the parse is right.

```
TRIEquipSlot:
  uint16  Slot           // 0 head, 1 torso, 2 legs, 3 feet, 4 suit
  PStr    ItemId         // bare slot = 01 00, NOT 00 - see section 1
  float32 Condition      // 1.0 for a bare slot
```

Sample A equipment: `cloth.wool.hat`, `cloth.wool.jacket`, `cloth.wool.pants`,
(empty), `cloth.radsuit`.

### The hotbar has no count field — how it is read

No plausible count precedes the hotbar array. It is read by **probing**: speculatively
parse a `TRIItem` without `Extra`, and accept it only if `Slot < 64`, `Quantity` is
sane, `Condition >= 0.001` (reject denormals), and `ItemId` is a well-formed
namespaced `PStr`. The probe also **stops when the next bytes look like
`TrailPre`** (`… FF FF 07 00`). On rejection the reader rewinds and the array ends.

This is still the weakest link in the model. A save whose next-field bytes happen to
look like a valid item can over-read — the infection-block failure mode above is the
worked example. Round-trip verification (§8) remains the safety net: a mis-probe shows
up as a byte difference and the editor refuses to write.

### Player stats — refined candidates (still THEORY unless noted)

Five floats immediately before the inventory header, plus `PostStat` after equipment.
Identities remain unconfirmed except where noted; the GUI labels them with the same
caveat.

| Index | Sample A | Current reading |
| --- | --- | --- |
| 0 | -27.119444 | **Not a stat bar** — duplicated negative values that move with index 1 over time; likely camera / world angle or heading |
| 1 | -27.119444 | same as index 0 |
| 2 | 0.310379 | **Hunger or thirst (candidate)** — trends downward with play time across 6 samples; activity-linked drain (near-constant sprinting) supports this |
| 3 | 0.10000000149011612 | **Static engine constant (CONFIRMED)** — identical across all samples; **not** a gameplay stat. Do not label as unknown or dynamic. |
| 4 | 1.000000 | **Strongest health candidate** — note: observed slightly above `1.0` in some samples (regen overshoot? unconfirmed) |
| `PostStat` | 0.602321 | **Fatigue / sleep-debt (candidate)** — observed negative when overdue for sleep and positive after sleeping in a controlled test, which rules out a simple 0..1 normalised bar |

Further isolation diffs (`RISECmd fdiff` on controlled before/after pairs) are still the
right way to harden the remaining candidate labels (not index 3).

---

## 7. Tail region — `0x0D4B` onward

Contains a repeating ~280-byte structure (3 instances, apparently difficulty or
creature tuning profiles), then a large zero-filled span, then a 40-byte footer at
`0x12A7` in sample A:

```
int32   2
int32   16
int32   0
int32   3
int32   1
float32 502.186     // play time again
float32 502.186     // and again
byte    pad[12]     // zero
```

Play time is therefore stored **four times**: header `0x00C`, once at `0x10A0`, and
twice in the footer. `TRISaveEditor.SetPlayTime` rewrites the header copy and then
scans the tail for float copies of the old value and updates those too, so an edit
does not leave the file internally inconsistent.

The exact boundary between the tuning profiles and the zero span is not nailed down —
`3 x 280` bytes from `0x0D4B` lands at `0x1093`, but the float `-1.0` at `0x1090`
suggests the true boundary is a few bytes earlier. **This does not matter for
correctness:** the whole region from the end of `BackpackFlags` to EOF is captured as
one opaque `TailBlob` and written back byte for byte. Nothing in the editor needs to
understand it.

---

## 8. Safety model — no checksum, so what protects the file?

There is no checksum to recompute, which removes the usual save-editor hazard but
introduces a subtler one: because the format has unknown spans and a probe-based
array, a parser bug corrupts silently.

The defence is **round-trip verification**, in `TRISaveFile.VerifyRoundTrip`:

1. Load the file, keeping the original bytes.
2. Immediately re-serialise the parsed model.
3. Compare against the original, byte for byte.

If the rebuilt image is not identical, the parser misread something and **the editor
refuses to write**. Every unknown span is stored verbatim precisely so that this
check can pass.

Both sample saves pass. `analysis/verify_model.py` mirrors the Delphi read/write
sequence statement for statement and asserts the same property, so the field order can
be re-checked without a Delphi compiler:

```
> python verify_model.py
..\samples\svr_state_0.A.stt
  size           4815      objects 25 (11 containers)   spawners 2
  player @ 0xA9C  backpack 4  hotbar 4  equipment 5  capacity 40
  opaque tail    1453 bytes
  ROUND TRIP     OK (byte identical)
..\samples\svr_state_0.B.stt
  size           4793      objects 24 (10 containers)   spawners 2
  ROUND TRIP     OK (byte identical)
ALL PASS
```

This check earned its keep immediately: it caught both the 9-byte terminator
miscount and the empty-`PStr` encoding bug described above, either of which would have
produced a corrupt save that looked fine in a field-by-field dump.

This turns "did I understand the format?" from a hope into a per-file assertion, and
it is the reason the tail region and the transform matrix were left uninterpreted
rather than guessed at.

`Validate` is a separate structural gate (equipment indices, capacity range, effect
list sanity, etc.). It does **not** flag repeated backpack `Slot` values — those are
expected once capacity exceeds one 6-wide row. Structural validity also does **not**
imply known-safe-to-load: see §10.

---

## 9. Offset summary for modification

| Target | Where | How to reach it | Confidence |
| --- | --- | --- | --- |
| play time | `0x00C` + 3 tail copies | fixed | CONFIRMED |
| player position | `0x21B` + entity `Pos1`, `Pos2` | fixed + parse | CONFIRMED |
| map / recipe / story unlocks | `0x010 .. 0x21A` bitfield | fixed; bits 0/352 permanent THEORY; bit 1 transient THEORY | region CONFIRMED |
| container contents | per-object item array | parse | CONFIRMED |
| backpack | player entity, counted array | parse | CONFIRMED |
| backpack `Slot` | column 0..5 in 6-wide grid; row = array order | parse | CONFIRMED (repeats expected) |
| hotbar | player entity, probed array | parse | CONFIRMED structure, fragile detection |
| infection | `TrailPost[0..1]` + optional severity block | parse | CONFIRMED |
| equipped clothing | player entity, `TRIEquipSlot[]` | parse | CONFIRMED; Condition 0-100 wear CONFIRMED |
| clothing durability | equipment / inventory `Condition` | parse | CONFIRMED dynamic 0-100 (e.g. shoes 59.0) |
| tool / melee durability | item `Condition` | parse | CONFIRMED maxima: stone 20 / iron 60 / Ti 100 / katana ~200 |
| special-variant Condition | e.g. `wep.flameclawclub` | parse | THEORY - not durability; effect float |
| unlock id list | player `UnlockIds[]` | parse | field CONFIRMED; ids 15/18/24/25/28 observed unmapped |
| backpack capacity | player entity `Capacity` | parse | CONFIRMED |
| backpack upgrade flags | `BackpackFlags[5]` / `backpack_up0..4` | parse | field CONFIRMED; row-per-flag THEORY |
| health / hunger / thirst / fatigue | entity `Stats[5]` + `PostStat` | parse | THEORY (index 3 = static constant CONFIRMED) |
| checksum | none exists | - | CONFIRMED |

---

## 10. Known limitation — backpack / hotbar record-count edits

**Changing the record *count* of the player backpack or hotbar arrays (adding or
removing whole item records) reliably breaks the game on load**, even when the file
passes `VerifyRoundTrip` and `Validate`. This is a known, reproducible limitation —
not an open mystery queued for more file-format speculation.

Observed failure modes (controlled tests):

| Edit | In-game result |
| --- | --- |
| Backpack grows | black screen; hotbar renders empty; world does not render |
| Backpack shrinks | loads; backpack correct; hotbar empty |
| Hotbar grows alone | hard crash to desktop at ~80% load |
| Hotbar shrinks alone | partial failure/errors at ~80%, then recoverable, but equipment wiped |
| Both grow together (matched counts) | loads fully but with graphical / colour corruption |

**Ruled out** (tested and eliminated): checksum, missing tail offset, item-catalogue
rejection, stale player `SizeField`. Root cause remains unknown and likely needs a
memory debugger / Cheat Engine on the running process rather than further `.stt`
analysis.

**Still fully safe:** same-count edits — quantity changes, in-place item/id swaps
(regardless of resulting byte-length change of the `PStr`), condition edits, stats,
flags, infection fields, equipment contents, world containers. The editor's
`CheckCountSafety` gate warns before writing if backpack or hotbar counts differ from
the values captured at load.
