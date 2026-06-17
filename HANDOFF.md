# Beat Matchers — Project Handoff

> Read this first in a new session to get full context without re-deriving anything.
> Quick start for a new chat: **"Read bpm-match-game/HANDOFF.md, then <task>."**

## What it is
A browser game. A popular song plays at the **wrong tempo**; you drag a fader to
restore it to its real speed **by ear** (no meter, no hint). Single-file web app,
real 30-second song previews from Apple's free iTunes API. Live at **beatmatchers.net**.

## Where everything lives
- **Code:** `/Users/jonahmacbook/Desktop/codeProjects/bpm-match-game/`
  - `index.html` — the **entire game** (HTML + CSS + vanilla JS, ~2200 lines). No build step.
  - `privacy.html` — privacy policy page (AdSense prereq).
  - `README.md`, `HANDOFF.md` (this file), `CNAME` (`beatmatchers.net`), `.gitignore`.
  - Icon/launcher artifacts (`BPMicon.*`, `icon_master.png`, `BPMicon.iconset/`) are **gitignored**.
- **GitHub repo:** `Putlo-byte/Beat-Matchers` (was `bpm-matc`; the old name **redirects**, so the existing `origin` remote still pushes fine). https://github.com/Putlo-byte/Beat-Matchers
- **Live site:** **https://beatmatchers.net** (HTTPS done + enforced). Also `https://putlo-byte.github.io/bpm-matc/` redirects to it.
- **Git identity:** name `Putlo-byte`, email `jmbyars05@gmail.com`. End commits with:
  `Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>`
- **gh CLI** authed as `Putlo-byte` (had to `sudo chown -R $(whoami) ~/.config` once to fix a root-owned config dir).

## How to make changes & deploy
```bash
export PATH="/opt/homebrew/bin:$PATH"
cd /Users/jonahmacbook/Desktop/codeProjects/bpm-match-game
# edit index.html ...
git add index.html
git -c user.name="Putlo-byte" -c user.email="jmbyars05@gmail.com" commit -m "msg"
git push                 # GitHub Pages auto-rebuilds in ~1 min
```
- **Local preview:** `.claude/launch.json` has a `bpm-game` config = `python3 -m http.server 5190 --directory bpm-match-game`. Use the Claude Preview tools (preview_start "bpm-game", preview_eval, preview_screenshot). Preview runs **Chromium** — can't test Safari-specific behavior there. NOTE: `getComputedStyle` in headless preview can be stale for CSS-variable-dependent props; trust **screenshots** as ground truth.
- **Bash note:** git/gh/network commands usually need `dangerouslyDisableSandbox: true`.

## ⚠️ OPEN ITEMS / TODO
1. **Firebase Auth authorized domain** — for Google sign-in (Ranked mode) to work **on beatmatchers.net**, that domain must be in Firebase → Authentication → Settings → **Authorized domains**. (`localhost` + `putlo-byte.github.io` are there; `beatmatchers.net` was being added — verify it's listed; otherwise sign-in errors with `auth/unauthorized-domain`.) Party mode does NOT need auth.
2. **AdSense not live yet** — `privacy.html`, Consent Mode v2 (in `<head>`), and the cookie banner are in place, but the AdSense `<script>` loader is **commented out** with a `ca-pub-XXXX` placeholder. After AdSense approves the site: uncomment the loader, set the real publisher ID, add ad units.
3. (Optional polish) The cookie banner is a hardcoded **dark** bar (`#bm-cc`) — doesn't follow the light/dark theme. Functional; could be themed.
4. (Done) HTTPS cert — approved, Enforce HTTPS on. (Done) Ranked rename, Party mode, dark/light toggle, mobile layout.

## Modes (3 + theme)
- **Singleplayer** — solo; artist-pool playlists (Big Hits / 2010s / Throwbacks / Pop Icons). Score, streak, fireworks on a perfect.
- **Ranked** — Google-auth 1v1 (formerly "Multiplayer"); matchmaking queue, songs from the **live charts**, wins/accuracy leaderboard.
- **Party** — link-based lobby (`?party=CODE`), **up to 8 players, NO login** (pick a username). Host picks the song pool (charts / a category / **paste a custom song list** — Spotify/Apple *links* can't be read, only pasted song names). Synced 5 rounds; reveal bar shows **ALL players' guesses with the top 3 highlighted**; final standings. Firebase node `/parties/{CODE}`. Stall-watchdog auto-advances if a player never locks. **Host leaving stalls the party** (no host migration — known limitation).
- **Theme toggle** — small **sun/moon icon button** in the header top-right corner (visible everywhere). `localStorage.bm_theme` (default light). Dark theme = `html[data-theme="dark"]` CSS-variable overrides.

## External services & config
- **iTunes / Apple APIs (no key):**
  - Search (singleplayer + custom party songs): JSONP `itunesSearch(term, limit)`.
  - Lookup by id (chart previews): JSONP `itunesLookup(id)`.
  - **Live charts (Ranked + Party):** `fetch('https://itunes.apple.com/us/rss/topsongs/limit=200/json')` (CORS-OK; ~80 songs). `loadTopChart()` / `resolveChartTrack()`.
  - Preview audio fetched as ArrayBuffer for BPM detection (CORS works on Apple preview URLs).
- **Firebase** (project **`bpmgame`**), compat SDK v10.12.0 (app + database + auth):
  - `FIREBASE_CONFIG` is in `index.html` (apiKey `AIzaSyCKYOWw-...`, databaseURL `https://bpmgame-default-rtdb.europe-west1.firebasedatabase.app`). Web keys are **public by design** — safe to commit.
  - **Realtime DB rules:** `{ "rules": { ".read": true, ".write": true } }` (public — fine for a casual game; a known anti-cheat weakness). Leaderboard reads use `orderByChild('wins')` — add `".indexOn": ["wins"]` on `/players` for efficiency (works without it, just slower + a console warning).
  - **Google sign-in** enabled (Ranked only).
- **Custom domain:** `beatmatchers.net` at **Cloudflare** (DNS-only / grey cloud A+AAAA → GitHub Pages `185.199.108-111.153`, `www` CNAME → `putlo-byte.github.io`). HTTPS issued + enforced.

## Architecture (one IIFE in index.html)
- **Audio:** an `<audio>` element; tempo via `player.playbackRate` + `preservesPitch=true` (changes BPM, keeps pitch). Slider 0–100 → rate `rateFromPos(pos)=1+(pos-targetPos)/100`; hidden fractional `targetPos` = original tempo (rate 1.0). **Safari** stalls audio on every rate change → on Safari the tempo change is debounced (apply on settle/release); other browsers update live.
- **Mode switching:** `mode` = `null|'single'|'multi'|'party'`. `show(el,on)` is null-safe. Screens toggled: `#menuView`, `#queueView`, `#leaderboardView`, `#partyJoin`, `#partyLobby`, `#play` (shared stage/slider/timer). Entry fns: `startSingle()`, `startMulti()`, `createParty()`/`openPartyName(code)`, `showLeaderboard()`, `showMenu()` (also `leaveParty()` + `cleanupMatch()`).
- **Scoring** (`scoreFor(err)`, `err = |sliderPos - targetPos|` ≈ % off, `accuracy = 100-err`):
  - `err≤10`: `round(250*sqrt(1-err/10))` (96%≈194, 92%≈112 — steep, one good guess can swing a match)
  - `10<err≤30`: 0 pts; `err>30`: penalty `-min(60, round((err-30)*2))`
  - Singleplayer adds streak bonus + canvas fireworks when `err≤1`. Results show accuracy to 2 decimals, **green if scored / red if not**, plus detected BPM.
- **BPM detection:** `startBpmDetect(url)` → fetch+decode preview, `computeBPM()` (OfflineAudioContext low/high-pass + peak/interval histogram). Estimate (`≈`); can be octave-off on syncopated songs.
- **Ranked (Firebase):** single `/queue` slot transaction pairs 2; `/matches/{id}` holds `players` (uid→{name,photo}), `songs[]`, `currentRound`, `rounds/{r}/guesses/{uid}={pos,acc,pts}`, `present/{uid}`. Reveal when both locked; host advances; stats → `/players/{uid}` (`wins,losses,games,accSum,accCount`). Opponent-left guarded by `oppSeen` (avoids join-race false positive). **You=green, opponent=red** (each client labels its own data "You").
- **Party (Firebase):** `/parties/{CODE}` = `host`(pid), `state` (`lobby|playing|finished`), `settings`(source/custom), `players/{pid}`={name,present}, `songs[]`, `currentRound`, `rounds/{r}/guesses/{pid}`, `rounds/{r}/revealed`. Identity = `localStorage.bm_pid` (no auth). Host resolves songs (`resolvePartySongs`), sets `state:playing`. Reveal driven by `rounds/{r}/revealed` (host sets it when all present locked OR a watchdog fires). `partyTotals()` ranks; multi-marker reveal bar; final standings local to the party.
- **Leaderboard:** top-N by wins (30s in-memory cache `_lbCache`); shows rank · name · **avg accuracy** · wins · W-L.
- **Input:** Space = play/pause; arrows = ±0.5; detent tick on drag; **back button** (`#backBtn`, header left) returns to menu.

## Design / styling
Single skin in use: **minimalist Teenage-Engineering/Braun** flat look (a `:root`/MINIMAL SKIN block that overrides an earlier, now-dead "RETRO SKIN"). Off-white panel (`--panel`), orange accent (`--accent #f2451e`), **Space Grotesk** (UI) + **Space Mono** (numerals), no gloss/glow, flat mixer fader, **custom SVG line icons** (no emoji anywhere). Dark theme via `html[data-theme="dark"]` variable overrides + a few literal overrides (lock/active-chip buttons, fader track, lbRow.me). Mobile breakpoint `@media (max-width:560px)`: mode cards stack full-width (icon beside name), toggle pinned top-right, no overflow down to 320px.

## Ads prerequisites (in place, AdSense not enabled)
- `privacy.html` (cookies/advertising/your-choices, links to Google opt-outs).
- **Block A** in `<head>`: Google Consent Mode v2 defaults = denied; reads `localStorage.bm-consent`.
- **Block B** before `</body>`: cookie banner (`#bm-cc`) — Accept/Reject, persists in `localStorage.bm-consent`, floating "Cookie settings" reopen button. AdSense loader commented out (`ca-pub-XXXX`).
- Verified: banner shows on first visit, persists, privacy page loads.

## Known limitations / caveats
- **BPM detection is an estimate** (octave errors on some songs) — labeled `≈`.
- **Anti-cheat:** client-written scores + public DB rules — a determined user could fake stats. Fine for friends.
- **First play needs one user gesture** (browser autoplay rule); after that, auto-plays.
- **Party host leaving stalls the party** (no host migration).

## Possible next features (discussed, not built)
- Theme the cookie banner to match light/dark.
- Rewarded "watch for a hint" ad or a donate button.
- Tighter BPM octave folding / hide-when-low-confidence.
- A "Top Charts" playlist for singleplayer.
- Auth-restricted DB rules for anti-cheat.
