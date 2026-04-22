# Pokeforge Meta Data Update Runbook

**Trigger:** User says "update" or "update the meta data" or similar.

This document is a prompt for Claude — it tells future-you exactly what to do to refresh all competitive meta data on the site. Follow it end-to-end, uninterrupted, no check-ins unless a genuine blocker appears.

## Data sources

| Source | What it provides | URL |
|---|---|---|
| Pikalytics (primary) | Usage %, abilities, moves, items, teammates | `https://www.pikalytics.com/ai/pokedex/championstournaments` |
| Pikalytics per-Pokemon (primary) | Same fields per mon | `https://www.pikalytics.com/ai/pokedex/championstournaments/<slug>` |
| Serebii (authoritative) | Full Champions-legal movepool per Pokémon | `https://www.serebii.net/pokedex-champions/<slug>/` |
| Smogon (fallback, for EV spreads only) | Spread aggregations (252-scale, auto-converted to 32-scale in Champions mode) | `https://pkmn.github.io/smogon/data/stats/gen9vgc2025.json` |

**Critical:** Pikalytics' `championstournaments` format has **no EV spread data**. That's why Smogon stays in the mix — only for the `spreads` field inside `usageStats.pokemon[name]`. Do not "fix" the Smogon fetch out of existence.

**CORS:** Pikalytics has no CORS headers. Data MUST be scraped by Claude via WebFetch during updates and embedded as static JSON into `index.html` under the `PIKALYTICS_DATA` constant. Do not try to make the browser fetch Pikalytics at runtime.

## Step-by-step

### 1. Fetch the top usage index

```
WebFetch https://www.pikalytics.com/ai/pokedex/championstournaments
Prompt: "Return ALL 50 entries verbatim as: RANK. NAME USAGE% SLUG. Include data month. No summary."
```

Extract: data month (for the timestamp), the ranked list of top 50 names with their URL slugs and usage percentages.

### 2. Fetch per-Pokemon data for each of the top 50

Fire 10 parallel `WebFetch` calls per message across 5 messages (50 total). Use this prompt template for every fetch, varying only the URL slug:

```
URL: https://www.pikalytics.com/ai/pokedex/championstournaments/<slug>
Prompt: "Return raw data in this exact format, nothing else:

ABILITIES:
- Name: pct%
(top 5)

MOVES:
- Name: pct%
(top 10)

ITEMS:
- Name: pct%
(top 10)

TEAMMATES:
- Name: pct%
(top 10)

Use exact names verbatim as shown on the page. No commentary."
```

Discard obvious typo entries (<0.1% with misspelled names like "Intimidaci", "Floetitte", "glimmoranite"). Keep the real entries verbatim.

### 3. Rewrite `PIKALYTICS_DATA` in `index.html`

Location: search for `const PIKALYTICS_DATA={` in [index.html](index.html). It sits right before the `// --- VGC Usage Stats (Smogon gen9vgc2025 for EV spreads only) ---` comment.

For each Pokemon, write one line:

```js
'<Name>':{usage:<decimal>,abilities:{'<A>':<dec>,...},moves:{...},items:{...},teammates:{...}},
```

Rules:
- Name key uses Pikalytics display format (e.g. `'Rotom-Wash'`, `'Floette-Eternal'`, `'Kommo-O'`, `'Ninetales-Alola'`). This matches `fmt()` in the code and Smogon's Gen 9 keys — if they match, the merge just works.
- Percentages are decimals: `51.18%` → `0.5118`. Round to 4 decimals.
- Top 10 for moves/items/teammates, top 5 for abilities (fewer if the page has fewer).
- Skip typo/garbage entries under 0.1%.
- Escape apostrophes in move names: `"King's Shield"` (use double quotes to avoid JS escape headaches).

Replace the entire `PIKALYTICS_DATA = {...}` block in one `Edit`. Don't incremental-edit.

### 4. Re-rank `VGC_POKEMON` and `VGC_THREATS`

Find the `const VGC_POKEMON=[` and `const VGC_THREATS=[` arrays in [index.html](index.html).

- **`VGC_POKEMON`** (top 40): order by new Pikalytics usage. Keep the `{name, types, role, why}` shape. Update `why` text to reflect current ability/item/move patterns (one short editorial line, never more than a few words). Pokémon that drop out of top 40 are removed; new top-40 entries are added.
- **`VGC_THREATS`** (top 30): order by new Pikalytics usage. Shape is `{name, types, stab}` — **no `key` field** (removed deliberately; don't add it back).

`stab` should equal `types` for dual-type mons. For mono-type, `stab` is a single-element array matching types.

### 5. Refresh Champions movepools from Serebii (only when a Game Freak balance patch is rumored)

Serebii's `/pokedex-champions/` entries reflect the current Champions-legal movepools, which differ from the main-game Gen 9 SV learnsets (e.g. Incineroar has lost Knock Off + U-Turn; Body Press was removed from many mons). The site embeds these as `CHAMPIONS_LEARNSETS` in [index.html](index.html), keyed by the lowercase API name (e.g. `'incineroar'`, `'rotom-wash'`, `'ninetales-alola'`).

**You do NOT need to re-scrape every update** — movepools rarely change without a balance patch. Only refresh when:
- A new Game Freak patch notes mention move changes
- A user reports a specific Pokémon missing a move that should be legal, or having a move that's been banned

**How to scrape:**

URL pattern: `https://www.serebii.net/pokedex-champions/<name>/` (lowercase, hyphen-separated). Prompt:
```
List the moves in the "Standard Moves" section, one per line, move names only verbatim. No stats. Nothing else.
```

Batch 10–14 WebFetch calls in parallel per message. Watch for rate limit resets (1pm CDT / 2am CDT — if you hit the limit, wait and retry).

**Form variants:** Regional forms share the base URL and have additional sections (`"Alola Form Standard Moves"`, `"Galarian Form Standard Moves"`, `"Hisuian Form Standard Moves"`, `"Paldean Form Standard Moves"`). Use this prompt for base pages with alt forms:
```
List moves from BOTH form sections in this exact format, move names only, verbatim:

BASE_NAME:
(all moves from the base Standard Moves)

FORM_NAME:
(all moves from the form's Standard Moves section)
```

**Rotom** is unique: one page, one "Standard Moves" shared by all forms, plus "Special Moves" listing each form's exclusive move (Heat: Overheat, Wash: Hydro Pump, Frost: Blizzard, Fan: Air Slash, Mow: Leaf Storm). Build per-form arrays as `base + [form-exclusive]`.

**Lycanroc / Meowstic** have per-form sections (Midday/Midnight/Dusk, Male/Female). The site only keys `lycanroc` and `meowstic` — use the most common form (Midday for Lycanroc, Male for Meowstic) to match the site's canonical entry.

**Floette** in `CHAMPIONS_POKEMON` maps to Floette-Eternal's learnset (the form used in Champions). Key as `'floette'`.

**Known 404s:** `mr-rime` and `ursaluna` — Serebii doesn't expose these under the slugs we tried. Omit from `CHAMPIONS_LEARNSETS`; they fall back to PokeAPI/Showdown movepools (acceptable since they're near-zero usage).

**Writing the entries:**

Format each Pokémon as:
```js
'<name>':_mp("Move One,Move Two,Move Three,..."),
```
Use the `_mp` helper that's already in the file (converts spaces to hyphens to match site format). Names like "Will-O-Wisp" and "Double-Edge" stay hyphenated — `_mp` only replaces spaces. Use double quotes for moves with apostrophes (`"King's Shield"`).

### 6. Update the "Last updated" timestamp

One line in [index.html](index.html) near line 1014:

```html
<p style="font-size:12px;color:var(--muted)">Last updated: <MONTH DAY, YEAR> · By InciniForge</p>
```

Use today's actual date.

### 7. Verify

- Grep for `PIKALYTICS_DATA` — should be exactly one constant declaration.
- Grep for `CHAMPIONS_LEARNSETS` — should be exactly one constant declaration, plus usage inside `getMovePool`.
- Grep for `threat.key` — should return zero results (that field was removed).
- Grep for `loadUsageStats` — the merge loop should still iterate `PIKALYTICS_DATA`.
- Grep for `p.movePool||` — every consumer should be `getMovePool(p)`, not `p.movePool||[]`, except inside `getMovePool` itself.
- Spot-check 2–3 Pokemon: their names in `PIKALYTICS_DATA`, `VGC_POKEMON`, `VGC_THREATS`, and `CHAMPIONS_LEARNSETS` match exactly.
- Open `index.html` in a browser, switch Champions mode on, search Incineroar, confirm Knock Off and U-Turn do NOT appear in the move list. Confirm moves like Parting Shot and Fake Out do appear. Toggle Champions mode off and confirm the full PokeAPI move list returns.

### 8. Commit — only if the user asked

Don't auto-commit. Wait for the user to ask. If they do, write a commit message like:
`Update Pikalytics meta data to <YYYY-MM> snapshot`

## Things NOT to change during a routine update

- `CHAMPIONS_POKEMON` legal set — only changes when GameFreak opens/closes the roster
- `ITEMS` / `POPULAR_ITEMS` — edit manually when a new item is introduced
- `BENCHMARKS` (speed tiers) — only touches when a new relevant Pokemon is added
- `MEGA_DATA` — only when new megas are added
- The EV spread conversion (div by 8, cap 32, total 66) — working as intended
- The Smogon fetch URL — keep `gen9vgc2025.json`; it's the best available spread source

## If Pikalytics changes their format

Signs: the markdown no longer parses, the `/ai/pokedex/championstournaments` index returns a different structure, or they add EV spreads for Champions.

- If they add Champions EV spreads: extract them, override Smogon spreads, consider dropping the Smogon fetch entirely.
- If the markdown format changes: adjust the WebFetch prompt to match. The `ABILITIES/MOVES/ITEMS/TEAMMATES` sections are load-bearing.

## Known quirks

- Some mons have Mega Stone names that look like typos (e.g. `Floettite` — this is actually the real Floette-Eternal mega stone in Champions). Keep these; they aren't typos even though they look like them.
- Percentages sometimes total >100% across item lists (multiple Pokemon per entry summed). That's Pikalytics' reporting quirk, not ours to correct — store as given.
- `Kommo-O` uses a literal hyphen-O; `Ninetales-Alola` and `Rotom-Wash` are the canonical forms. Don't helpfully "correct" these.
