---
name: ritual-wanted
description: "Manage and price a Magic: The Gathering wanted list (cards to acquire) with Ritual. Use when the user wants to track cards they want to buy, add cards to a wishlist, record a purchase by moving a wanted card into a collection, import a wanted list from a CSV or text file, or price a wanted list."
ritual-version: 0.1.0-beta25
ritual-content-hash: 22da57efcdba4c80e5f9f306004b3291b04d85a400d4d9e7a5b6152030c46b10
---

# Managing wanted lists with Ritual

Wanted lists (cards to acquire) live in `wanted/<name>.md`. Entries may be
name-only — no printing required. See the **ritual** skill for the file format.

## One-shot edits (non-interactive — best for agents)

Use the one-shot commands (covered in full by the **ritual-edit** skill).
`add-card` works on wanted lists, and — when the type is pinned with `--wanted`
or a `wanted:` prefix — creates the list if it does not exist yet, including in a
workspace with no wanted lists at all (the file is created only once the add is
certain to succeed, so a failed add or a `--dry-run` leaves none behind). Every wanted
add chooses a specificity: `--name-only` (any copy), a specific printing via
`--set`/`--collector-number`, or `--specific` (open the interactive printing
picker; needs a terminal). A terminal prompts when none is given; a
non-interactive run without one exits 2. A specific-printing add also needs
`-f` when that printing comes in several finishes — headless runs never guess
one:

```bash
ritual add-card "To Buy" "Mox Ruby" --wanted --name-only
ritual add-card "To Buy" "Lightning Bolt" --wanted --name-only -f foil   # -f finish (optional)
ritual add-card "To Buy" "Demonic Tutor" --wanted --set sta --collector-number 90 -f nonfoil
ritual remove-card "To Buy" "Mox Ruby" --wanted              # one entry
ritual set-card "To Buy" "Lightning Bolt" --wanted --finish foil
ritual note "To Buy" "Mox Ruby" --wanted -n "budget copy only"
```

## Bought a card? Move it to a collection

`ritual move` with `--from`/`--to` records a purchase in one command: the entry
leaves the wanted list and lands in the collection. Collections need a concrete
printing, so a name-only entry takes it from `--set`/`--collector-number` (or its
single known printing in the local card cache — a card the cache does not hold has
no printing list, so pass the flags); already-pinned entries move as-is:

```bash
ritual move "Demonic Tutor" --from "wanted:To Buy" --to "collection:Main Binder" \
  --set sta --collector-number 90
```

## Plan purchases with a diff

`ritual diff` compares a wanted list against a collection: entries only in the
wanted list are still unowned, matches are already covered (mismatched
quantities are listed separately). `--by name` (the default) matches card names;
`--by printing` requires the exact set/collector-number/finish to match:

```bash
ritual diff wanted:to-buy "collection:Main Binder"
ritual diff wanted:to-buy "collection:Main Binder" --by printing --output json
```

## Interactive management

`ritual edit` opens the interactive editor (covered in full by the
**ritual-edit** skill); pick a wanted list (or `➕ New Wanted List`) from its
list selection menu. It **requires a terminal**, so it is not suitable for
non-interactive agents — use the one-shot commands instead.

```bash
ritual edit
ritual edit "wanted:To Buy"          # open one list directly (matches the file basename)
ritual edit --sets "FDN,SPG"         # restrict to these set codes
ritual edit --finish foil
ritual edit --collector              # enter cards by collector number
ritual edit --allow-digital-only-cards
ritual edit --refresh never          # use the existing cache as-is, no prompt
ritual edit --refresh auto           # redownload the cache when prices are >1 day old
```

The shared `--refresh <mode>` option controls card-cache freshness: under `ask`
(the default) a cache last fully downloaded more than a week ago prompts to
redownload before the session starts; `auto` redownloads without prompting when
the cached prices are more than a day old; `no-bulk` and `never` use the
existing cache as-is.

**Saving:** changes accumulate **in memory** — `💾 Save` writes the file and
changelog without exiting. Backing out (`🔀 Switch List` or Esc) returns to the
list selection menu **keeping unsaved changes in memory**, so edits can span
several lists before one save — Save flushes every open list, and a separate `💾
Save current list changes` item appears when more than one open list has unsaved
changes. `🚪 Exit` with anything unsaved opens an exit menu: save and exit, exit
without saving (discards everything unsaved), or cancel to keep editing. Saving
more than once in one session folds the later changes into that list's existing
changelog entry (bumping its timestamp) — each saved list gets exactly one
changelog entry per session.

**Edit mode:** `🛠️ Switch to Edit Mode` turns the search prompt into a picker
over the list's existing entries — change a card's printing (or make it
name-only), finish, or note, or remove it — and `↩️ Undo Last Edit` reverts the
latest edit. The `✨ Change Finish` item is hidden for name-only entries — a
finish only annotates a specific printing.

**Undo within the session:** `↩️ Undo Last Add` removes the most recent card,
and `📋 View Session Changes` opens a picker over every change made this session
— adds, edits, and removals — where selecting one offers to discard just that
change (same-card changes must be discarded newest-first). Discarding an add
frees that card's `&N` id and keeps the remaining session ids dense (each later
card slides down one).

## Import from a text file

`import` turns a decklist-style text file into a new wanted list (quantities
expand to one bullet line per copy):

```bash
ritual import wants.txt --type wanted
ritual import wants.txt --type wanted --overwrite --no-input
```

Without `--type` an interactive run prompts for the list type; under the global
`--no-input` flag the type defaults to a deck, so agents should always pass
`--type wanted`. A body line that is neither a section header nor a card line is
skipped, listed on stderr, carried in the JSON `warnings` array, and exits 1 —
the import is still written. Ritual's own format and the MTG Arena/MTGO export
dialect (`4 Lightning Bolt (M10) 146`, bare `Deck`/`Sideboard` markers, a
trailing `*F*`/`*E*` foil marker) are both read, as is the inside of a ``` fence
— on the import path only, since a pasted decklist usually arrives wrapped in
one. A `(SET)` with no collector number is left in the card name rather than
read as a printing; a line that imports but still holds a printing token in its
name prints an advisory (stderr, JSON `advisories`) without failing the run.

`--dry-run` resolves and validates everything but leaves the workspace
byte-for-byte untouched — it does not even create the `decks/`, `collections/`,
or `wanted/` directory it would have written into.

## Import from a CSV file

A `.csv` source makes `import` import a CSV file into a new wanted list, or
append to an existing one (`--csv` forces CSV parsing for other extensions).
Non-interactive agents must pass all flags (running it bare opens an interactive
column-mapping wizard):

```bash
ritual import wants.csv --type wanted --name "To Buy" \
  --columns "name=1,set=2,collector-number=3,finish=4,quantity=5"
ritual import more.csv --type wanted --name "To Buy" --append --columns "name=1"
```

`--columns` maps fields to 1-based column numbers (fields: `name`, `set`,
`collector-number`, `condition`, `finish`, `section`, `quantity`); only `name`
is required and wanted lists carry no `condition` column. Add `--no-header` when
the first row is data — a scripted run without it drops the first row as a
header and warns when that row looks like data. Add `--overwrite` to replace an
existing wanted list, or `--append` to add to one (appends continue card IDs and
record the changelog). Rows naming the same card and printing merge into one
line (create and append agree), and a `--columns` number the file has no column
for is a usage error (exit 2) instead of a per-row failure. Failed rows are
reported with line numbers on stderr and the rest still import (exit code 1 on
partial failure).

## Price

The unified `price` command covers all list types; scope it with `--wanted` or a
name. An interactive browser opens on a TTY — for agents, always pass
`--summary`, `--output json`, or the global `--no-input` flag:

```bash
ritual price --wanted --summary                # every wanted list's totals
ritual price to-buy --no-input                 # one list's cards + totals
ritual price to-buy --output json --quiet
ritual price to-buy --sort price --descending --no-input
ritual price to-buy --prices eur               # usd | eur | tix (defaults to config defaultCurrency)
```

Each wanted list also reports a "lowest" total: name-only entries use the cheapest
printing, printing-pinned entries the cheapest finish of that printing, and
fully-specified entries their exact price.
