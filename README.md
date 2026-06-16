# 🎚️ Beat Matchers

A browser game: a popular song plays at the **wrong tempo**, and you drag a slider
to restore it to its real speed — by ear. No meter, no hints; you trust your memory
of how the track actually goes.

**Play:** once hosted, open the GitHub Pages URL (see below).

## How it works
- Real 30-second song previews stream from the free **Apple / iTunes Search API** (no key needed).
- The slider changes playback tempo (pitch preserved), so it's a true BPM shift.
- Each round you have **20 seconds** before it auto-locks.
- **Scoring:** a steep, generous curve rewards high accuracy — up to ~250 points
  (96% ≈ 194, 92% ≈ 112), so one great guess can swing a match. 70–90% = no points,
  below 70% = penalty, and ≤1% off triggers 🎆 fireworks. Results show your accuracy to
  two decimals (e.g. `97.68%`) — green if you scored, red if you didn't.
- The song pool is effectively endless (pulled live from a rotating set of popular artists)
  and rarely repeats.

## Tech
Single static `index.html` — HTML, CSS, and vanilla JS with the Web Audio API.
No build step, no backend. Hostable on any static host.

## Hosting (GitHub Pages)
1. Push this repo to GitHub.
2. **Settings → Pages → Source: Deploy from a branch → `main` / `(root)` → Save.**
3. Live in ~1 minute at `https://<username>.github.io/<repo>/`.
