# Viral Duel
2-player Virus card game clone made with AI.

Live page: https://ravinsp.github.io/virus-game/

## Running locally

There is no build step and no dependencies. Open `index.html` in a browser, or serve the folder:

```sh
python -m http.server 8000
```
Then open http://localhost:8000 in a browser.

The only external resource is the Baloo 2 font from Google Fonts.

## Codebase structure

```
virus-game/
├── index.html    # The whole game: markup, CSS and JavaScript in one file
├── favicon.png   # Browser tab icon
└── README.md
```

Everything lives in [index.html](index.html). The script is plain JavaScript (no framework) and is split into sections marked with `// ---- Section ----` comments.

### Styles (`<style>`)

- Color tokens on `:root`, with a dark theme via `prefers-color-scheme` or `data-theme="dark"`.
- Mobile-first layout capped at 520px wide: rival zone, middle bar (deck, hint, discard bin), your zone, hand, action buttons.
- Card states: `.infected`, `.vaccinated`, `.immune`, `.target` (valid drop spot), `.sel` (selected hand card).
- Drag, fly-in animation, tooltip and full-screen overlay styles.

### Script sections

| Section | What it does |
| --- | --- |
| **Card data** | Colors, organ names (`ORG`) and treatment cards (`TREAT`). |
| **Card art** | All card art is SVG generated in code. Shared gradients are injected once; `ORGAN_ART`, `VIRUS_ART`, `MED_ART` and `TREAT_ART` draw the pictures, `frame()` draws the card face, and `cardSVG(card)` puts it together. `mini()` draws the small virus/medicine chips shown on organs (`MINI_VIRUS`, `MINI_MED`). |
| **Win stats** | Win counts per mode, saved in the browser's `localStorage` under `viralDuelStats` as `{ai: [you, computer], duo: [player 1, player 2]}`. `addWin()` records a win, `resetStats(mode)` clears one mode, `statsCard()` draws a mode's panel. |
| **Game state** | The global state object `S`, deck building (`buildDeck`, 68 cards), `newGame`, `draw`, and rule helpers like `canAttach`, `immune`, `healthy` and `transplantOK`. |
| **Valid targets** | `targets()` works out which organs or drop zones the selected card can be played on, plus the hint text. |
| **Actions** | `playOnSlot()` (virus, medicine, transplant, organ thief), `useCard()` (organs, contagion, latex glove, medical error) and `endTurn()` (refill, win check and recording, switch player). |
| **Computer player** | Scores a board with `val()` / `evalState()`, lists every legal move with `moves()`, tries each one on a copy of the state (`simulate`) and picks the best. If nothing helps, it discards. |
| **Computer move animations** | `fly()` moves card art across the screen; `animatePlan()` shows the AI's move before it is applied. `flipFrom()` slides organs to their new spots after each redraw (FLIP technique). |
| **Render** | `render()` rebuilds the screen from `S` for each screen: `start`, `rules`, `stats`, `pass`, `play`, `ai`, `win`. |
| **Input** | `onAct()` handles every button and tap, keyed by the element's `data-a` attribute. |
| **Long-press tooltip** | Holding a card shows its name, effect (`info()`) and organ status. |
| **Drag and drop** | Pointer events drag a hand card onto an organ, a zone, or the discard bin. |

### Game state (`S`)

| Field | Meaning |
| --- | --- |
| `ai` | `true` for vs-computer mode. |
| `deck`, `discard` | Draw pile and discard pile. The discard is reshuffled when the deck runs out. |
| `pl[0]`, `pl[1]` | Players: `hand` (cards), `body` (organs), `skip` (misses next turn). |
| `body[n]` | An organ slot: `{card, v: [viruses], m: [medicines]}`. Two medicines = immune. |
| `cur` | Whose turn it is (0 or 1). In AI mode, player 1 is the computer. |
| `sel`, `tp`, `mode`, `dsel` | UI state: selected hand card, first organ picked for a transplant, discard mode, cards picked to discard. |
| `screen` | Which screen `render()` shows. |
| `log` | Last action message. |
| `confirm` | On the stats screen, which mode's reset button is waiting for a second tap. |

### Cards

| Type | Count | Notes |
| --- | --- | --- |
| Organs | 5 per color + 1 wild | Heart (red), Brain (blue), Stomach (green), Bone (yellow) |
| Viruses | 4 per color + 1 wild | |
| Medicine | 4 per color + 4 wild | |
| Treatments | 10 | Transplant ×3, Organ thief ×3, Contagion ×2, Latex glove ×1, Medical error ×1 |
