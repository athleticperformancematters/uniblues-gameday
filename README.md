# Uni Blues Game Day Manager

A single-file web app for managing player rotations live on an iPad during an Australian Rules football match.

Built by [Athletic Performance Matters](https://athleticperformancematters.com.au) for University Blues Football Club (VAFA Premier A Seniors).

**Live:** [gamemanage1.netlify.app](https://gamemanage1.netlify.app)

---

## What it does

Tracks who is on the ground, who is on the bench, how long each player has been on since their last rotation, and how that compares to the rest target set for them. The aim is to let a coach make rotation decisions at a glance from the boundary, rather than doing arithmetic in their head.

- **Live stint timers** — each player's time since last rotation, colour-coded green → amber → red against their individual rotation target
- **Tap-to-rotate** — tap an on-field player then a bench player to interchange them
- **Tap-to-swap** — tap two on-field players to exchange positions (not counted as a rotation)
- **Ground layout** — cards arranged as an actual field: backs, mid/wings/ruck, forwards
- **Interchange bench** — grouped by line, ordered so the most-rested player sits at the bottom, with a warning when someone has rested past their allowance
- **Quarter management** — clock, speed control, quarter start/end with wall-clock timestamps
- **Export** — tab-separated match data that pastes straight into Google Sheets

## Why it exists

Rotation management in amateur football is usually done on paper or from memory. Neither scales well when you are also coaching. This app makes the load visible in real time and produces a match record afterwards that can be reviewed alongside GPS data.

## Technical notes

Deliberately built as **one self-contained HTML file** — no build step, no dependencies, no framework. It runs offline once loaded, which matters at grounds with poor reception, and deployment is a single file drop.

Target device is an **iPad 9th generation in landscape** (1080 × 810 CSS points). The layout is tuned to fit that screen without scrolling.

Player data is loaded from a CSV (squad, positions, build, per-player rotation targets). Match state is autosaved to the browser, with a rolling backup and an automatic file download at each quarter break.

See [`HANDOVER.md`](HANDOVER.md) for architecture, data model, and development notes.

## Usage

Open `index.html` in a browser, or visit the live site. Upload a squad CSV, or use the built-in sample squad. Press Start at the first bounce.

## Licence

© Good Future Investments Pty Ltd t/a Athletic Performance Matters. All rights reserved.

Published publicly for transparency and deployment convenience. Not licensed for reuse or redistribution — please get in touch if you would like to use it.
