# Beat Matchers — Project Handoff

> Read this first in a new session to get full context without re-deriving anything.

## What it is
A browser game. A popular song plays at the **wrong tempo**; you drag a fader to
restore it to its real speed **by ear** (no meter, no hint). Single-file web app.
Real 30-second song previews stream from Apple's free iTunes API.

## Where everything lives
- **Code:** `/Users/jonahmacbook/Desktop/codeProjects/bpm-match-game/`
  - `index.html` — the **entire app** (HTML + CSS + vanilla JS, ~1700 lines). No build step.
  - `README.md`, `HANDOFF.md` (this file), `CNAME` (`beatmatchers.net`), `.gitignore`
  - Icon/launcher artifacts (`BPMicon.*`, `icon_master.png`, `BPMicon.iconset/`) are **gitignored**.
- **GitHub repo:** https://github.com/Putlo-byte/bpm-matc  *(note: repo name is `bpm-matc`, missing the "h")*
- **Live site:** https://beatmatchers.net  (also `https://putlo-byte.github.io/bpm-matc/`, which redirects to the custom domain)
- **Git identity for commits:** name `Putlo-byte`, email `jmbyars05@gmail.com`.
  End commit messages with: `Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>`
- **gh CLI** is authenticated as `Putlo-byte` (was broken because `~/.config` was root-owned; fixed with `sudo chown -R $(whoami) ~/.config`).

## How to make changes & deploy
```bash
export PATH="/opt/homebrew/bin:$PATH"
cd /Users/jonahmacbook/Desktop/codeProjects/bpm-match-game
# edit index.html ...
git add index.html
git -c user.name="Putlo-byte" -c user.email="jmbyars05@gmail.com" commit -m "msg"
git push                 # GitHub Pages auto-rebuilds in ~1 min
```
**Local preview:** `.claude/launch.json` has a `bpm-game` config = `python3 -m http.server 5190 --directory bpm-match-game`. Use the Claude Preview tools (preview_start "bpm-game", preview_eval, preview_screenshot) to test. The preview runs Chromium — Safari-specific behavior can't be tested there.
**Bash note:** `git`/`gh`/network commands often need `dangerouslyDisableSandbox: true`.

## ⚠️ OPEN ITEMS / TODO
1. **HTTPS cert pending.** Site serves over HTTP only; GitHub hasn't issued the Let's Encrypt cert yet (`cert state: None`). DNS + CAA are correct, nothing blocks it.
   - Check: `gh api repos/Putlo-byte/bpm-matc/pages` → look at `https_certificate.state`.
   - Fix if stuck: GitHub → Settings → Pages → delete custom domain → Save → wait 1 min → re-enter `beatmatchers.net` → Save. (Forces re-validation.)
   - Once cert exists, enable Enforce HTTPS: `gh api --method PUT repos/Putlo-byte/bpm-matc/pages -F https_enforced=true`
2. **Firebase Auth authorized domain.** For Google sign-in to work **on beatmatchers.net**, that domain must be added in Firebase → Authentication → Settings → **Authorized domains**. (`putlo-byte.github.io` and `localhost` are already authorized; `beatmatchers.net` likely still needs adding — verify.)
3. `beatmatchers.net.zone` (in the parent `codeProjects/` folder) was the Cloudflare DNS import file — can be deleted now.

## External services & config
- **iTunes / Apple APIs (no key needed):**
  - Search (singleplayer songs): JSONP via `itunesSearch(term, limit)` → `https://itunes.apple.com/search?...&callback=`
  - Lookup by id (chart previews): JSONP via `itunesLookup(id)` → `https://itunes.apple.com/lookup?id=...&callback=`
  - **Live charts (multiplayer songs):** `fetch('https://itunes.apple.com/us/rss/topsongs/limit=200/json')` (CORS-OK; returns ~80 songs). Function `loadTopChart()` / `resolveChartTrack()`.
  - Preview audio fetched as ArrayBuffer for BPM detection (CORS works on Apple preview URLs).
- **Firebase** (project **`bpmgame`**), compat SDK v10.12.0 (app + database + auth):
  - Config is in `index.html` → `const FIREBASE_CONFIG = {...}` (apiKey `AIzaSyCKYOWw-...`, databaseURL `https://bpmgame-default-rtdb.europe-west1.firebasedatabase.app`). These web keys are **public by design** — safe to commit.
  - **Realtime Database rules** (set in Firebase console): `{ "rules": { ".read": true, ".write": true } }` (public — fine for a casual game; a known anti-cheat weakness).
  - **Google sign-in** is enabled. Auth required for multiplayer.
- **Custom domain:** `beatmatchers.net`, registered at **Cloudflare**. DNS = 4 A records (`185.199.108-111.153`), 4 AAAA (`2606:50c0:8000-8003::153`), `www` CNAME → `putlo-byte.github.io`. All set to **DNS only (grey cloud)** so GitHub can issue the cert. Cloudflare SSL/TLS mode should be **Full** if proxy ever turned on.

## Architecture (it's one IIFE in index.html)
- **Audio:** an `<audio>` element; tempo via `player.playbackRate` with `preservesPitch=true` (changes BPM, keeps pitch). The slider position 0–100 maps to rate via `rateFromPos(pos) = 1 + (pos-targetPos)/100`; the hidden `targetPos` (fractional) = original tempo (rate 1.0).
- **Modes:** `mode` = `null|'single'|'multi'`. Views toggled with `show(el, on)` (null-safe). Screens: `#menuView`, `#queueView`, `#leaderboardView`, `#play` (shared by SP & MP). `showMenu()`, `startSingle()`, `startMulti()`, `showLeaderboard()`.
- **Scoring:** `scoreFor(err)` where `err = |sliderPos - targetPos|` (≈ % tempo off), `accuracy = 100 - err`:
  - `err <= 10`: `round(250 * sqrt(1 - err/10))` → 96%≈194, 92%≈112 (steep, generous; one good guess can swing a match)
  - `10 < err <= 30`: 0 points
  - `err > 30`: penalty `-min(60, round((err-30)*2))`
  - Singleplayer adds a streak bonus and fires fireworks (canvas) when `err <= 1`.
  - Results show accuracy to 2 decimals, **green if you scored / red if not**, plus detected BPM.
- **BPM detection:** `startBpmDetect(url)` → fetch+decode the preview, `computeBPM()` runs an OfflineAudioContext low/high-pass + peak/interval histogram. Shown as an estimate (`≈`), can be octave-off on syncopated songs.
- **Multiplayer (Firebase Realtime DB):**
  - Matchmaking: a single `/queue` slot; transaction pairs two players. Waiter learns its match via `/playerMatch/{uid}`. Host resolves 5 **chart** songs and writes `/matches/{id}` (`players` map of uid→{name,photo}, `songs[]`, `state`, `currentRound`, `rounds/{r}/guesses/{uid}={pos,acc,pts}`, `present/{uid}`).
  - Both clients subscribe to the match node (`onMatchState`). Round reveals when **both** guesses are in; host advances `currentRound` after ~5s; `state:'finished'` shows the winner.
  - Identity = Google `uid`. Opponent's real name shown. **You are always green, opponent red** (each client labels its own data "You").
  - Stats written to `/players/{uid}`: `wins, losses, games, accSum, accCount` (avg accuracy = accSum/accCount). `recordResult(outcome, m)`.
  - Opponent-left detection guarded by `oppSeen` to avoid a join-race false positive.
- **Leaderboard:** reads all `/players`, sorts by wins client-side (no DB index needed), shows rank · name · **avg accuracy** · wins · W-L. Has a small in-memory cache (`_lbCache`/`renderLeaderboard`).
- **Input niceties:** Spacebar = play/pause; arrow keys = ±0.5 fine-tune; detent "tick" click while dragging (throttled). **Back button** (`#backBtn`, header top-left) returns to menu during a game.

## Design / styling
- Two skins exist in `<style>`. The **second, appended block "MINIMAL SKIN"** wins on source order — this is the current look: flat **Teenage Engineering / Braun** style. Off-white panel (`--panel #efece5`), single orange accent (`--accent #f2451e`), **Space Grotesk** (UI) + **Space Mono** (numerals), no gloss/glow, flat mixer-style fader, custom **SVG line icons** (no emoji anywhere), numbered leaderboard ranks.
- The earlier "RETRO SKIN" (80s silver/black hi-fi receiver) is still in the file but fully overridden; could be deleted for cleanliness.
- Mobile breakpoint `@media (max-width:560px)`: stage stacks, buttons full-width, etc.

## Known limitations / caveats
- **BPM detection is an estimate** (octave errors on some songs) — labeled `≈`.
- **Anti-cheat:** scores/wins are client-written with public DB rules; a determined user could fake stats. Fine for friends; would need auth-restricted rules + server validation for real competition.
- **First play needs one user gesture** (browser autoplay rule); after that, songs auto-play.
- **Safari** stalled audio on every `playbackRate` change during drag → fixed by debouncing the tempo change on Safari only (`isSafari`), applied on settle/release.

## Misc
- Desktop launcher: `~/Desktop/BPM Match.app` (AppleScript applet that opens the local index.html). Custom icon built via `iconutil`. Still named "BPM Match" — could rename to "Beat Matchers".
- The app was renamed from "BPM Match" → **Beat Matchers** (title, brand, README). Repo/URL still use `bpm-matc`.

## Possible next features (discussed, not built)
- Rewarded-ad "watch for a hint" or a donate button (monetization).
- Tighter BPM octave folding or hide-when-low-confidence.
- A "Top Charts" playlist for singleplayer too.
- Auth-restricted DB rules for anti-cheat.
