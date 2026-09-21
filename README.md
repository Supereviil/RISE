# RISE - Radiation Island Save Editor
<img width="615" height="518" alt="logo" src="https://github.com/user-attachments/assets/e262d6da-29cc-4b9d-90b5-11b0c7eb6e12" />

**RISE** is a Windows save editor and reverse-engineering toolkit for the Steam version of **Radiation Island**.

Built from a ground-up reverse engineering of the game's `.stt` save format, RISE provides a graphical interface for inspecting and editing inventory, equipment, player data, status effects, world containers, and progression flags while preserving the parts of the save that are not yet fully understood.

RISE was designed around one rule: **don't blindly write to a save file.** Every loaded save is parsed, rebuilt, and checked against the original before normal saving is allowed.

> **Current status: Beta**
>
> The save format has been extensively reverse engineered and tested, but Radiation Island contains data and progression fields that are still being mapped. Always keep a backup of your save.

---

## Features

### Inventory & Hotbar

- View and edit backpack inventory
- Change item quantities
- Add and remove items
- Assign backpack items to hotbar slots
- Edit the game's six hotbar slots (`0-5`)
- Max stack quantities using known in-game stack limits
- Repair damaged items
- Browse confirmed item IDs
- Optionally expose candidate/unconfirmed item IDs discovered during reverse engineering
- Friendly item names alongside the game's internal IDs
- Preserves oversized stacks already present in a save until that stack is edited.

### Equipment

- View all equipped clothing
- Equip and unequip items
- Repair equipped items
- Edit head, torso, legs, feet, and suit slots
- Preserve item-specific condition and durability behavior

### Player

- View and edit player position
- Inspect player scalar values
- Move the player by editing X/Y/Z coordinates
- Work with partially mapped player-state data without pretending unknown fields are confirmed

### Status Effects

- Inspect status-effect lists
- Add or remove effects
- Enable or clear infection
- Edit infection severity

Some effect IDs remain unidentified and are intentionally presented as such.

### World / Containers

RISE parses the game's world object graph and exposes inventory containers directly.

- Browse world containers and their contents
- Inspect chests, safehouse storage, and other inventory containers
- Add items directly to a selected container
- Browse objects across unlocked island save files
- Preserve each island's world data separately

When sibling `svr_state_<n>.stt` files are present, RISE can load them alongside the primary save so their containers can be inspected and edited without manually opening each file.

### Progression Flags

RISE exposes the game's progression bitfield used for data such as:

- Map unlocks
- Story progress
- Tasks
- Recipes
- Other progression state

Known flags are labeled where possible. Most individual bit meanings are still being mapped, so progression editing should be treated as experimental.

---

## Safety First

Radiation Island does not use a checksum for its save files, but that does **not** mean arbitrary edits are safe.

RISE uses several layers of protection.

### Byte-exact round-trip verification

When a save is opened, RISE parses it and immediately serializes the resulting model back to memory.

The rebuilt data is compared **byte-for-byte** against the original file.

If an untouched save cannot round-trip correctly, RISE treats that as evidence that the parser does not fully understand that particular file and blocks normal overwrite saving.

### Structural validation

Before saving, RISE checks the edited model for known structural problems such as invalid ranges, malformed effect data, equipment problems, and other conditions that may indicate a damaged or incorrectly parsed save.

### Unknown data preservation

Parts of the Radiation Island save format are still unidentified.

RISE preserves unknown and opaque regions rather than normalizing, deleting, or guessing their purpose.

### Automatic backups

Normal **Save** operations create a backup alongside the original file:

```text
svr_state_0.stt.rise-bak
```

Keeping your own backup of the entire save-slot folder is still strongly recommended.

---

## Important Limitation: Inventory Record Counts

Changing an existing item's data is substantially safer than changing the number of inventory records.

Confirmed safe editing includes operations such as:

- Changing stack quantities
- Swapping an item ID in an existing record
- Repairing item condition
- Editing equipment
- Editing infection/status data
- Editing player values
- Editing progression flags
- Editing existing world-container contents

**Changing the number of backpack or hotbar records can cause Radiation Island to reject the save, display an empty hotbar, produce graphical corruption, or crash.**

RISE detects backpack/hotbar count changes and warns before writing them.

Until the game's internal handling of these arrays is fully understood, prefer modifying existing records instead of growing or shrinking them.

---

## Using RISE

1. **Exit Radiation Island completely.**
2. Back up your save folder.
3. Launch RISE.
4. Click **Open...** and select your `svr_state_*.stt` save.
5. Check the status bar to make sure the file is structurally valid and known-safe to load.
6. Make your changes.
7. Use **Verify** if you want to inspect the current model's validation state.
8. Click **Save**.
9. Launch Radiation Island and test the edited save.

Making a small number of changes at a time is recommended, especially when experimenting with fields that are not yet fully mapped.

---

## Save Location

A typical Steam installation stores Radiation Island saves under:

```text
<SteamLibrary>\steamapps\common\RadiationIsland\save\<user-id>\000\
```

The primary save is normally:

```text
svr_state_0.stt
```

Additional unlocked islands may use files such as:

```text
svr_state_2.stt
svr_state_4.stt
```

Each island file contains its own world and player snapshot. RISE does not blindly merge player state between them.

---

## Confirmed vs. Experimental Data

Reverse engineering an undocumented binary format means some discoveries are more certain than others.

RISE deliberately distinguishes between:

- **Confirmed** behavior observed in real save files and tested in-game
- **Theory / experimental** fields whose location or purpose is not yet fully established

RISE does not intentionally present a guess as a confirmed fact.

For the technical details, including field layouts, parser behavior, known structures, theories, and previous hypotheses that were disproven during development, see **[FORMAT.md](FORMAT.md)**.

---

## Item IDs

Radiation Island stores item IDs as namespaced strings rather than numeric indexes.

Examples:

```text
ammo.revolverbullet
ammo.machinegunbullet
wep.bow
wep.fact.axe
cloth.radsuit
res.flint
```

RISE maintains two groups of item IDs:

**Confirmed IDs** have been observed in real save files.

**Candidate IDs** were discovered or inferred during reverse engineering but have not necessarily been confirmed to work in-game.

The editor defaults to confirmed IDs. Candidate IDs can be displayed for experimentation.

Using an invalid item ID may cause the game to reject or ignore the item, so confirmed IDs are recommended for normal use.

---

## Reverse Engineering

RISE exists because Radiation Island's save format was undocumented.

The format was reverse engineered from raw save files and found to be a byte-packed, little-endian serialization of the game's object state. It is not compressed or encrypted and contains variable-length records, strings, world objects, inventory structures, player state, progression data, and opaque regions.

The resulting format documentation is included in this repository:

### [FORMAT.md](FORMAT.md)

`FORMAT.md` documents the current understanding of the `.stt` format, including:

- Binary layout
- String encoding
- Inventory records
- Equipment
- World objects and containers
- Player data
- Status effects
- Infection
- Progression flags
- Multi-island saves
- Parser traps
- Confirmed findings vs. working theories

The documentation is being published both to explain how RISE works and to give anyone interested in Radiation Island modding a starting point that did not previously exist.

---

## Help Improve RISE

Radiation Island has many possible save states, and more samples mean more of the format can be confirmed.

I'm particularly interested in saves containing:

- Active infection
- Radiation/status effects
- Late-game progression
- Different unlocked islands
- Unusual inventory or equipment
- Progression states not yet represented in the current samples

If you discover an item ID, progression flag, status effect, or other field that RISE does not currently recognize, please open an issue or provide before/after save samples when possible.

Controlled before/after saves are especially useful: perform **one action in-game**, save, and provide the files from immediately before and after that action.

---

## Reporting Problems

If RISE opens a valid Radiation Island save but:

- Round-trip verification fails
- Structural validation fails
- An edit causes the game to reject the save
- A confirmed item behaves incorrectly
- A world container is parsed incorrectly
- You discover a new save layout

please open a GitHub issue with as much detail as possible.

If possible, include the original unmodified save that reproduces the problem.

**Do not upload saves containing information you do not want to share publicly.**

---

## Screenshots

<img width="1040" height="720" alt="01" src="https://github.com/user-attachments/assets/cba03e9c-62be-4dd1-979c-5872c3da4060" />
<img width="1040" height="902" alt="02" src="https://github.com/user-attachments/assets/d9821fab-7c0b-43f4-9322-d549dd18df9d" />
<img width="1040" height="741" alt="03" src="https://github.com/user-attachments/assets/879bff95-2813-454d-b336-460da641710d" />
<img width="1040" height="741" alt="04" src="https://github.com/user-attachments/assets/765a525c-36b2-48c4-96cf-40be8c768d12" />
<img width="1040" height="741" alt="05" src="https://github.com/user-attachments/assets/bdfee057-bed2-4bde-a5e2-1b47e3dd71db" />
<img width="1040" height="942" alt="06" src="https://github.com/user-attachments/assets/96510a11-8e73-42d3-8fff-41ddc4080045" />
<img width="1026" height="713" alt="07" src="https://github.com/user-attachments/assets/9088301c-ffae-4e2a-ae0e-23e89bfc245f" />


---

## Compatibility

RISE is currently designed for:

- **Windows**
- **Radiation Island**
- **Steam version**
- Save format **version 6**

Other releases or save-format versions have not been confirmed.

---

## Documentation

- **[User Guide](USERGUIDE.md)** - Detailed instructions for using RISE
- **[Save Format Documentation](FORMAT.md)** - Reverse-engineered Radiation Island save format

---

## Disclaimer

RISE is an unofficial community project and is not affiliated with or endorsed by Atypical Games.

Editing save files always carries some risk. Keep backups and test modified saves before continuing normal gameplay.

---

## Credits

**RISE - Radiation Island Save Editor**

Created by **Superevil Enterprises**

Reverse engineering, development, testing, and documentation by **Superevil**

© 2026 Superevil Enterprises
