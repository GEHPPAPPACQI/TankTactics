# Tank Tactics — async form-driven tank game

15×15 board, fog of war (6), attack range (3), 1 AP start / +1 AP daily at 12:00,
3 lives, actions submitted any time before the daily deadline.

## Architecture

```
Players (Google Sheet: name, password, position, AP, lives, ...)
        │
Google Apps Script web app  ── doGet: password → fog-of-war view (JSON)
        │                     ── doPost: queue action (validated, 1 AP reserve)
        │                     ── dailyTick() trigger @ 12:00: resolve queued
        │                        actions in random order → +1 AP → reset defense
        │
GitHub Pages site ── login (name+password) → renders your fog-of-war board,
                     form to submit actions, auto-refresh after submit
```

## Setup (≈15 min)

### 1. Google Sheet backend
1. Create a new Google Sheet.
2. **Extensions → Apps Script**. Delete default code, paste `gas/Code.gs`
   (and replace `appsscript.json` content if you use clasp).
3. Run `setupSheets` once (authorize the script).
4. Run `addPlayer("Alice","herPassword")` once per player — random spawn, 1 AP, 3 lives.
   (Or paste passwords into the `Players` sheet manually and only set name/password.)
5. **Triggers → Add trigger**: `dailyTick`, time-driven, day timer, 12:00–13:00
   (noon local time — set the script's timeZone in `appsscript.json` if needed).
6. **Deploy → New deployment → Web app**: execute as *Me*, access *Anyone*.
   Copy the `/exec` URL.

### 2. GitHub Pages site
1. Create a repo `tank-tactics` (or use an existing Pages site).
2. Commit `site/index.html` as `index.html` on the `main` branch.
3. Settings → Pages → deploy from `main` / root.
4. In `index.html`, replace `API_URL` with your Apps Script `/exec` URL.

### 3. Play
- Players open the Pages site, log in with their name + password.
- They see only cells within Chebyshev distance 6 of their tank.
- They submit one action (move / attack / transfer / defend); it queues and
  resolves at the 12:00 tick. Resolution order is randomized each day.
- Dead players see only who killed them.

## Rules implemented (see `CONFIG` in `gas/Code.gs` to tune)

| Action  | Cost | Effect |
|---|---|---|
| Move    | 1 AP | 1 cell (8-directional, blocked cells/bounds) |
| Set avatar | free | set your tank image URL (shown to anyone who sees you) |
| Attack  | 1 AP | 1 damage, range 3 (Chebyshev), blocked by active defense |
| Transfer| n AP | give n ≥ 1 AP to any living player (any distance — configurable) |
| Defend  | 1 AP | blocks 1 damage; max 1 per day regardless of AP |

- Actions are **queued** and all resolve at the daily tick (prevents
  first-mover advantage; random order).
- `last_attacker`/`killer` tracked per player → dead players learn only their killer.
- Dead tanks disappear from everyone's board.
- You can queue actions **only up to your available AP** (queued actions reserve their cost).
- **Fog of war is circular** (Euclidean radius 6); attack range 3 remains square/Chebyshev.

## Design decisions / assumptions (flagged, easy to change)

- **Fog uses Euclidean (circular) radius 6**; attack range 3 still uses Chebyshev distance.
- **Diagonal moves allowed**; set `ALLOW_DIAGONAL_MOVE: false` for 4-directional.
- **AP transfer works at any distance**; set `TRANSFER_ANY_DISTANCE: false` to require visibility.
- **Actions resolve at noon**, so "submit by next day" = before the 12:00 tick.
- Queued actions reserve nothing; AP is checked again at resolution.
- Passwords are stored in plaintext in the sheet — fine for a friends game,
  not for anything serious.

## Possible next steps
- Action history page (per-player, from the `Log` sheet).
- Telegram/Discord bot instead of the web form.
- Spectator view (full board, no fog) with a separate password.
- Google Form as an alternative action channel (form → sheet → backend reads it).
