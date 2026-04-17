# World Explorer — A Cartographer's Quiz

A single-page geography game. A country is highlighted on an old-world-style map; you name it (and, for bonus points, name its capital). Three tries per answer before the map reveals it. Answer by typing or by voice.

## Play

Open `index.html` in a browser, or visit the GitHub Pages site.

## Controls

- **Type** your answer and press **Enter** to submit
- Click the **🎤** to answer by voice (Chrome / Safari / Edge)
- **Space** advances to the next country after a reveal, or skips the capital bonus
- Filter by region from the top-right selector

## Scoring

| Outcome | Points |
|---|---|
| Country — 1st try | 10 |
| Country — 2nd try | 7 |
| Country — 3rd try | 4 |
| Country — revealed | 0 |
| Capital bonus — 1st try | +5 |
| Capital bonus — 2nd try | +3 |
| Capital bonus — 3rd try | +1 |

Streak counts consecutive 1st-try country answers. Best streak is saved locally.

## Tech

- D3 v7 + TopoJSON (`world-atlas` countries-110m)
- Natural Earth projection
- Web Speech API for voice input
- Cormorant Garamond + Source Serif 4 (Google Fonts)
- No build step — single `index.html`

## Deploy to GitHub Pages

1. Create a new GitHub repo (e.g. `mapgame`).
2. Push this folder to `main`.
3. In the repo settings → Pages, set source to **main branch / root**.
4. The site will be at `https://<username>.github.io/<repo>/`.
