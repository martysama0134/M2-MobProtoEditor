# M2 Mob Proto Editor

A single-file browser editor for Metin2 `mob_proto.txt` and `mob_names.txt`.
No build step, no server, no dependencies — open `index.html` in a browser and
you are running it.

- [Using the editor](#using-the-editor)
- [File format](#file-format)
- [Modifying the editor](#modifying-the-editor) — how to add/change struct
  columns, mob types, ranks, battle types, AI/race/immune flags, and locales
  (**agents: read this section before touching `index.html`**)

## Using the editor

### Loading files

- **↑ Load proto** — load a `mob_proto.txt` (tab-separated text format, first
  line is the header row).
- **↑ Load names** — load a `mob_names.txt` (`VNUM<TAB>LOCALE_NAME`, first
  line header). Names are matched to proto rows by vnum.
- **Sample** — restore the built-in demo mobs (useful for testing changes to
  the editor itself).

### Browsing and editing

- Search accepts a vnum (`591`), a vnum range (`100-200`), or a name substring
  (matches both LOCALE_NAME and NAME(K)).
- Filter by mob type with the dropdown; **⌀ untranslated** shows only rows
  with an empty LOCALE_NAME. The list shows type badge and level.
- Click a row to open the edit panel. Fields are grouped (Identity, Combat,
  Stats, Behavior, Race & Immunity, Rewards, Enchants, Resists, Skills,
  Special points, Misc). Enum fields are dropdowns, flag fields are toggle
  chips, everything else is a text/number input.
- **+ New** creates a mob at `max(vnum)+1` (defaults: MONSTER, PAWN).
  **Duplicate** clones the open mob to a new vnum. **Delete** removes it.

### Names, charsets, and byte preservation

Metin2 protos are legacy single-byte/EUC-KR encoded, so the editor works at
the byte level:

- **LOCALE_NAME charset** selects the codepage used to decode/encode
  `mob_names.txt` and the proto's ASCII columns (CP1250/1252/1253/1254/1256,
  or UTF-8 = anything goes).
- **NAME(K)** is the throwaway Korean source-name column; its charset toggles
  independently between CP949 and CP1252.
- **Byte preservation:** a name you never edit is exported with its *original
  bytes*, untouched — loading and re-exporting a file does not re-encode or
  corrupt anything. Only rows whose name you actually edited are re-encoded.
- A **⚠** marker means the current text contains characters not encodable in
  the selected codepage; they will be written as UTF-8 bytes on export (and
  counted in the export toast).

### Exporting

**↓ Export** downloads both files: `mob_proto.txt` (all rows, all columns)
and `mob_names.txt` (only rows that have a LOCALE_NAME).

## File format

`mob_proto.txt` is tab-separated with one header line. Columns are
**positional** — the editor reads and writes all 71 of them strictly in this
order:

| # | Header | Internal key | Editor field |
|---|--------|--------------|--------------|
| 0 | VNUM | `vnum` | number |
| 1 | NAME | `nameK` | text (byte-backed) |
| 2 | RANK | `rank` | enum (`RANKS`) |
| 3 | TYPE | `type` | enum (`MTYPES`) |
| 4 | BATTLE_TYPE | `battleType` | enum (`BTYPES`) |
| 5 | LEVEL | `level` | number |
| 6 | SIZE | `size` | enum (`SIZES`, empty allowed) |
| 7 | AI_FLAG | `aiFlag` | flags (`AIFLAGS`, comma-separated) |
| 8 | MOUNT_CAPACITY | `mountCapacity` | number |
| 9 | RACE_FLAG | `raceFlag` | flags (`RACEFLAGS`, comma-separated) |
| 10 | IMMUNE_FLAG | `immuneFlag` | flags (`IMMUNEFLAGS`, comma-separated) |
| 11 | EMPIRE | `empire` | number |
| 12 | FOLDER | `folder` | text |
| 13 | ON_CLICK | `onClick` | number |
| 14–17 | ST, DX, HT, IQ | `st` `dx` `ht` `iq` | number |
| 18–19 | DAMAGE_MIN/MAX | `dmgMin` `dmgMax` | number |
| 20 | MAX_HP | `maxHp` | number |
| 21–22 | REGEN_CYCLE, REGEN_PERCENT | `regenCycle` `regenPercent` | number |
| 23–24 | GOLD_MIN/MAX | `goldMin` `goldMax` | number |
| 25 | EXP | `exp` | number |
| 26 | DEF | `def` | number |
| 27–28 | ATTACK_SPEED, MOVE_SPEED | `atkSpeed` `moveSpeed` | number |
| 29–30 | AGGRESSIVE_HP_PCT, AGGRESSIVE_SIGHT | `aggrHpPct` `aggrSight` | number |
| 31 | ATTACK_RANGE | `atkRange` | number |
| 32 | DROP_ITEM | `dropItem` | number |
| 33 | RESURRECTION_VNUM | `resurrectVnum` | number |
| 34–39 | ENCHANT_CURSE/SLOW/POISON/STUN/CRITICAL/PENETRATE | `enchant*` | number |
| 40–50 | RESIST_SWORD/TWOHAND/DAGGER/BELL/FAN/BOW/FIRE/ELECT/MAGIC/WIND/POISON | `resist*` | number |
| 51 | DAM_MULTIPLY | `damMultiply` | text (float, e.g. `1.0`) |
| 52 | SUMMON | `summon` | number |
| 53 | DRAIN_SP | `drainSp` | number |
| 54 | MOB_COLOR | `mobColor` | number |
| 55 | POLYMORPH_ITEM | `polymorphItem` | number |
| 56–65 | SKILL_LEVEL0/SKILL_VNUM0 … SKILL_LEVEL4/SKILL_VNUM4 | `skillLevel0`… | number |
| 66–70 | SP_BERSERK/STONESKIN/GODSPEED/DEATHBLOW/REVIVE | `spBerserk`… | number |

`mob_names.txt` is `VNUM<TAB>LOCALE_NAME` with a header line.

Flag columns hold comma-separated names (`STUN,SLOW,TERROR`). All values are
kept as strings internally.

## Modifying the editor

### Architecture

The entire app is `index.html`:

- The `<x-dc>` block is the HTML template (dc-runtime templating: `{{ expr }}`
  interpolation, `<sc-for>` loops, `<sc-if>` conditionals).
- The `<script type="text/x-dc" data-dc-script>` block holds
  `class Component extends DCLogic` — all state, parsing, export, and the
  static tables that define the mob proto schema.

**`support.js` is a generated runtime bundle (dc-runtime). Never edit it.**

There is no build step. Edit `index.html`, reload the browser, click
**Sample** to test.

### Where everything is defined

All schema knowledge lives in static tables on `Component` in `index.html`:

| Table | What it defines |
|-------|-----------------|
| `COLS` | Internal field keys; **array order = column order in mob_proto.txt** |
| `HEADERS` | Header row written on export; **must stay index-aligned with `COLS`** |
| `DEFAULTS` | Default value per field for new/parsed rows |
| `FIELD` | Per-field editor metadata: `label`, `kind`, `options`, `sep`, `span` |
| `GROUPS` | Edit-panel sections; each lists the field keys it shows |
| `RANKS` | Options for `rank` (PAWN … KING) |
| `MTYPES` | Options for `type` (MONSTER, NPC, STONE, WARP, …) |
| `BTYPES` | Options for `battleType` (MELEE, RANGE, MAGIC, …) |
| `SIZES` | Options for `size` (empty, SMALL, MEDIUM, BIG) |
| `AIFLAGS` | Chip options for `aiFlag` (AGGR, NOMOVE, BERSERK, …) |
| `RACEFLAGS` | Chip options for `raceFlag` (ANIMAL, UNDEAD, ATT_FIRE, …) |
| `IMMUNEFLAGS` | Chip options for `immuneFlag` (STUN, SLOW, POISON, …) |
| `LOCALES` | `[code, label, codepage]` triples for the charset dropdown |
| `EXTRA` | Per-codepage string of encodable non-ASCII characters |
| `SAMPLE` | Built-in demo rows (they inherit `DEFAULTS` for missing keys) |

Field kinds in `FIELD`:

- `num` — text input with numeric input mode
- `text` — plain text input
- `enum` — dropdown; needs `options: Component.SOMETABLE`
- `flags` — toggle chips; needs `options`. `sep:','` makes the column
  comma-joined (`STUN,SLOW`); without `sep` it joins as `A | B`. All three
  mob flag fields use `sep:','`.
- `span: 2` — field takes the full panel width

### Add a proto column (new field)

Five places must change, and **`COLS`/`HEADERS` must get the new entry at the
same index**. Parsing and export are positional, so put the column where your
server's file actually has it — for a new trailing column, append to both.

Example — add a trailing `SUNGMA_ST` column:

1. `COLS` — append `'sungmaSt'`
2. `HEADERS` — append `'SUNGMA_ST'`
3. `DEFAULTS` — add `sungmaSt:'0'`
4. `FIELD` — add `sungmaSt:{label:'Sungma STR',kind:'num'}`
5. `GROUPS` — add `'sungmaSt'` to a group's `keys` (e.g. Stats), or it won't
   appear in the edit panel

Notes:

- Columns 0 (`vnum`) and 1 (`nameK`) are special-cased in `parseProto()` and
  `export()` (nameK is raw bytes). Insert new columns only at index ≥ 2.
- Files with *fewer* columns than `COLS` load fine — missing fields get
  `DEFAULTS`. Export always writes every column in `COLS`.
- `SAMPLE` does not need updating; sample rows inherit `DEFAULTS`.

### Add a mob type, rank, or battle type

Append the name to `MTYPES`, `RANKS`, or `BTYPES`. It appears in the dropdown
(and, for types, the list filter) immediately. Optional: give a new type a
list-badge color in `typeColor()`.

### Add an AI / race / immune flag

Append to `AIFLAGS`, `RACEFLAGS`, or `IMMUNEFLAGS`. Chips render
automatically; toggled flags are exported comma-joined in the order of the
options array. Use the exact enum name your server source uses — the text
proto stores the name, not the number.

### Add a locale / codepage

1. Add `['xx','Language (CPnnnn)','nnnn']` to `LOCALES`.
2. If the codepage is new to the editor:
   - add an `EXTRA['nnnn']` string listing every encodable non-ASCII character
     (used for the ⚠ encodability check),
   - add `'nnnn':'windows-nnnn'` to the map inside `decodeName()`.
3. Codepages whose repertoire can't be listed as a string (CJK etc.) get range
   logic in `encChar()` instead — see the CP949 and CP1256 branches there.

### Gotchas

- **Index alignment.** A `COLS`/`HEADERS` mismatch silently shifts every
  exported column. Count twice — there are 71.
- **Flag separators.** Mob flag columns are comma-separated (`sep:','` in
  `FIELD`); `toggleFlag()` and the chip renderer both read `meta.sep`. Don't
  hardcode `|` anywhere.
- **Byte-backed names.** `nameK` and `locale` keep original file bytes
  (`bytes` / `localeBytes`) until edited; editing sets the string and nulls
  the bytes for that row (see `update()`). Don't "normalize" this — it is the
  no-corruption guarantee.
- **Everything is a string.** Row values are strings even for numeric fields
  (`damMultiply` is a float string like `1.0`); comparisons/parsing use
  `parseInt` where needed.
- **Static init order.** `FIELD` references `Component.RANKS` etc. at
  class-init time — option arrays must be declared before `FIELD`.
- **`support.js` is generated.** Repeat: never edit it.

### Testing a change

1. Open `index.html` in a browser (or reload).
2. Click **Sample**, open a mob, check your field/option renders and edits.
3. **Export** and inspect the downloaded files — header count, column count,
   and tab positions must match your server's expectations.
4. Load a real `mob_proto.txt`, spot-check a few rows, re-export, diff.
