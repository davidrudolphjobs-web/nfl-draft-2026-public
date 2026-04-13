# CLAUDE.md

Guidance for Claude Code (and other AI assistants) when working in this repository.

## Project overview

**2026 NFL Draft Big Board** — a single-file, client-side static website (`index.html`) that
renders a top-50 prospect board with search, filtering, sorting, a detail panel with scouting
reports, NFL Combine measurables, "Daniel Jeremiah's Take," and external source links.

- **Live hosting**: static site — just open `index.html` in a browser. No build, no server, no
  dependencies installed locally.
- **Stack**: Vanilla HTML + CSS + JS. All markup, styles, data, and logic live in one file.
- **Data sourcing (human-curated, pasted inline)**:
  - Tankathon Big Board — overall ranks
  - The Ringer / Danny Kelly — scouting profiles (strengths, weaknesses, background, fun facts,
    NFL comps, `ringerRank`, traits)
  - The Athletic / Dane Brugler — used for players not in Ringer's Top 50
  - Daniel Jeremiah's Top 50 (NFL.com) — `djRank` and `djNote`
  - NFL.com prospect pages — `nflGrade`, `nflUrl`, headshot IDs
  - NFL Scouting Combine — `combineData` measurables (height/weight/arm/wing/hand + on-field
    testing: 40, split, vertical, broad, 3-cone, shuttle, bench)

## Repository layout

```
/
├── index.html      ← the entire app (~765 lines: <style>, data, <script>)
└── .git/
```

There is no `package.json`, no build tool, no test suite, and no CI. Do not add any unless
explicitly asked.

## Running locally

Just open `index.html` in a browser. For iOS/mobile testing, serve over HTTP (any quick static
server works, e.g. `python3 -m http.server 8000` from the repo root).

## Code architecture (within `index.html`)

Lines are approximate — use them as orientation, not exact targets.

1. **Head + CSS** (lines 1–248)
   - CSS custom properties in `:root` at the top define the palette (`--bg`, `--card`,
     `--border`, `--accent`, `--muted`, position colors, etc.).
   - Google Fonts: **Playfair Display** (editorial serif for names / numbers), **Libre Franklin**
     (body sans), **JetBrains Mono** (pill/tab labels), **Bebas Neue**.
   - Design direction is **NYT-editorial-on-dark**: pure `#0a0a0a` background, thin borders, no
     gradients, muted grays, italic serif headings, staggered `panelReveal` animations.
   - Mobile breakpoint: `@media(max-width:768px)`. Mobile hides the DJ-Take, Combine, and HT/WT
     columns and renders the detail panel full-screen.

2. **DOM shell** (lines ~250–292)
   - `.header` → title + sub-source line.
   - `.controls` → `#searchBox`, `.tab-group` (OVERALL RANK / BY SCHOOL), `.pill-group` (position
     filters ALL / QB / RB / WR / TE / OT / IOL / DL / EDGE / LB / CB / S).
   - `.main-content` splits into `#board` and `#detailPanel` (with a `.drag-handle`).

3. **Scroll restoration** (lines 294–300)
   - `history.scrollRestoration = "manual"` plus multiple `scrollToTop()` calls. Do not remove —
     this fixes an iOS Safari regression documented in commit `7d7f8b6`.

4. **Data structures** (lines ~301–409)
   - `profiles` (**line 301**, huge single-line object): keyed by player name. Shape per entry:
     ```js
     {
       ringerRank: Number|null,    // null means "Not in Ringer Top 50"
       nflComp: String,            // e.g. "Josh Downs"
       age: String, class: String, // optional
       tagline: String,            // single-sentence hook
       traits: String[],           // shown as pill chips
       strengths, weaknesses, background: String,  // long prose
       funFacts: String[],
       source: String,             // e.g. "The Ringer / Danny Kelly (Jan 2026)"
                                   //  or  "The Athletic / Dane Brugler (Feb 2026)"
       nflGrade: Number,           // e.g. 6.36
       nflUrl: String,             // full nfl.com/prospects/... URL (last path segment = PID)
       djRank: Number|null,        // Daniel Jeremiah Top 50 rank
       djNote: String              // DJ's take (quoted verbatim)
     }
     ```
   - `prospects` (line 309, array of 50): `{rank, name, pos, school, height, weight, stats, conf}`.
     `stats` shape depends on position — see `getStatColumns()` at line ~442.
   - `espnIds` (line 362): school → ESPN team ID, used to build logo URLs via
     `https://a.espncdn.com/combiner/i?img=/i/teamlogos/ncaa/500/${id}.png`.
   - `logoOverrides` (line 364): per-school logo URL overrides (e.g. Ohio State uses the
     500-dark block-O).
   - `combineData` (line 370): keyed by player name, measurables + optional `forty`, `split`,
     `vert`, `broad`, `threeCone`, `shuttle`, `bench`, plus `dnp`/`dnpNote`/`partial` flags.

5. **Helpers** (lines ~365–448)
   - `logoUrl`, `logoHtml`, `fallbackLogo`, `detailLogoFallback` — school crest rendering.
   - `headshot(p)` — NFL.com headshot with two-URL fallback chain
     (`god-prospect-headshots/2023/{pid}` → `god-draft-headshots/2026/{pid}` → school logo
     fallback). Implemented via `hsError()` at line 369.
   - `combineHtml(name)` — renders the combine section. Handles `dnp`, `partial`, and missing
     cases.
   - `djTakeHtml(name)` — DJ's Take section (styled as a pull-quote, respects `--detail-font`).
   - `gradeColor(g)` — grayscale gradient from bright white (≥7.0) to dim (#666 <6.0).
   - `nameSlug` / `buzzSlug` — URL slug builders for outbound Tankathon and NFLDraftBuzz links.
     `buzzSlug` has a position map (`EDGE→DE`, `IOL→G`, `DL→DT`) and a school override
     (`David Bailey → Stanford`).
   - `getStatColumns(pos)` — returns the per-position stat column set. QB/RB/WR/TE/O-line all
     have different stat shapes; defense uses tackles/sacks/PD/INT/FF.

6. **State + rendering** (lines ~450–622)
   - Module-level state: `activePos`, `activeView`, `sortCol`, `sortDir`, `searchTerm`,
     `activePlayer`, `boardSortCol`, `boardSortDir`.
   - `getFiltered()` (line 452) — filter + sort pipeline.
   - `renderBoard()` (line 471) — main table renderer (rank view). Toggles `body.board-wide`
     class which controls DJ-Take column visibility (only shown when no player is active).
   - `renderBySchool(list)` (line 519) — groups the filtered list by school with sticky
     group headers.
   - `renderProfile(name)` (line 546) — assembles the scouting sections inside the detail
     panel. Source attribution at line 557 picks Brugler/Athletic vs. Ringer based on
     `pr.source.includes('Brugler')`.
   - `showDetail(rank)` (line 564) — builds the detail panel DOM, including toolbar
     (font slider + fullscreen button), header, measurables row, combine section, YouTube
     highlights link, DJ's Take, profile, and a Source Profiles section with outbound links.
   - `gradeKeyHTML()` (line 517) — the NFL.com grade scale legend. Appended below every board.

7. **Detail panel UX** (lines ~624–754)
   - Font slider: `fontSteps` array `[12…40]`, writes CSS var `--detail-font`, persisted to
     `localStorage.draftFontSize`.
   - Fullscreen: uses `visualViewport` API where available. **iOS Safari quirk**: the panel is
     re-parented to `document.body` on enter and restored on exit. Do not "simplify" this —
     multiple commits fixed this specific iOS behavior (`4afed85`-ish and neighbors).
   - Resizable panel: drag handle on the left edge, saves width to `localStorage.detailPanelWidth`
     (min 200px, max `window.innerWidth - 160`).
   - Close via ESC key or "← Back to Board."

8. **Event wiring + boot** (lines ~756–763)
   - Pill / tab / search / sort listeners, then initial `renderBoard()`.

## Conventions and gotchas

- **Single file.** Keep all changes inside `index.html` unless the user explicitly asks to split
  things out. Don't introduce a bundler, framework, or module system.
- **Style priority**: Playfair Display italic for names and large numbers; all-caps
  letter-spaced labels in Libre Franklin or JetBrains Mono; muted gray for secondary text.
  Accent is always pure `#fff` — avoid colored/blue accents (commit `cc077eb` intentionally
  removed blue tints; commit `1114059` stripped a colored combine badge). The combine indicator
  is a muted serif checkmark `✔`, not a green dot (see commit `04086ef`).
- **Data integrity**:
  - A player must appear in `prospects[]` to show up on the board. A matching key in `profiles`
    and `combineData` (exact name match, including punctuation like "Jr." and apostrophes) is
    what unlocks the scouting panel and combine section.
  - `ringerRank: null` is the correct signal for "Not in Ringer Top 50" — do not fabricate
    ranks (see commits `1c67c20`, `fc647eb`).
  - `djRank`/`djNote` come from Daniel Jeremiah's Top 50 2.0 (NFL.com) — keep source
    attribution in `djTakeHtml` pointing at that article.
  - For non-Ringer profiles, set `source` to contain `"Brugler"` so the attribution footer at
    line 557 renders correctly (commit `5498bfe`).
  - `nflGrade` values correspond to NFL.com's official grade scale rendered in `gradeKeyHTML()`
    — stay within that scale (5.50–8.0) and pull from actual nfl.com profile pages.
- **No duplicate profile keys.** JS object literals silently overwrite duplicate keys — a past
  JS syntax error came from this (commit `63a728d`). When editing the mega-line `profiles`
  object, search for duplicates by name.
- **Combine data flags**: use `dnp: true` + `dnpNote` when a player didn't test; `partial: true`
  when they did some drills only. Don't invent 40 times.
- **Position pills** render in the order listed in the HTML — keep consistent with the existing
  palette in the `.pos-*` CSS classes.
- **iOS Safari**: any change to fullscreen, scroll restoration, or the detail panel's positioning
  should be sanity-checked on mobile. Several commits (`9aca2ae`, `7d7f8b6`, iOS fullscreen
  commits) exist specifically to fix iOS regressions.
- **Do not remove or "clean up" the inline-all-on-one-line data objects** (lines 301 and 308
  region). That's an intentional trade-off — the file is source-controlled as a single static
  artifact.

## External URLs used at runtime

- School logos: `https://a.espncdn.com/combiner/i?img=/i/teamlogos/ncaa/500/{id}.png`
- Player headshots (primary): `https://static.www.nfl.com/image/private/f_auto,q_85/league/god-prospect-headshots/2023/{pid}`
- Player headshots (fallback): `https://static.www.nfl.com/image/private/f_auto,q_85/league/god-draft-headshots/2026/{pid}`
- Source links per player: The Ringer big board, NFL.com prospect page, NFLDraftBuzz, Tankathon
- Google Fonts CSS for Playfair/Libre Franklin/JetBrains Mono/Bebas Neue

When adding a new player with no existing NFL.com entry, set `nflUrl: null` — `headshot()` will
fall back to the school logo crest.

## Git / branch workflow

- Default branch: `main`.
- The user uses disposable per-task feature branches, typically `claude/<slug>-<id>` (see
  `claude/fix-player-profiles-loading-*` and `claude/add-claude-documentation-*`).
- Develop on the branch the user specifies, commit in small logical units with descriptive
  messages (see existing log — commits tend to be single-topic and imperative mood), then push
  to the same branch.
- **Do not open a pull request unless explicitly asked.** Do not push to `main` directly.

## Typical edit patterns

- **Add a new prospect**: append to `prospects[]` with `rank`, add matching `profiles[name]`
  entry with at least `nflComp`, `tagline`, `strengths`, `weaknesses`, `source`, `nflGrade`,
  `nflUrl`. Optionally add `combineData[name]`. If the school is new, add to `espnIds`.
- **Update combine data**: edit `combineData[name]` in place. Flag partial/DNP with the
  established keys.
- **Update rankings**: reorder `rank` fields in `prospects[]`. Keep them a contiguous 1…50
  sequence.
- **Style tweaks**: prefer editing existing CSS (top of the file). Keep the editorial direction
  consistent — no colored accents, no gradients, no rounded decorative elements beyond what's
  already there.

## What *not* to do

- Don't add `README.md`, docs, licenses, or meta-files unless requested.
- Don't introduce frameworks, bundlers, package managers, TypeScript, or JSX.
- Don't split `index.html` into multiple files.
- Don't add analytics, third-party scripts, or service workers.
- Don't fabricate `nflGrade`, `ringerRank`, `djRank`, or combine numbers — use real sourced
  data or leave the field `null` / omit it.
- Don't commit to `main` or push force unless explicitly authorized.
