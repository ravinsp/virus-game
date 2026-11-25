# Viral Duel
2-player Virus card game clone made with AI.

Live page: https://ravinsp.github.io/virus-game/

## Running locally

There is no build step and no dependencies. Open `index.html` in a browser, or serve the folder:

```sh
python -m http.server 8000
```
Then open http://localhost:8000 in a browser.

The only external resources are the Baloo 2 font from Google Fonts and, for online play only, the [Trystero](https://github.com/dmotz/trystero) library from jsDelivr (loaded when someone opens the online menu).

## Online play

"2 players, online" connects two browsers directly (WebRTC) with Trystero. Trystero finds the other player through free public Nostr relays, so there is no server to deploy and no API key.

- **Host:** taps *Create a game* and types a room name (up to 8 letters or numbers), then tells their friend the name.
- **Guest:** taps *Join game* and types the same room name.
- **Room names:** the last name typed is saved in `localStorage` under `viralDuelRoom` and fills in the box next time, for both Create and Join. If two people both create a game with the same name, both are sent back to pick another name.
- **Sync:** the host picks a random seed. Both browsers shuffle with that seed, so they hold the same game. After that, each side sends only its moves, which the other side replays with the same animations as the computer's moves.
- **Rematch:** either player can ask for one; the host deals it, and the first player alternates.
- **Leaving:** if one player leaves, the other is told. If the guest returns to the same room, the host starts a new game.

Online play only works on a secure page: `https://` (like GitHub Pages) or `http://localhost`. Trystero uses the browser's `crypto.subtle`, which browsers turn off on plain `http://` addresses such as `http://192.168.1.4:8000`. To test on a phone, use the GitHub Pages site. To test on one computer, use two browser windows (one private) on localhost.

Limits: it's built for casual play between friends. Both browsers hold the full game state, so someone using the browser's developer tools could see the other player's hand. Very strict networks (some corporate or mobile networks) can block direct WebRTC connections, because there is no TURN relay.

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
| **Win stats** | Win counts per mode, saved in the browser's `localStorage` under `viralDuelStats` as `{ai: [you, computer], duo: [player 1, player 2], net: [you, rival]}`. `addWin()` records a win, `resetStats(mode)` clears one mode, `statsCard()` draws a mode's panel. |
| **Light/dark mode** | Follows the device by default. The switch under the main menu sets `data-theme` on `<html>` and saves it in `localStorage` under `viralDuelTheme`; choosing the same mode as the device clears it, so the game follows the device again. A small script in `<head>` applies the saved choice before the page draws. |
| **Game state** | The global state object `S`, deck building (`buildDeck`, 68 cards), `newGame`, `draw`, and rule helpers like `canAttach`, `immune`, `healthy` and `transplantOK`. `shuffle()` uses `rnd()`, which is seeded in online games so both browsers deal the same cards. |
| **Valid targets** | `targets()` works out which organs or drop zones the selected card can be played on, plus the hint text. |
| **Actions** | `playOnSlot()` (virus, medicine, transplant, organ thief), `useCard()` (organs, contagion, latex glove, medical error) and `endTurn()` (refill, win check and recording, switch player). In online games, each of these calls `share()` to send the move to the rival. |
| **Computer player** | Scores a board with `val()` / `evalState()`, lists every legal move with `moves()`, tries each one on a copy of the state (`simulate`) and picks the best. If nothing helps, it discards. |
| **Computer move animations** | `fly()` moves card art across the screen; `animatePlan()` shows the AI's move before it is applied. `flipFrom()` slides organs to their new spots after each redraw (FLIP technique). |
| **Online play** | `netOpen()` joins a Trystero room named by the room name, `netDeal()` (host) starts a game or rematch, `netGet()` handles messages (`start`, `move`, `again`, `bye`) and `remoteTurn()` animates and applies the rival's move. Discards and skips go through `aiExec()` so they are shared like other moves. |
| **Render** | `render()` rebuilds the screen from `S` for each screen: `start`, `rules`, `stats`, `online`, `lobby`, `pass`, `play`, `ai` (the computer's or online rival's turn), `win`, `gone` (online rival left). |
| **Input** | `onAct()` handles every button and tap, keyed by the element's `data-a` attribute. |
| **Long-press tooltip** | Holding a card shows its name, effect (`info()`) and organ status. |
| **Drag and drop** | Pointer events drag a hand card onto an organ, a zone, or the discard bin. |

### Game state (`S`)

| Field | Meaning |
| --- | --- |
| `ai` | `true` for vs-computer mode. |
| `online`, `seat`, `gid`, `rs`, `turn` | Online games: this browser's player (0 host, 1 guest), the game's seed, the shuffle generator's state, and a turn counter used to check incoming moves. |
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
