# Yahtzii Pro

A full-featured desktop Yahtzii scorecard and animated dice roller built with Python and PyQt6. Supports 1–8 players, enforces official rules (including Joker rules and Yahtzee bonuses), uses a live-clock dual timer for game and turn tracking, colour-codes every scoring option after each roll, and carries a full "Midnight Steel" dark theme across the entire application. A separate optional animated roller window replaces physical dice with 3-D flip animations, landing bounces, held-die glow pulses, and a power-charge bar — all tightly integrated with the scorecard.

---

## Requirements

```
Python 3.10+
PyQt6
PyQt6-Qt6
PyQt6-sip
PyQt6-Qt6-Svg (provides QSvgWidget and QSvgRenderer)
```

```bash
pip install PyQt6 PyQt6-Qt6 PyQt6-sip
```

> **Note:** `PyQt6.QtSvgWidgets` and `PyQt6.QtSvg` are imported at the top of the file. These ship with the standard PyQt6 wheel on most platforms; if you get an import error install `PyQt6-Qt6` separately.

---

## Running

```bash
python yahtzee.py
```

No other files are required. All die art is bundled as inline SVG path data inside the script.

---

## Project Structure

```
yahtzee-project/
│
├── yahtzee.py                  # Single-file application (scorecard + roller, ~2700 lines)
│
├── images/                     # Optional custom die artwork (SVG, one file per face)
│   ├── 1.svg
│   ├── 2.svg
│   ├── 3.svg
│   ├── 4.svg
│   ├── 5.svg
│   └── 6.svg
│
└── yahtzee_highscores.json     # Auto-generated on first completed game (top-10 all time)
```

The `images/` folder is **optional**. If present, the roller loads `images/{face}.svg` for each die face and recolours the fill with the active theme's accent colour. Any `fill="#000000"` attribute in the SVG source is replaced at load time with the current accent hex. If the folder or a specific file is absent, the script falls back seamlessly to built-in inline SVG path data that matches the same visual design.

`yahtzee_highscores.json` is created automatically in the working directory when the first game completes. It stores the top 10 all-time winning scores with player name, score, and date.

---

## Visual Design — Midnight Steel Theme

The scorecard uses a fixed dark palette called **Midnight Steel**. All colour constants are defined at the top of the file:

| Constant | Hex | Role |
|---|---|---|
| `CLR_BACKGROUND` | `#0F1117` | Window background |
| `CLR_TABLE` | `#161B27` | Table background |
| `CLR_UNCLAIMED` | `#1E2740` | Unclaimed cell |
| `CLR_ACTIVE_UNCLAIMED` | `#253354` | Active player unclaimed |
| `CLR_TOTAL_BG` | `#0A0D14` | Calculated rows |
| `CLR_ACCENT` | `#3B82F6` | Accent / highlights |
| `CLR_CLAIMED_TEXT` | `#93C5FD` | Text in claimed cells |
| `CLR_ACTIVE_TURN` | `#FBBF24` | Current player marker / timer |
| `CLR_DISABLED` | `#0D1020` | Joker-blocked cells |
| `CLR_VALID` | `#34D399` | Valid roll / confirmed dice |
| `CLR_INVALID` | `#F87171` | Zero-only cells (roller mode) |

When the roller is open, the scorecard repaints itself to match whichever roller theme is active (see **Themes** below). The `apply_roller_theme()` method derives new background, table, and accent colours from `_ROLLER_THEMES`, rebuilds the full Qt stylesheet via `colorsys` HSV manipulation, and triggers a complete table repaint.

---

## Application Flow

The app runs a `while True` loop from `__main__` that cycles through three screens — Registration → Roll-Off → Game — and handles all play-again paths without restarting the process.

1. **Registration** (`PlayerSetupDialog`) — players enter names, choose Digital Roller on/off, and pick a colour theme.
2. **Roll-Off** (`RollOffDialog`) — animated virtual dice determine turn order. Skipped automatically for single-player games.
3. **Game** (`YahtzeeScorecard`) — the main scorecard window. When the game ends, `GameOverDialog` lets players choose Same Order, Roll for Order, New Game / Change Players, or Quit. The outer loop handles each path without returning to the OS.

Theme and roller-mode choices are **carried forward** across all play-again paths. "New Game / Change Players" returns to Registration with all player names pre-filled and the last theme pre-selected; the roller checkbox resets to unchecked so players can make a fresh choice.

---

## Setup & Registration

On launch the **Registration** screen appears (`PlayerSetupDialog`, fixed 420 × 630 px). Enter between 1 and 8 player names. Unnamed slots default to `P1`, `P2`, … at game start. Players appear in the order entered; the Roll-Off screen can randomise order after this step.

### Digital Roller checkbox

Tick **🎲 Use Digital Roller (fully digital experience)** to activate the integrated animated roller window. The setting applies for the entire game and is carried forward on "Same Order" and "Roll for Order" restarts. It resets on "New Game / Change Players".

### Theme picker

A dropdown in the Registration screen lets you choose the initial roller/scorecard colour theme: **Classic**, **Forest**, **Ocean**, **Sunset**, or **Storm**. Changing themes later inside the roller window repaints both windows live; the chosen theme is also carried forward across all play-again paths.

---

## Determining Turn Order — Roll-Off

After Registration the **Roll-Off** screen (`RollOffDialog`) appears for games with 2 or more players.

Each player gets a card showing their name, animated dice roll, and total. Clicking **🎲 Roll for All Players** triggers a sequential animated roll for every player. Rolls are shown one player at a time with a short reveal delay between them for drama. The player with the highest total is placed first. **Ties are detected automatically and only the tied players re-roll** — the tie-resolution cycle repeats until all ties are broken. Each re-roll increments an internal cumulative score so that a player who wins a tie-break is ranked against their previous round's result.

Clicking **Start Scoring** skips the roll entirely and uses the Registration order directly.

---

## The Scorecard Window

`YahtzeeScorecard` is a `QMainWindow` that contains:

- A **dual clock bar** at the top — game elapsed time (⏱ left) and per-turn elapsed time (🎲 right). The turn timer turns orange at 60 s and red at 120 s.
- A **collapsible banner** below the clock that shows turn state, joker instructions, correction warnings, and confirmed-roll summaries.
- The **scoring table** — one row per scoring category, one column per player.
- A **bottom toolbar** with High Scores, Reset, Rules, and (in roller mode) 🎲 Open Roller.
- A **two-line status bar** showing current leader, scores, upper-section progress, last-scored category, hot-streak tracking, and the all-time high score.

### Scorecard Rows

| Row | Category | Type |
|---|---|---|
| 0–5 | Ones … Sixes | Dropdown (count of matching dice) |
| 6 | Sum | Calculated |
| 7 | Bonus (35) | Calculated — awarded when Sum ≥ 63 |
| 8 | Total Upper | Calculated |
| 9 | 3 of a Kind | Dropdown (dice total) |
| 10 | 4 of a Kind | Dropdown (dice total) |
| 11 | Full House | Dropdown (25 or 0) |
| 12 | Small Straight | Dropdown (30 or 0) |
| 13 | Large Straight | Dropdown (40 or 0) |
| 14 | Yahtzii | Dropdown (50 or 0) |
| 15 | Yahtzii Bonus (Count) | Button-counter widget |
| 16 | Chance | Dropdown (dice total) |
| 17 | Total Lower | Calculated (includes Yahtzii bonus × 100) |
| 18 | GRAND TOTAL | Calculated |

Rows 6, 7, 8, 17, 18 are non-editable and recalculate after every dropdown change. The **Yahtzii Bonus** cell (row 15) is a special compound widget — a `QLabel` showing the bonus count plus a `+` button — housed in a `QWidget` container set via `setCellWidget`.

---

## Playing a Turn

### Physical Dice Mode

Roll your own dice (up to three rolls per turn). Select a value from the dropdown in your column to claim a category. Choosing any value other than **—** on an unclaimed row claims it, triggers `recalc()` for that column, and calls `advance_to_next_player()`.

Upper section dropdowns offer counts 0–5. The app multiplies count × face value internally when storing the score. Lower section dropdowns offer the dice total or fixed value when the roll qualifies; 0 is always available to burn the category.

### Digital Roller Mode

1. The roller window (`YahtzeeRollerWidget` in `scorecard_mode=True`) opens automatically at the start of each turn, showing the current player's name in a banner.
2. Press **ROLL** — or hold the button to charge the power bar for a longer, more dramatic animation.
3. During rolling the scorecard is **locked** (table disabled).
4. Click any die to **hold** it; held dice do not participate in subsequent rolls and display a pulsing HELD label.
5. You may click **✔ Done — Use These Dice** after any roll (including the first). You are not required to use all 3 rolls.
6. Once confirmed, the roller hides automatically, the scorecard unlocks, and the turn banner updates to show the confirmed roll, the best-scoring hand name (e.g. "Full House"), and the point value.
7. Every unclaimed category for the active player is **colour-coded**: blue for valid scoring options, red (dark `#2D0F0F` background, `#F87171` text) for categories where the roll scores zero. Dropdowns are simultaneously restricted to only the valid options for the roll (see **Dropdown Restriction** below).
8. Clicking a category claims it, ends the turn, and automatically opens the roller for the next player.

If the roller window is accidentally closed the **🎲 Open Roller** button in the scorecard toolbar re-opens it. The player's roll state (dice values, held flags, rolls remaining) is fully preserved — closing the window only hides it (the `closeEvent` calls `event.ignore()` and `hide()` in scorecard mode).

### Dropdown Restriction (Digital Roller Mode)

After a roll is confirmed, `_update_upper_dropdowns()` and `_update_lower_dropdowns()` rebuild every combo-box for the active player to show only the options valid for the specific dice:

- **Upper section**: shows only the exact count of matching dice (e.g. three 4s → offers `3` only). Shows `0` if no dice match.
- **3 of a Kind / 4 of a Kind / Chance**: shows only the actual dice total when the roll qualifies; `0` otherwise.
- **Full House**: `25` if exactly 3+2, else `0`.
- **Small Straight**: `30` if any four consecutive values are present (1-2-3-4, 2-3-4-5, or 3-4-5-6), else `0`.
- **Large Straight**: `40` if all five are consecutive, else `0`.
- **Yahtzii**: `50` if five-of-a-kind, else `0`.

All restriction logic is re-evaluated inside `update_turn_ui()` and also accounts for Joker priority (see below).

---

## Dual Timer

Two `QElapsedTimer` instances run from `QMainWindow.__init__`. A single `QTimer` fires every second and updates both displays in `_tick_clock()`:

- **Game clock** (top left, grey `#94A3B8`): `HH:MM:SS` while over an hour, `MM:SS` otherwise.
- **Turn clock** (top right, theme-accent colour): resets to `00:00` at every `advance_to_next_player()` call. Turns orange at 60 s, red at 120 s.

---

## Status Bar

The two-row status bar at the bottom of the scorecard window provides a live commentary on the game state. Row 1 shows the current leader (or a solo score), all player scores separated by `·`, and the last-scored category. Row 2 shows upper-section progress for the active player (green if bonus secured, amber if still reachable, grey if missed), the best and all-time high scores, and an active-turn streak indicator (e.g. "🔥 Alice — 3 in a row" or "📊 Bob trails Alice by 42 pts").

---

## Scoring a Zero

You may score a **0** in any open category at any time. In digital roller mode, categories your roll cannot score are highlighted in red with only `0` available in the dropdown. Selecting `0` claims the category and ends the turn normally.

---

## Yahtzii Bonus & Joker Rules

If you roll a second (or subsequent) Yahtzii **and** your Yahtzii box already shows 50, click the **+** button in the Yahtzii Bonus row to register a bonus Yahtzii. Each click increments the bonus count by 1 and awards 100 points in `Total Lower`. Attempting to click `+` when the Yahtzii box does not show 50 produces a rule-violation warning dialog.

Clicking `+` sets `self.joker_active = True` on the scorecard, which immediately triggers a full `update_turn_ui()` repaint that applies **Joker priority colouring and dropdown locking**.

### Joker Priority (official rules)

When `joker_active` is True, the scorecard enforces the three-step priority. In digital roller mode with confirmed dice, `joker_roller_state` is computed from the dice to determine exactly which step applies:

1. **`"must_upper"`** — the matching Upper box (face value row) is still open. Only that row is highlighted; all other unclaimed primary categories are dimmed to `CLR_DISABLED` (`#0D1020`) and their dropdowns show `0` only.
2. **`"use_lower"`** — the matching Upper box is already claimed. Any open Lower category (`LOWER_SECTION_PRIMARY = [9, 10, 11, 12, 13, 16]`) is highlighted. Upper rows are dimmed.
3. **Physical dice mode (no confirmed roll)**: existing dim logic applies — categories outside Lower and Upper sections are dimmed.

The turn banner updates to a 🃏 message describing the exact priority step, including the specific face name (e.g. "🃏 Joker — Five Fours! ① Fours box is open — you must score there.").

---

## Correcting a Score

Before your turn is committed (i.e. before you claim an unclaimed box) you may correct a previously claimed box:

1. Open the dropdown for the claimed cell and select **—** to unclaim it. An amber warning banner appears: "⚠️ {Category} unclaimed — fill any box to replace it, then score your turn".
2. Fill that box with the correct value (or fill any other box as the replacement). A green confirmation banner shows which category was replaced.
3. Claim any **unclaimed** box to score your actual turn. The turn ends normally.

Corrections are tracked via `_correction_pending` and `_last_unclaimed_name` on the scorecard. They do **not** consume your turn; the turn only advances once a genuinely unclaimed box is claimed while `_correction_pending` is False.

---

## Roller Window in Detail

`YahtzeeRollerWidget` is a standalone `QWidget` (`scorecard_mode=False`) and also the integrated roller (`scorecard_mode=True`). Fixed size 520 × 720 px (standalone) or 520 × 780 px (scorecard mode, to accommodate the player banner).

### DieWidget

Each of the five dice is an individual `DieWidget(QWidget)` of fixed size 92 × 110 px. Die art is rendered via `QSvgRenderer` and `QPainter` into a `QPixmap`, then drawn with full custom `paintEvent` logic. Every frame the painter applies:

- **3-D Y-axis flip animation**: `spin_angle` drives a cosine-based x-scale. `show_front` flips between `visual_face` (pre-roll face) and `face` (landed face) based on angle modulus. A cubic ease-out snap tween (`snap_t`) straightens the die to upright at roll end.
- **Landing bounce**: `land_t` (0→1 over 260 ms) drives a three-phase squash-and-stretch: compress (0→20%), stretch + vertical arc (20→55%), damped ring-out (55→100%).
- **Drop shadow**: an elliptical shadow below the die whose width tracks x-scale during rolling, and whose height shrinks as the die rises during the bounce arc.
- **Held glow halo**: three concentric rounded-rect fills in the theme accent colour with layered alpha produce a soft glow; a pulsing border outline (sin-wave driven `pulse_t`) reinforces the hold state.
- **Background rect**: during rolling, the rect narrows with x-scale and fades in with `opacity`. When held, it is filled with a translucent tint of the accent colour.

### Animation Timers

Three `QTimer` instances run inside the roller:

| Timer | Interval | Purpose |
|---|---|---|
| `self.timer` | 16 ms (~60 fps) | Main animation tick during CHARGING and ROLLING states |
| `self._bounce_timer` | 16 ms | Landing bounce, snap tween, and held pulse after roll ends |
| `self._pulse_timer` | 30 ms (~33 fps) | Held-die glow pulse between rolls (IDLE state) |

### Roll Sequence

1. **IDLE → CHARGING**: `_start_charge()` fires on button `pressed`. A random roll duration between 600–1200 ms is chosen. The charge timer runs for 600 ms, filling the power bar.
2. **CHARGING → ROLLING**: `_start_roll()` rolls all non-held dice, records `_last_frame_time`, and starts the main tick.
3. **ROLLING**: each tick computes a per-die stagger offset and a global easing curve. Dice spin at different speeds driven by `spin_rate`, settle sequentially based on their individual `settle_after` threshold (a fraction of `roll_duration`). When a die settles, it fires its snap tween and landing bounce.
4. **ROLLING → IDLE**: when all dice have settled, `_finish_roll()` decrements `rolls_left`, shows the result frame, and updates button states. In scorecard mode, **✔ Done — Use These Dice** becomes enabled after the first completed roll.

### Roll History

The last 6 confirmed rolls (across all turns) are displayed in a scrolling history section at the bottom of the roller window. Each `RollerHistoryRow` widget shows the player name, five miniature SVG dice (18 × 18 px each), the hand name, and the point total.

### Quick-Score Helper (`_roller_score`)

A standalone function used both by the roller's result display and by the scorecard's turn banner to name and score a given set of five dice. It evaluates (in priority order): Yahtzii → Four of a Kind → Full House → Three of a Kind → Large Straight → Small Straight → One Pair → Chance.

---

## Themes

Five colour themes are defined in `_ROLLER_THEMES` and apply to both the roller and the scorecard (via `apply_roller_theme()`):

| Theme | Background | Accent | Bar gradient |
|---|---|---|---|
| Classic | `#1a1a2e` | `#e94560` | red → amber |
| Forest | `#1b2e1a` | `#4ade80` | green → light green |
| Ocean | `#0a1628` | `#38bdf8` | blue → sky |
| Sunset | `#2d1b0e` | `#fb923c` | orange → amber |
| Storm | `#1a1a1a` | `#a78bfa` | violet → lavender |

In **standalone roller mode** (not connected to a scorecard), theme buttons appear in a row below the PRO EDITION subtitle. In **scorecard mode** these buttons are hidden from the roller; theme switching is done exclusively via the roller's theme buttons as connected to the scorecard's `on_theme_changed` callback.

When a theme is applied, `apply_roller_theme()` on the scorecard derives table and unclaimed-cell colours by taking the theme background hex, converting to HSV via `colorsys`, clamping value and saturation, then converting back. This ensures the table colours are always a coherent darkened variation of the chosen background.

---

## Game Over

When all players have filled every primary category (`PRIMARY_CATEGORIES = [0,1,2,3,4,5,9,10,11,12,13,14,16]`), `check_game_over()` collects and sorts scores, saves the winner to `yahtzee_highscores.json` (top 10 by score), and shows `GameOverDialog`.

### Solo vs. Multi-Player Game Over

For a **solo** game, the dialog shows the player's score and offers three options: **▶ Play Again**, **📋 New Game / Change Players**, and **✖ Quit**. The "Roll for Order" button is absent.

For a **multi-player** game, the full podium is shown (🥇 🥈 🥉 for top three) and four options are offered:

| Option | Behaviour |
|---|---|
| ▶ Same Order | Immediate rematch — skips Registration and Roll-Off entirely; loops in the inner `while True` |
| 🎲 Roll for Order | Returns to the Roll-Off screen with the same players |
| 📋 New Game / Change Players | Returns to Registration with names pre-filled and last theme pre-selected |
| ✖ Quit | Calls `sys.exit(0)` |

---

## High Scores

Scores are stored in `yahtzee_highscores.json` as a JSON array sorted descending, capped at 10 entries. Each entry has `name`, `score`, and `date` (`YYYY-MM-DD`). The all-time high is loaded at scorecard startup via `_load_alltime_high()` and shown in the status bar. The **High Scores** button in the toolbar opens a modal dialog with a styled 3-column table.

---

## Internal Architecture Notes

### Key Classes

| Class | Role |
|---|---|
| `DieWidget` | Custom `QWidget` per die — full 3-D animation via `paintEvent` |
| `RollerHistoryRow` | Compact history row widget (player + mini dice + result) |
| `YahtzeeRollerWidget` | Animated 5-die roller; standalone or scorecard-integrated |
| `RulesDialog` | Scrollable HTML rules reference dialog |
| `PlayerSetupDialog` | Registration: player names, roller toggle, theme picker |
| `RollOffDialog` | Animated per-player roll-off with tie-break re-roll logic |
| `GameOverDialog` | Podium display and play-again options |
| `YahtzeeScorecard` | Main `QMainWindow`: table, timers, all scoring & turn logic |

### State Flow Between Roller and Scorecard

The scorecard creates the roller lazily on first use and wires three callbacks:

- `on_turn_done(dice)` → `_on_roller_done()`: hides the roller, stores `_roller_dice`, unlocks the table, updates the banner.
- `on_window_hidden()` → `_on_roller_hidden()`: re-enables the Open Roller button so the player can recover without losing roll state.
- `on_theme_changed(name)` → `apply_roller_theme()`: repaints the scorecard to match.

`advance_to_next_player()` clears `_roller_dice = None` and opens a fresh roller for the next player. If the roller window already exists, `prepare_for_player()` calls `_new_round()` to blank and reset the dice for the new player.

### Signal Guard

All dropdown changes flow through `handle_dropdown()`. A boolean flag `self._is_updating` prevents re-entrant signals during programmatic combo-box rebuilds (which occur in `_update_upper_dropdowns()` and `_update_lower_dropdowns()` on every `update_turn_ui()` call).

---

## Scoring Reference

### Upper Section (Ones – Sixes)

Select the count of matching dice. The app multiplies by face value: three 4s → select 3 → scores 12.

Score **63 or more** in the upper section total to earn the **+35 bonus**.

### Lower Section

| Category | Row | Score condition |
|---|---|---|
| 3 of a Kind | 9 | ≥ 3 of same face → total all 5 dice |
| 4 of a Kind | 10 | ≥ 4 of same face → total all 5 dice |
| Full House | 11 | Exactly 3+2 → fixed 25 pts |
| Small Straight | 12 | Any 4 consecutive values → fixed 30 pts |
| Large Straight | 13 | All 5 consecutive → fixed 40 pts |
| Yahtzii | 14 | Five of a kind → fixed 50 pts |
| Yahtzii Bonus | 15 | Each additional Yahtzii (Yahtzii box = 50) → +100 pts per click |
| Chance | 16 | Any roll → total all 5 dice |

Fixed-score categories (Full House, Small/Large Straight, Yahtzii) offer only the fixed value or 0 in their dropdowns — no manual number entry.
