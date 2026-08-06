---
name: ritual-decks
description: "Create, build, import, sync, and price Magic: The Gathering decks with Ritual. Use when the user wants to make a new deck, interactively build a deck by adding cards to sections, import a decklist from Archidekt, Moxfield, or MTGGoldfish, import a deck from a CSV file, pull or push changes to Archidekt, extract a deck primer, or price a deck."
ritual-version: 0.1.0-beta24-dev.b5ce8c1
ritual-content-hash: 784545bc402355dade3989dae3fa07cfbc6d4bfbb7c2370a66dc5864a5aced9a
---

# Managing decks with Ritual

Decks live in `decks/<name>.md`. See the **ritual** skill for the file format and
the **ritual-edit** skill for adding/removing individual cards.

## Create

```bash
ritual new deck "Winota Stax"                 # defaults to commander
ritual new deck "Mono-Red Aggro" -f standard  # -f / --format
```

Renaming and deleting decks (`ritual rename`, `ritual delete`) are covered in
the **ritual** skill.

### Deck format

`--format` takes one of: `commander`, `oathbreaker`, `standard`, `modern`,
`pioneer`, `legacy`, `vintage`, `pauper`, `historic`, `alchemy`, `explorer`,
`timeless`, `penny-dreadful`, `brawl`, `historic-brawl`, `duel-commander`,
`pauper-commander`, `pre-dh`, `pre-modern`, `limited`. Common aliases are accepted
and normalized (`EDH` → `commander`, `premodern` → `pre-modern`); anything else is
an error.

The format is stored as `format:` in the deck's front matter. A deck that declares
none is inferred from its sections — an `## Oathbreaker` or `## Signature Spell`
section means Oathbreaker (checked first), and a command-zone section such as
`## Commander` means Commander — and that inference is written into the file on
its next save, so do not add a `format:` by hand to "fix" a deck that displays
correctly.

The rest of the deck's front matter (`description`, `tags`, `format`, and the
`sourceId`/`sourceUrl` sync link) is scripted with `ritual metadata` — a
front-matter-only write that never touches card lines and records no changelog:

```bash
ritual metadata set my-deck description "A budget mono-red burn list"
ritual metadata set my-deck tags aggro budget     # replace; --add / --remove merge
ritual metadata set my-deck format modern
ritual metadata list my-deck                      # every field, (unset) included
ritual metadata unset my-deck description
```

## One-shot edits (non-interactive — best for agents)

Use the one-shot commands (covered in full by the **ritual-edit** skill) to edit an
existing deck without a TUI:

```bash
ritual add-card "Winota Stax" "Sol Ring" --deck
ritual add-card "Winota Stax" "Lightning Bolt" --deck -q 4         # -q quantity
ritual add-card "Winota Stax" "Kenrith, the Returned King" --deck --commander
ritual add-card "Winota Stax" "Pyroblast" --deck --section Sideboard -f foil
ritual remove-card "Winota Stax" "Lightning Bolt" --deck -q 2      # or --all-copies
ritual set-card "Winota Stax" "Sol Ring" --deck --section Sideboard
ritual set-card "Winota Stax" "Winota, Joiner of Forces" --deck --commander
ritual note "Winota Stax" "Sol Ring" --deck -n "fast mana"         # or --clear
ritual move "Lightning Bolt" --from "deck:Winota Stax" --to deck:burn
```

Deck adds merge onto an existing line for the same card and printing (never a
duplicate line, and the merged line keeps its `&N` and any `{note}`) and append
new lines at the end of the deck's first regular section unless
`--section`/`--commander` says otherwise. Merging wins over placement: when the
deck already runs the printing, the copies join that line where it is, and
`--commander` then moves the whole line into the Commander section. Add `-n`/`--dry-run`
to any one-shot command to preview it without writing anything.

## Build interactively

`ritual edit` opens the interactive editor (covered in full by the
**ritual-edit** skill); pick a deck (or `➕ New Deck`, which prompts for a
format) from its list selection menu, then add cards to named `## Section`
headers with name/collector entry modes and session filters (`-s/--sets`,
`-f/--finish`, `-c/--condition`) plus section targeting and a `🏷️ Change
Format` action. It **requires a terminal**, so it is not suitable for
non-interactive agents — use the one-shot commands instead.

```bash
ritual edit                                   # pick a deck, prompt for a section per card
ritual edit "Winota Stax"                     # open one deck directly (matches the file basename)
ritual edit --section Sideboard               # add every deck card to one section
ritual edit --collector --sets "FDN, SPG"     # collector-number entry, sets preloaded
ritual edit --refresh never                   # use the existing cache as-is, no prompt
ritual edit --refresh auto                    # redownload the cache when prices are >1 day old
```

The shared `--refresh <mode>` option controls card-cache freshness: under `ask`
(the default) a cache last fully downloaded more than a week ago prompts to
redownload before the session starts; `auto` redownloads without prompting when
the cached prices are more than a day old; `no-bulk` and `never` use the
existing cache as-is.

Set the **target section** to a fixed section or "prompt every time" via `--section`,
the `🗂️ Set Target Section` menu, or the session filters. Adding a card whose printing
already exists in the deck increments its quantity instead of duplicating the line.

**Saving:** changes accumulate **in memory** — `💾 Save` writes the deck file
and changelog without exiting. Backing out (`🔀 Switch List` or Esc) returns to
the list selection menu **keeping unsaved changes in memory**, so edits can span
several lists before one save — Save flushes every open list, and a separate `💾
Save current list changes` item appears when more than one open list has unsaved
changes. `🚪 Exit` with anything unsaved opens an exit menu: save and exit, exit
without saving (discards everything unsaved), or cancel to keep editing. Saving
more than once in one session folds the later changes into that list's existing
changelog entry (bumping its timestamp) — each saved list gets exactly one
changelog entry per session.

**Edit mode:** `🛠️ Switch to Edit Mode` turns the search prompt into a picker
over the deck's existing lines — change a line's printing, add/remove copies,
move it to another section, edit its note, or remove it entirely — and `↩️ Undo
Last Edit` reverts the latest edit.

**Undo within the session:** `↩️ Undo Last Add` takes back the most recent card,
and `📋 View Session Changes` opens a picker over every change made this session
— copy adds, field edits, and removals — where selecting one offers to discard
just that change (same-line changes must be discarded newest-first). Discarding
an add decrements or removes the line; a fully removed session line frees its
`&N` id and keeps the remaining session ids dense.

## Import from a URL or text file

```bash
# Archidekt, Moxfield, or MTGGoldfish URL, or a local decklist file
ritual import https://archidekt.com/decks/123456
ritual import ./my-decklist.txt --type deck
ritual import <url> --overwrite          # replace an existing deck of the same name
ritual import <url> --dry-run            # preview without writing files
ritual import <url> --no-input           # never prompt (fail if input is required)
```

URLs always import decks. A text file import prompts for the list type (deck,
collection, or wanted list) unless `--type` is passed; under the global
`--no-input` flag a run without `--type` defaults to a deck.

Archidekt and Moxfield imports keep each card's printing (set, collector number,
and foil/etched finish) exactly as the source states it; MTGGoldfish carries no
printing data, so those stay name-only.

Text imports read Ritual's own format and the MTG Arena/MTGO export dialect —
`4 Lightning Bolt (M10) 146` lines plus bare `Deck`/`Sideboard`/`Commander`/
`Companion`/`About` markers, a trailing `*F*`/`*E*` foil marker, and the
inside of a ``` fence (a decklist pasted from Discord or GitHub arrives wrapped
in one, so on the import path — and only there — the fence is packaging, not
prose). A `(SET)` with no collector number is **not** read as a printing: half a
printing cannot be written to a card line, and `Very Cryptic Command (Untap)` is
a real card name, so the name is kept verbatim and an advisory is printed.
Lines the parser cannot read are skipped and reported (exit 1); lines that import
but look wrong (a name still holding a printing token) print an advisory on
stderr and appear in the JSON `advisories` array without changing the exit code.

`--moxfield-user-agent` applies to URL imports only — passing it with a CSV or
text-file source is a usage error (exit 2).

`--dry-run` resolves and validates everything but leaves the workspace
byte-for-byte untouched — it does not even create the `decks/`, `collections/`,
or `wanted/` directory it would have written into.

Moxfield imports need a unique User-Agent: pass `--moxfield-user-agent
"you@example.com"` or set `MOXFIELD_USER_AGENT`.

## Import from a CSV file

A `.csv` source makes `import` import a CSV export into a new deck, or append to
an existing one (`--csv` forces CSV parsing for other extensions).
Non-interactive agents must pass all flags (running it bare opens an interactive
column-mapping wizard):

```bash
ritual import burn.csv --type deck --name "Burn" --deck-format modern \
  --columns "quantity=1,name=2,section=3"
ritual import more.csv --type deck --name "Burn" --append \
  --columns "quantity=1,name=2"          # merge into existing lines
```

`--columns` maps fields to 1-based column numbers (fields: `name`, `set`,
`collector-number`, `condition`, `finish`, `section`, `quantity`); only `name`
is required for decks. Add `--no-header` when the first row is data — a scripted
run without it drops the first row as a header and warns when that row looks
like data. Add `--overwrite` to replace an existing deck, or `--append` to add
to one (appends merge identical printings, continue card IDs, and record the
changelog). Conditions/finishes/sections are normalized (e.g. `Near Mint` →
`NM`, `F` → foil, `side` → `Sideboard`). `--deck-format` applies only when
creating a deck — passing it with `--append` is a usage error. Rows naming the
same card and printing merge into one line (create and append agree), and a
`--columns` number the file has no column for is a usage error (exit 2) instead
of a per-row failure. Failed rows are reported with line numbers on stderr and
the rest still import (exit code 1 on partial failure).

## Import an entire Archidekt account

```bash
ritual import-account someuser            # interactively pick decks
ritual import-account someuser --all      # import every deck
ritual import-account --all               # use the logged-in account
ritual import-account someuser --all --output json --quiet   # structured result
```

Deck selection is a prompt, so `--all` is mandatory for an agent: without a
terminal (or under `--no-input`) the run exits 2 before fetching anything.
Existing decks conflict unless `--overwrite`/`--yes` says what to do. The whole
account is fetched (every page), and `--output json` reports
`found`/`selected`/`imported`/`failed`/`skipped` plus a per-deck array.
A username with no results is reported honestly: Archidekt cannot distinguish an
unknown user from an account with no public decks, so check the spelling.

## Sync with Archidekt

`deck-sync` has four subcommands — `pull` (Archidekt → local), `push`
(local → Archidekt), `link`, and `status`; anything else exits with code 2:

```bash
ritual deck-sync pull                        # pull remote changes for all linked decks
ritual deck-sync push "Winota Stax"          # push local changes for one deck
ritual deck-sync push --dry-run              # preview without sending anything
ritual deck-sync pull --yes                  # accept dropping lines the parser can't read
ritual deck-sync pull --only additions       # add cards locally, never remove any
ritual deck-sync push --only removals        # push removals only, add nothing remotely
ritual deck-sync push "Winota Stax" --force  # overwrite remote edits made since the last sync
ritual deck-sync status --output json        # what is linked, and when each last synced
ritual deck-sync link "Alpha Deck" https://archidekt.com/decks/123456  # link an existing remote deck
```

A text-mode run closes with a tally —
`Synced 4 decks (2 with changes), 1 skipped, 1 failed.` — while
`--output json` emits the full per-deck report instead.

### Linking and status

`push` only operates on decks whose front matter carries `sourceUrl` +
`sourceId`, which `import`/`import-account` write. For a deck built locally,
create it on archidekt.com first (Ritual cannot create one — Archidekt has no
API for it), then link it. `link` takes an Archidekt **deck** URL (a bare id or
another service exits 2), canonicalizes it, and rewrites the front matter only —
card lines, `&N` ids, and prose survive byte for byte. It takes `--dry-run`,
`--output`, and `--quiet`; the MCP `set_list_metadata` tool performs the same
write (there, `sourceId` and `sourceUrl` must name the same Archidekt deck or
the call is rejected). A deck name two decks answer to exits 2, not 3.

`status` is read-only and offline (no Archidekt session needed): it lists every
linked deck with its URL and `lastSynced`, plus when the account's collection
last synced (or, when that record exists but cannot be read,
`Collection: sync state unreadable (…)` rather than `never synced`).

### Push divergence guard

A push makes Archidekt match the local file, so cards added on archidekt.com
since the last sync would be deleted. A push therefore compares the remote deck's
`updatedAt` against the deck's `sourceUpdatedAt` — the remote `updatedAt` the
last sync observed — and **fails** that deck when the remote moved on:

```
Remote deck changed since last sync (remote: …, last synced against: …) — pull first, or pass --force to overwrite remote changes.
```

Pull that deck first (the usual fix), or pass `--force` to overwrite
deliberately. A pull records the baseline even when it finds **no card changes**
— a remote rename or category shuffle moves `updatedAt` without giving a pull
anything to apply, and "pull first" has to clear the refusal in that case too
(such a pull rewrites the front matter only). `--dry-run` reports the same
refusal without needing `--force`. A deck that has never synced has nothing to
compare against and pushes normally; a remote that reports no usable
`updatedAt` is pushed with a warning saying the guard could not run.

Both sides of the comparison are Archidekt's clock, which is why it is
`sourceUpdatedAt` and not `lastSynced` (your machine's wall clock, shown by
`status`): a client running behind the server would otherwise diverge against
its own push. Never hand-author either field.
Only decks that pushed cleanly get fresh stamps — a failed deck keeps
its old ones. `collection-sync push` has **no** such guard: it is
last-writer-wins, so preview it with `--dry-run` when you also edit on
Archidekt.

`--only additions` / `--only removals` narrows a run to one side of the diff.
The vocabulary is relative to the **sync destination** — the local files on a
pull, Archidekt on a push — so additions are new cards and quantity increases
*there*, removals are deleted cards and quantity decreases. The other side is
still reported ("Skipped 3 removals (--only additions).") and simply not
applied.

Use it when the remote and local decks are deliberately out of step and only one
direction of change should carry over. It filters cards only: a pull still
adopts the remote format. `collection-sync` takes the identical flag (see the
**ritual-collections** skill).

Syncing rewrites the deck file, so a line the parser cannot read would be deleted.
Such decks are listed with their exact lines and confirmed before syncing;
`--yes` answers up front, and without a terminal (`--no-input`, a pipe, or
`--output json`) those decks fail instead.

A pull also adopts the deck's Archidekt format (mapped onto Ritual's format
keys). A push does not push the local format back.

The same sync runs from the admin site's **Sync Decks** page (deck toggles,
direction, change filter, live per-deck progress, and each deck's last-synced
time) and from the MCP `sync_decks` tool (same `only` field).

## Primer

```bash
ritual get-primer "Winota Stax"           # print a local deck's primer as Markdown
ritual get-primer <moxfield-url>          # fetch a primer from Moxfield
```

## Price

The unified `price` command covers all list types; scope it with `--deck` or a
name. An interactive browser opens on a TTY — for agents, always pass
`--summary`, `--output json`, or the global `--no-input` flag:

```bash
ritual price --deck --summary                       # every deck's totals
ritual price "Winota Stax" --no-input               # one deck's cards + totals
ritual price "Winota Stax" --output json --quiet
ritual price "Winota Stax" --prices eur             # usd | eur | tix (defaults to config defaultCurrency)
```

Deck totals cover every section except extras (maybeboard/token). Each deck also
reports a "lowest" total (cheapest printing of every card) and a quantity-weighted
unpriced-card count.
