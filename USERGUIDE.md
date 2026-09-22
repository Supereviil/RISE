# RISE User Guide

Radiation Island Save Editor — Windows utility for inspecting and editing `.stt` save files.

---

## 1. Overview

RISE reads, decodes, displays, and safely edits Radiation Island (Steam) save files (`.stt`).

It parses the binary save into a structured model (player, inventory, equipment, world objects, status effects, progression flags), shows that data in a VCL editor (`RISEGui`), and writes changes back only when structural checks allow. A companion console tool (`RISECmd`) provides verify, dump, and field-diff commands for the same format.

Edits operate on the parsed object graph, not raw offsets. Variable-length records are re-laid-out on write so string lengths and array sizes stay consistent.

---

## 2. Loading a Save File

### Where the game stores saves

Typical Steam path:

```
<SteamLibrary>\steamapps\common\RadiationIsland\save\<user-id>\000\svr_state_0.stt
```

Example:

```
E:\SteamLibrary\steamapps\common\RadiationIsland\save\68637065\000\svr_state_0.stt
```

- `<user-id>` is a per-account folder (numeric).
- `000` is the save slot. Sibling `_BKP` copies may also appear.
- Quit the game before editing. The game writes the save on exit and will overwrite your changes if it is still running.

### Multiple islands = multiple `.stt` files

Each unlocked island region is a **separate full save**, not extra objects inside one file:

| File | Typical role |
| --- | --- |
| `svr_state_0.stt` | First / northern island region |
| `svr_state_2.stt` | Broader / center region (many shared chests) |
| `svr_state_4.stt` | Later island (sparse world objects) |

When you open any `svr_state_<digits>.stt`, RISEGui also loads matching siblings in the same folder. The **World** tab lists each file as `[primary]` or `[island]` with its containers. Edits to a chest are written back to the file that owns it. **Save** writes the primary plus any dirty siblings (each gets a `.rise-bak`). **Save As** writes only the primary copy you choose.

The **Player** tab (inventory, stats, equipment, progression) edits the **primary** file you opened — that island’s own character snapshot. Sibling islands keep separate live player state; RISE does **not** copy inventory or stats across island files. Unlock lists and backpack capacity are typically identical across islands, but that is the game’s shared account-level data, not something the editor merges.

Backup / copy names (`*_BKP`, `*.rise-bak`, `svr_state_0_1.stt`, etc.) are ignored as siblings.

### Opening a save in RISE

1. Launch `RISEGui.exe`.
2. Click **Open** and select a `.stt` (or `.stt_BKP`) file.
3. Check the status bar after load (it reports how many island siblings were attached).

### Automatic parsing and validation

On open, RISE:

1. Parses the full file into the in-memory model.
2. Runs a **round-trip check**: re-serialises the model and compares it byte-for-byte with the original.
3. Runs structural **Validate** checks (capacity range, equipment slots, effect-list sanity, etc.).
4. Learns any new item ids from the save into the catalogue.

| Status | Meaning |
| --- | --- |
| Round trip OK | Safe to edit and save (subject to count-safety rules below). |
| Round trip FAILED | Parser misread something. Inspection is fine; **do not overwrite** the original. Use **Save As** only if you intentionally want a rebuilt copy for research. |

---

## 3. Editing Inventory

### Backpack vs hotbar

| Area | Role | Notes |
| --- | --- | --- |
| **Backpack** | Full inventory grid | Counted array. `Slot` is the **column** (`0`–`5`) in a fixed 6-wide grid; the row comes from array order. Repeated slot numbers across a large backpack are normal. |
| **Hotbar** | Quick-access bar (slots `0`–`5`) | Confirmed six-slot game UI. No count field on disk; RISE detects records by probing. Assign items from the backpack or set a slot directly. |

Capacity is the backpack cell count. Upgrade flags (`backpack_up0`…`up4`) relate to expanding that grid; treat capacity edits with care.

### How item stacks work

- Each inventory entry is a stack: item id (ASCII string, e.g. `food.bandage`, `ammo.arrow`), quantity, condition/durability, and slot.
- Stackable ids (ammo, bandages, resources, food) merge into an existing stack of the same id when added.
- Non-stackable gear takes its own record.
- Condition is meaningful for gathering tools (real durability). Many ranged/combat weapons show a constant `1.0` / `N/A` — they do not track wear the same way.
- Prefer **confirmed** catalogue ids (observed in real saves). Candidates from config guesses can be wrong (e.g. use `wep.fact.axe`, not `wep.factoryaxe`).

### Known stack limits (in-game)

These are the currently known gameplay caps. Keep quantities at or below them unless you are deliberately testing overflow behaviour:

| Item | Known max stack |
| --- | --- |
| Bandages (`food.bandage`) | **10** |
| Machinegun bullets (`ammo.machinegunbullet`) | **100** |
| Other ammo (`ammo.bullet`, `ammo.arrow`, `ammo.revolverbullet`, `ammo.riflebullet`, etc.) | **50** |

RISE clamps Add / Add to Hotbar / Max stacks to these ammo caps (and other known prefix caps) when editing. Oversized stacks already present in a loaded save are left untouched until you change that stack's quantity. Quantity is still a 16-bit field on disk.

**Important:** Changing backpack or hotbar **record count** (adding/removing whole stacks) is unsafe even when validation passes. Prefer quantity changes and in-place id swaps on existing records. See [Safety Features](#5-safety-features).

---

## 4. Player Stats & World Data

### Infection

On the **Status Effects** tab:

- **Infection active** — trailer flag (`0` / `1`). Confirmed: clears when cured in-game (e.g. medkit).
- **Infection severity** — `float32` present while infected (example: ≈ `0.92` ≈ ~85% on the in-game meter). Zero when uninfected / post-cure.

Use **Apply Infection** to write the checkbox and severity into the model. Clear both to remove active infection from the save.

### Health, hunger, thirst

Player scalars appear on the **Player** tab (five floats + post-equipment `PostStat`). Labels are best-effort candidates, not final names:

| Field | Current reading |
| --- | --- |
| Stat `[0]` / `[1]` | Not bar stats — duplicated negatives; likely camera / heading. |
| Stat `[2]` | Hunger or thirst candidate (drains with play / activity). |
| Stat `[3]` | Often constant `0.1` — may be a fixed setting, not a live bar. |
| Stat `[4]` | Strongest **health** candidate (sometimes slightly above `1.0`). |
| `PostStat` | Fatigue / sleep-debt candidate (can go negative when overdue for sleep). |

Edit carefully and verify in-game. For isolation testing, use `RISECmd fdiff` on before/after saves.

### Time and world state

- **Play time** — cumulative seconds (header + mirrored copies in the tail). Editable via the editor API / related UI; RISE updates mirrored copies so the file stays consistent.
- **Player position** — X/Y/Z on the Player tab; written to the header and entity copies.
- **World / Containers** — placed chests, deployables, and ground pickups across the opened file and any loaded island siblings; container item lists can be edited.
- **Progression flags** — bytes `0x010`…`0x21A` (map areas, recipes, story bits). Region is confirmed; most bit meanings are still unmapped. Flip one bit at a time and test in-game.

A dedicated day/night clock field is not exposed as a confirmed, labelled control. Do not assume play time equals in-world time of day.

---

## 5. Safety Features

Radiation Island saves have **no checksum**. Safety comes from parse fidelity and gated writes.

### Corruption prevention

- **Round-trip verify on load** — if rebuild ≠ original bytes, overwrite of the source file is blocked.
- **Opaque regions preserved** — unknown spans (including the large tail) are kept verbatim and written back unchanged.
- **Count-safety warning** — if backpack or hotbar stack *count* differs from load-time values, Save prompts before writing. Count changes reliably break load (black screen, empty hotbar, crash, or graphical corruption) even when the file looks structurally valid.

### Illegal value detection

`Validate` flags out-of-range or nonsensical fields, for example:

- Implausible effect counts / bad effect list indices
- Capacity outside a sane range
- Equipment slot / condition problems
- Probe rejection of garbage that would be misread as hotbar items

Status text reports **Structurally valid** vs **Known-safe to load** separately. Both matter.

### Quantity and stack hygiene

- Quantity entry is clamped to the valid `uint16` range (no silent wrap).
- **Max stacks** sets stackable backpack/hotbar quantities toward a high default (999), clamped per id via known stack limits (e.g. ammo caps above).
- Prefer editing quantity on existing stacks over inserting or deleting records.

### Backup creation

On **Save** to an existing path, RISE writes a sibling backup:

```
svr_state_0.stt.rise-bak
```

Keep your own copy of the whole `000\` folder as well. Restore from backup if the game rejects the file.

---

## 6. Saving & Exporting

### Writing changes

1. Confirm the status bar still shows a successful round trip from load (or re-check with **Verify** / `RISECmd verify` after heavy edits).
2. Click **Save** to overwrite the opened file (creates `.rise-bak` first), or **Save As…** to write a new `.stt`.
3. If count-safety warns, cancel unless you are deliberately reproducing a broken load for research.
4. Launch the game and load the slot. Change one category at a time so a bad load is easy to isolate.

### How structure is preserved

On write, RISE:

- Re-serialises from the object model in the confirmed field order.
- Rebuilds counted lists and trailer span (effects, infection block) from current data.
- Copies unknown/opaque blobs unchanged.
- Preserves exact empty-string encodings and other layout quirks required for byte-stable round trips.

Same-count edits (quantities, in-place item id swaps, conditions, stats, flags, infection, equipment contents, container contents) are the supported safe path. Growing or shrinking the backpack/hotbar arrays is not.

---

## 7. Troubleshooting

### Common parsing / open errors

| Symptom | Likely cause | Action |
| --- | --- | --- |
| Round trip failed at offset `0x…` | Parser/layout mismatch for this save | Do not overwrite. Inspect read-only; keep the original. Report the offset if analysing the format. |
| `Implausible effect count …` (or similar Validate failure) | Misaligned trailer / hotbar probe walked into following data | File may be damaged or from an unsupported layout. Restore backup; do not force-save. |
| Status shows structurally valid but not known-safe | Backpack/hotbar record count changed since load | Undo count changes, or accept that the game will likely fail on load. |
| Item missing / ignored in-game | Wrong or candidate-only item id | Use an id observed in a real save; restore backup and retry. |

### Game will not load a modified save

1. Quit the game completely.
2. Restore `svr_state_0.stt` from `*.rise-bak` or your folder backup.
3. Re-apply a smaller edit set:
   - Safe: quantities, conditions, equipment ids, infection clear/set, single flag/stat tweaks.
   - Unsafe: adding/removing backpack or hotbar **records** (even if RISE saved successfully).
4. Confirm with `RISECmd verify` that round trip still passes after your edit.
5. If the game still fails after a same-count edit, the item id or value is probably illegal for the runtime — revert that change.

### Quick reference — safe vs unsafe

| Safe (same record counts) | Unsafe (changes record counts) |
| --- | --- |
| Change stack quantities | Add a new backpack stack |
| Swap item id in place | Remove a backpack stack |
| Repair / set condition | Add or clear hotbar slots as whole records |
| Infection, stats, flags, position | Grow backpack and hotbar together “to match” |

---

## Related docs

| File | Audience |
| --- | --- |
| [README.md](README.md) | Build, CLI workflow, developer overview |
| [FORMAT.md](FORMAT.md) | Reverse-engineered `.stt` layout (confirmed vs theory) |
