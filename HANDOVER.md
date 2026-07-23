# Uni Blues Game Day Manager — Developer Handover

A single-file web app for managing AFL player rotations live on an iPad during a match.
Built for University Blues FC (VAFA Premier A Seniors).

---

## 1. What this is

`index.html` is a **single self-contained file** — HTML, CSS and JavaScript all inline.
No build step, no dependencies, no framework, no npm. You open it in a browser and it runs.

That is a deliberate constraint, not an oversight. It means:
- It works offline once loaded (important — grounds often have poor reception).
- Deployment is dragging one file onto Netlify.
- There is nothing to break between "works on my machine" and "works on the iPad".

**Keep it that way unless you have a strong reason not to.** If you add a build step you
also add a class of failure that can only surface on match day.

**Target device:** iPad 9th generation, landscape, Safari. 1080×810 CSS points.
The layout is tuned to fit that screen with no scrolling. Always test at that size.

---

## 2. Getting set up in VS Code

```bash
mkdir uniblues-gameday && cd uniblues-gameday
git init
# copy index.html into this folder
git add index.html
git commit -m "Initial commit — working app as of handover"
```

Version control matters more than usual here, because there's one file and no tests.
Commit before each change so you can always get back.

**Useful extensions:**
- **Live Server** (Ritwick Dey) — right-click `index.html` → "Open with Live Server".
  Gives you auto-reload on save, which makes iterating on layout much faster.
- **Prettier** — optional. If you use it, be aware it will reformat the entire file;
  commit first.

**Simulating the iPad in Chrome/Edge DevTools:**
F12 → toggle device toolbar → Responsive → set to **1080 × 810**.
This is the single most useful habit for this project. Most bugs in this app's
history have been "fits on my laptop, clips on the iPad".

---

## 3. Deploying to Netlify

The site is live at **gamemanage1.netlify.app**.

Two deployment paths. You are currently on the first.

### Current: drag-and-drop
1. The file **must be named `index.html`** and sit alone in a folder.
2. Go to app.netlify.com → the **gamemanage1** project → **Deploys** tab.
3. Drag the *folder* (not the file) onto the drop zone at the bottom.
4. Wait for "Published".

Do not use Netlify Drop (the front-page drop zone) for updates — that creates a
*new separate site* each time. This has caught you before.

### Recommended: connect to Git
Given you're moving to VS Code, this is worth doing once:

1. Push the repo to GitHub.
2. Netlify → gamemanage1 → **Site configuration → Build & deploy → Link repository**.
3. Build command: leave **empty**. Publish directory: `.` (or whichever folder holds `index.html`).
4. From then on, `git push` deploys automatically.

This gives you deploy history tied to commits, and one-click rollback if a change
breaks something on match day.

### Cache-busting on iPad — important
Safari caches aggressively and has repeatedly served you a stale build after a
successful deploy. After deploying:
- On the iPad, **close the tab entirely**, then reopen the URL. A refresh is often not enough.
- Or hold the reload button → "Reload Without Content Blockers".

If in doubt, put a visible version marker in the top bar temporarily so you can confirm
at a glance which build you're looking at. (There used to be a `BUILD v3 · GREEN` badge
for exactly this reason; it was removed once the layout settled.)

---

## 4. File structure

725 lines, in this order:

| Lines | Contents |
|---|---|
| 1–23 | `<head>`, CSS variables (colour palette) |
| 24–52 | CSS: top bar |
| 53–111 | CSS: layout, pitch geometry, bench column, live ticker |
| 112–172 | CSS: player cards |
| 173–~230 | CSS: modals; then HTML body |
| ~237–246 | Position/line constants, `QLEN`, storage keys |
| 248–262 | Persistence (`saveState`, `clearSavedState`, `hasAnySave`) |
| 264–305 | Built-in squad CSV + `parseCSV` |
| 307–330 | Game state `G`, `init()`, player accessors |
| 332–358 | Clock, speed, tick loop |
| 360–414 | Confirm dialog, quarter start/end, auto-backup |
| 416–460 | Tap handling: selection, rotation, position swap |
| 461–469 | Small helpers (`mmss`, `tmClass`, `lineCls`, …) |
| 470–546 | DOM structure build (`makeCard`, `buildField`) |
| 547–583 | Per-tick painting (`paintCard`, `paintAll`) |
| 584–612 | Top counters, controls, hint, log |
| 613–662 | Export generation |
| 663–724 | Reset, team upload, restore-on-load |

---

## 5. Core concepts

### Game state — `G`
One global object holding everything:

```js
G = {
  qtr, sec, running, speed, phase,   // phase: 'live' | 'break' | 'full'
  rotTot, qRot:{1,2,3,4},            // team rotation counts
  sel,                               // currently selected player name, or null
  log: [], events: [],               // ticker lines; rotation event records
  qtimes: {1:{start,end}, …},        // wall-clock per quarter
  players: { NAME: {...} }
}
```

Each player:
```js
{
  n, pos, flex, build, rotMin,       // from CSV. rotMin = rest target in minutes
  curPos,                            // current position — changes during play
  loc: 'on' | 'bench',
  slot,                              // stable ordering index — see below
  stintSec,                          // time since last rotation (the big number)
  onSecQ, restSecQ, rotQ,            // per-quarter
  onSecG, restSecG, rotG,            // whole game
  qField:{1..4}, qRot:{1..4}         // per-quarter breakdown for export
}
```

### The `slot` field — read this before touching layout
Cards are positioned on the pitch by **position group**, then ordered **within that
group by `slot`**. `slot` is assigned sequentially at init and only ever swapped.

This exists because two players can share a position (two wings, several backs), so
`curPos` alone can't determine which card sits where. When you swap two on-field
players, `swapPositions()` exchanges both `curPos` *and* `slot` — that's what makes
the cards physically trade places on screen.

Consequence worth knowing: you cannot simultaneously have manual swap ordering *and*
auto-sort-by-time-on-field, because both want to control the same axis. If you want
a "longest on field" indicator, render it as a badge rather than by reordering.

### Structure vs paint — the key performance split
- `buildField()` — **destroys and rebuilds the DOM**. Only call when the roster changes
  (rotation, swap, quarter change, team upload).
- `paintAll()` / `paintCard()` — **updates text and classes in place**, four times a
  second via the tick loop. Must not create or destroy elements.

`cardEls` caches element references per player so painting doesn't re-query the DOM.
If you add a new field to the card, add its reference to `cardEls` in `makeCard()`
and update it in `paintCard()`.

### Tap interaction model
Handled in `clickCard()`:

| First tap | Second tap | Result |
|---|---|---|
| on-field | on-field | **Position swap** — exchange positions and slots. Not a rotation. |
| on-field | bench | **Rotation** — bench player comes on, field player goes off. |
| bench | bench | Selection moves to the newly tapped player |
| any | same player | Cancel selection |

Tapping a vacant slot with a bench player selected sends them on (`clickVacant`).

Note `.card > * { pointer-events: none }` in the CSS — card children don't intercept
taps, so every tap lands on the card root. Removing this breaks tapping on iPad.

### Rotation counting — a real decision, not an accident
**One interchange = one rotation, credited only to the player leaving the field.**

The player coming on is *not* counted. Filling a vacant slot is *not* counted.
On-field position swaps are *not* counted.

An earlier version credited both players, which double-counted every quarter total
and produced export figures that didn't reconcile with the header. If you change this,
change it deliberately and check that Section 2's column sums still match the header.

For reference: the AFL cap is 75 *movements* per team per match. The wording is
genuinely ambiguous about whether a movement is one crossing or one interchange —
the odd number strongly implies one interchange = one movement, which is what this
app now does.

### Rest targets and the `!` marker
`rotMin` comes from the CSV's `Rotation Time` column, per player, in minutes.
`tmClass()` turns elapsed stint into a colour: green under 70% of target, amber
70–100%, red over. Red also pulses.

If a player has no `rotMin`, they show a red `!` in the timer box and never go
amber/red — the app is telling you it has no target for them, rather than silently
inventing one. `parseCSV` accepts either `Rotation Time` or `Rotation Time (min)`
as the column header.

---

## 6. The CSV format

```
Player Name,Playing,Interchange,Primary,Second Role,Build,Rotation Time
CONWAY,Y,N,FB,HB,Key,25
```

| Column | Meaning |
|---|---|
| Player Name | Unique — used as the key throughout |
| Playing | `Y` = in the squad, `N` = ignored |
| Interchange | `Y` = starts on bench, `N` = starts on field |
| Primary | Starting position: `FB HB M W Ruckman HF FF` |
| Second Role | Secondary position (parsed, currently unused in the UI) |
| Build | `Key` / `Medium` / `Small` — only `Key` shows a card tag |
| Rotation Time | Rest target in minutes. Blank → red `!` |

A built-in squad is hardcoded in `CSV_TEXT` so the app works with no upload.
The **Upload team** button replaces it at runtime; it does not persist across a reset.

---

## 7. Data persistence — the part that has actually bitten

There are three layers, added after a match's data was lost entirely:

1. **`localStorage['gdm_autosave_v1']`** — written on every rotation, swap, quarter
   change, and roughly every 5 game-seconds.
2. **`localStorage['gdm_autosave_backup']`** — the previous save, rolled over before
   each overwrite. Survives Reset and Upload Team.
3. **Auto-downloaded `.tsv` file at every quarter break** — `autoBackupFile()` fires
   from `doEndQuarter()`, producing e.g. `UniBlues_EndQ2_2026-07-04_1439.tsv`.

**Layer 3 is the one that matters.** Safari evicts localStorage for sites not visited
for ~7 days, and that is exactly how a full match of data was lost — both storage keys
came back empty. The downloaded files land in Files and are immune to that.

Declining the "Resume autosaved game?" prompt no longer deletes the save (it used to).
`clearSavedState()` only clears the primary key, never the backup.

**Operational advice worth passing to whoever runs the iPad:** check at half time that
the quarter files are appearing in Files. Treat them as the real record.

---

## 8. Export format

`buildExportText()` produces tab-separated text (pastes into Google Sheets with
automatic column splitting — no import dialog).

- **Header** — total and per-quarter rotations, wall-clock quarter times.
- **Section 1 — Player Summary**: field time, bench time, rotations, game %.
- **Section 2 — Quarter Load**: per-quarter time and rotation count, plus totals.
- **Section 3 — Rotation Events**: every interchange, paired off/on, game clock and
  wall clock.

Section 3 is the source of truth; 1 and 2 are derived. If they ever disagree,
recount from Section 3.

Known rough edge: `hhmm()` shows wall-clock times using device locale, and Game % is
derived from quarter wall-times when available, falling back to elapsed game time.
If quarters are paused a lot, that percentage drifts. Worth tightening.

---

## 9. Known issues and sensible next steps

**Known issues**
- Jersey numbers aren't in the CSV, so the `#` column in Section 1 exports empty.
- `buildLetter()` still exists but is effectively unused now that only `Key` renders a tag.
- Wall-clock timestamps are real tap times, so they diverge from game clock if you
  use 2×/4× speed or pause a lot. That's correct behaviour but can confuse.
- No undo. A mis-tapped rotation is logged as real and has to be corrected by
  rotating back, which then shows as two rotations.

**Sensible next steps, roughly in order of value**
1. **Undo last action** — highest value for match-day use. A single-step undo of the
   last rotation or swap would remove the main source of dirty data.
2. **Add/edit/delete players on the device** — currently roster changes require
   editing the CSV and re-uploading. This is the biggest usability gap.
3. **Jersey numbers** — one CSV column, then surface on cards and in the export.
4. **Mis-tap detection** — flag any player whose off/on gap is under ~60 seconds, as
   a likely accidental double-tap, either live or in the export.
5. **Longest-on-field indicator** — as a badge, not by reordering (see `slot` above).

---

## 10. Working with Claude Code on this

A few things that will make the collaboration go better, learned from building it:

- **Tell it the target viewport.** "Test at 1080×810, iPad 9th gen landscape" prevents
  a whole category of layout regressions.
- **Ask it to verify before shipping.** This app was developed with headless-browser
  checks (Playwright) that asserted things like "no card's bottom row is clipped" and
  "the field doesn't overflow". That caught several bugs that looked fine in a screenshot.
- **Be wary of "it works" without evidence.** Several times during development a change
  was reported as working when the live iPad was actually showing a cached build. If
  something looks unchanged, verify *which build* you're viewing before debugging the code.
- **Single file means large diffs.** Ask for targeted edits (`str_replace`-style) rather
  than whole-file rewrites, or you'll lose track of what changed.

---

## 11. Quick reference

```
Live site      gamemanage1.netlify.app
Deploy         app.netlify.com → gamemanage1 → Deploys → drag folder
File name      index.html  (required by Netlify)
Test viewport  1080 × 810
Storage keys   gdm_autosave_v1, gdm_autosave_backup
Quarter length QLEN = 20 (minutes) — line ~244
Backup files   UniBlues_EndQ<n>_<date>_<time>.tsv → Files app
```
