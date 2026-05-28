# Leave Her Johnny — Project Handoff

> **How to use this doc:** Paste this file into a new Claude chat **together with `index.html`** (attach the file). This doc is the *map and the rules*; `index.html` is the *source of truth*. Line numbers here are approximate and drift as the file changes — **always locate code by searching for the function/identifier name**, not the line number.
>
> Baseline at handoff: commit `53c5663`, `index.html` ≈ 8,800 lines. Platform: developed on **Windows, no Mac**.

---

## 1. What this is

**Leave Her Johnny** is a top-down 2D HTML5 canvas pirate game (cozy, "Bluey"-flavored art direction — chunky, rounded, flat-shaded, warm palette). You sail a ship, fight navy ships / monsters / bosses, raid cargo, explore islands on foot ("shore parties"), board crippled ships, buy ship upgrades, and progress through 8 themed seas plus an endless "Open Seas" mode.

It ships **two ways from the same code**:
- **Web:** getarsenal.app (GitHub Pages). The root `index.html` is the web build.
- **iOS:** a Capacitor app → Codemagic CI → TestFlight. The iOS bundle is `www/index.html` (a **copy** of root `index.html`).

The whole game is **one file**: `index.html`, containing one `<style>` block and one big `<script>` block. No build step, no framework, no modules. Pure DOM + Canvas2D + WebAudio.

---

## 2. Repository layout

```
Leave-Her-Johnny/
├── index.html              ← THE GAME (web build, single file: HTML+CSS+one <script>)
├── www/
│   ├── index.html          ← iOS bundle copy — MUST be kept identical to root index.html
│   └── assets/             ← mirror of /assets for the iOS bundle
├── assets/                 ← cannon.mp3, explosion.mp3, storm.mp3, musket.mp3 (SFX),
│                             mountain0-3.png, volcano0-3.png (island rocks),
│                             hero.png, icon.png
├── leave-her-johnny-icon-1024.png   ← splash/app icon (also copied into www/)
├── capacitor.config.ts     ← Capacitor config (appId, webDir:'www')
├── codemagic.yaml          ← iOS CI/CD pipeline (build + sign + TestFlight)
├── package.json            ← Capacitor deps (must match the native ios/ project)
├── ios/                    ← Capacitor iOS native project (SPM flavor, no CocoaPods)
└── README.md
```

> **Critical rule:** there is NO bundler. After editing `index.html`, you must copy it to `www/index.html` (`cp index.html www/index.html`) or the iOS build ships stale code. Keep `assets/` mirrored into `www/assets/` too.

**Audio note:** All SFX are mp3 files in `assets/`. **All music is procedural** (synthesized live via WebAudio oscillators — there are no music files). See §6 Music.

---

## 3. Architecture & runtime

- **One script.** Everything lives between `<script>` … `</script>` in `index.html` (≈ line 524 to the end). To syntax-check just the JS, extract that block and run `node --check` (see §9).
- **Global state object `G`** (declared `const G = { … }`, ≈line 725) holds *everything*: `state`, `t`/`dt`, `cam`, `world`, `mapIndex`, `gold`/`wood`, `ship`, `entities[]`, `cannonballs[]`, `particles[]`, `upgrades{}`, `stats{}`, run/combo counters, `shore` (also reused for boarding), `endless`, etc. Read the `G = {…}` block first — it's the data dictionary for the whole game.
- **State machine** — `G.state` is one of:
  - `'menu'` — title/menu, idle ship circling.
  - `'play'` — normal sailing/combat (the main game).
  - `'shop'` — an overlay is open (upgrade shop, songbook, records). World update pauses.
  - `'shore'` — a shore party **OR a boarding party** is active (both use `G.shore`; boarding sets `G.shore.boarding=true`). World update pauses; a separate update/draw path runs.
  - `'end'` — game over screen.
- **The loop** — `function loop(now)` (≈line 7022) runs each frame via `requestAnimationFrame`. It branches on `G.state`: in `'play'` it updates entities/combat/weather/prompts; in `'shore'` it runs `updateShore` (which delegates to `updateBoarding` if boarding); then it always calls `updateMusic()` and `render()`. The whole body is wrapped in try/catch that surfaces the first error into an on-screen `#errbox` (so a crash shows a message instead of a silent freeze).
- **Rendering** — `function render()` (≈line 6873) draws everything in world space using `wx(x)`/`wy(y)` (world→screen, applying `G.cam.x/y/zoom`). Order: ocean → shadows → wakes → wrecks → hazards → islands → floaters → shore/boarding overlay → entities (y-sorted) → cannonballs → particles → weather → HUD. The camera follows the ship (play) or the party (shore/boarding, at a closer zoom).
- **Coordinates:** world units. `wx/wy` convert to screen. Entities have `{x,y,a (angle), hp, maxhp, r (radius), type, …}`.

---

## 4. Code section map

The file is divided by `/* ===== … ===== */` banners. Major regions (search the title text to jump there):

| Region | What's there | Key identifiers |
|---|---|---|
| GAME STATE | the `G` object | `const G =` |
| SAVE / LOAD | localStorage persistence | `saveGame`, `loadGame`, `clearSave`, `saveLifetime` |
| ACHIEVEMENTS | lifetime-stat goals | `checkAchievements`, `G.stats` |
| INPUT | touch/drag + keyboard | `input` (`.active/.ang/.mag`), `KEYS`, `applyKeyboardInput` |
| WORLD GENERATION | seas, islands, forts, caches | `genWorld`, `startMap`, `MAPS`, `islandEdge`, `shoreAt` |
| OCEAN / ISLAND / CLOUD RENDER | environment drawing | `drawOcean`, `drawIsland`, `drawClouds` |
| SHIP FACTORY + STATS | ship/entity creation, the fort | `makeShip`, ship-stat scaling from `G.upgrades` |
| CARGO SHIP VARIETY | merchant types | `CARGO_TYPES`, `makeCargo`, `pickCargoType` |
| (navy) | warship classes | `NAVY_CLASSES`, `makeNavy`, `pickNavyClass` |
| BOSS_STATS / BOSS FIGHTS | per-boss stats + AI | `BOSS_STATS`, `spawnBossOf`, `updateBoss`, `updateWraith/Narwhal/Drowned/...` |
| ENTITY UPDATE / AI | per-entity behavior dispatch | `updateEntity`/`updateEntities` (the main entity loop) |
| PLAYER UPDATE | player ship movement | `updatePlayer` |
| CANNON / COMBAT | firing, cannonballs, damage, death | `fireCannons`, `updateCannonballs`, `dealDamage`, `killEntity` |
| ISLAND TREASURE CACHES + LANDING PARTY | buried-treasure dig trips | `sendLandingParty`, `updateLandingParty` |
| INTERACTIVE SHORE PARTY | on-foot island exploration | `startShoreParty`, `updateShore`, `drawShore` |
| ⚔️ BOARDING PARTIES | storm a crippled ship's deck | `startBoarding`, `updateBoarding`, `drawBoarding`, `isBoardable` |
| FORT behavior | shore-fort cannons/mortars | fort entity update |
| PARTICLES / SPLASH / WAKES | fx | `G.particles`, `updateParticles`, `drawParticles` |
| HAZARDS | reefs / drift ice / whirlpools | `G.world.hazards`, `updateHazards`, `drawHazards` |
| WEATHER | cosmetic storms/rain | `updateWeather`, `drawWeather` |
| SHIP & MONSTER SPRITES | all entity drawing | `drawEntity`, `drawHull`, `drawMonster` |
| CAMERA / MAIN RENDER / LOOP | frame orchestration | `render`, `loop`, `wx`, `wy` |
| HUD / UI | on-screen UI, shop | `updateHUD`, `el(id)`, shop rendering, `UP` |
| FLOW | map transitions, game over | `startMap`, `gameOver`, `resetRun`, `nextMap` |
| OPEN SEAS | endless mode | `updateEndless`, `G.endless`, `ENDLESS_BOSSES` |
| LOGBOOK / records | stats/achievements panel | records tab, `runSnap` |
| DEV MENU | spawn console (triple-tap map pill) | `DEV_ACTIONS`, `buildDevSpawners`, `devPicker` |
| SOUND | procedural WebAudio SFX + ambient | `SND`, `SAMPLE_SRC`, `tone`, `reedTone`, `sfx.*`, `initAudio`, `rebuildAudio` |
| SEA SHANTIES | songbook tunes | `N` (note→freq), `SHANTIES`, `playShanty`, `stopShanty`, `shantyTone` |
| BACKGROUND MUSIC THEMES | boss/shore looping music | `THEMES`, `playTheme`, `stopTheme`, `musicTone`, `updateMusic` |

---

## 5. Core data structures

- **`UP`** (≈line 779) — upgrade definitions, keyed: `hull, sails, cannons, crew, hold, hotshot, spread, ballast, chain, plating, boarding`. Each: `{name, emoji, max, desc, tiers:[3 names], cost:l=>({g,w}), gate:l=>minMapIndex}`. `G.upgrades` mirrors these as current levels (all default 0; the maxed/dev variant sets some to 3). **When you add an upgrade, add its key to every `G.upgrades={…}` initializer** (there are ~5: the `G` def, new-game resets, the save-load `Object.assign` defaults, and a dev maxed-out variant) or loaded saves go `undefined`.
- **`MAPS`** (≈line 849) — the 8 seas, each `{name, size, islands, cargo, navy, monsters, treasures, boss, …}`. Sizes escalate 7000→31000. Bosses by sea: `shark, squid, ghost, kraken, wraith (Coral Colossus), narwhal (Hoarfrost Leviathan), drowned (Davy Jones)`. Hazards appear in later seas: reef / drift-ice / whirlpool.
- **`NAVY_CLASSES`** (≈line 2407) — `sloop, cutter, frigate, galleon, manowar` (escalating r/hp/speed/cannons/bounty). `makeNavy` scales further by `mapIndex`.
- **`CARGO_TYPES`** (≈line 2440) — merchants (`fishingsmack … treasurebarge, galleon`); some flagged `rich:true`.
- **`BOSS_STATS`** (≈line 2660) — per-boss `{hp, r, dmg, speed, name, fireRate, range, shipL, phases}`. `spawnBossOf(mk,x,y)` builds the boss entity; `updateBoss` dispatches to the per-boss AI function.
- **Audio:** `SND` (audio context bundle), `SAMPLE_SRC` (SFX file paths), `N` (note name→Hz dict, e.g. `G2:98 … B5:988`; only naturals + `Fs`/`Cs` + `As4`), `SHANTIES` (songbook tunes), `THEMES` (boss/shore background music).

---

## 6. Key systems (with the non-obvious bits)

### World generation & the island-silhouette gotcha (recurring bug source)
`genWorld(idx)` builds a sea from `MAPS[idx]`. Islands are drawn as a soft blob: `islandPath` traces a **quadratic Bézier** between perimeter points whose control radius bows **INWARD** (`min(neighbourOffs)*0.99`). **Any code that measures the shore must use that same quadratic blend**, or it bows outside the painted coast and things spawn in the water. Three helpers must stay in sync: `islandEdge` (draw-time/collision), `shoreAt` (per-island closure in genWorld), `edgeAt` (cove/dock loop). **Place things at `shoreAt(angle)*fraction`, never `is.r*fraction`** (nominal radius ignores per-angle shape/stretch). This bug ("party/loot/natives spawn on water") has recurred several times — treat it as the #1 thing to check when adding anything placed on or near land.

### Entities & combat
- Entities live in `G.entities` (the player ship is in there too, `type:'player'`). `updateEntities`/`updateEntity` dispatches by `type` (`player/navy/cargo/monster/boss/fort`).
- `fireCannons` spawns `G.cannonballs` (each has arc height `h`; a collision gate `h<30` means a ball only hits once it's "low" enough). Friendly vs enemy balls flagged.
- `dealDamage(e, amt, by)` applies damage, hit-stop/shake, and combat timers; calls `killEntity(e, by)` at `hp<=0`. `killEntity` handles rewards (gold/wood, combo multiplier, loot bursts, stats, treasure-map drops, boss death) and marks `e.dead` (a filter removes it after it finishes sinking).

### Bosses (seas 6–8 are bespoke)
- **Coral Colossus** (`wraith`): coral-encrusted warship; broadside → "bloom" (erupts temp reef hazards) → ram.
- **Hoarfrost Leviathan** (`narwhal`): icy creature (orca/narwhal); ice-shard volleys → dive → tusk charge leaving floe wake.
- **Davy Jones** (`drowned`): the finale — fires sharks from cannons on a cadence, summons shootable octopi that latch to your hull, conjures a whirlpool you duel inside, long multi-phase fight.
- Octopi clinging to the player live in `G.octopi` (drawn over the ship; shake them by sailing hard).

### Hazards
`G.world.hazards` = `{type:'reef'|'ice'|'floe'|'whirlpool', …}`. `updateHazards` handles drag/damage; icebergs damage on contact and **break/sink when shot**; whirlpools pull you in and kill if you linger in the eye. Some hazards are *temporary* (`tempLife` countdown) — bosses spawn them.

### Shore party (`G.shore`, state `'shore'`)
`startShoreParty(island)` rows a landing party ashore; you drag to walk (`input.ang/mag`), marines auto-fire muskets at creatures (crabs/boar/snake/natives/raiders/a Tidal Brute guardian), grab loot, free captives. `partyHp` depletes; marines visibly die off as it drops; wipe = forced retreat (loot lost), safe return = loot banked. Phases: `rowin → explore → rowout`. Crew count scales off the `crew` upgrade. Rendering in `drawShore`.

### Boarding party (reuses `G.shore` with `boarding:true`, state `'shore'`)
This is the newest, most complex feature. **It is NOT a separate `G.state`** — it runs as a `G.shore` session so it reuses HUD-hide, the zoomed camera, the `input` controls, and the music path. `updateShore`/`drawShore` each have a one-line `if(sh.boarding){ updateBoarding/drawBoarding; return; }` at the very top; everything else is self-contained.
- **Trigger:** `checkPortPrompt` (runs each frame in `'play'`) offers a "Board Her!" prompt (mode `'board'`, allowed mid-combat) when you sail alongside a boardable ship below 50% HP. `isBoardable(e)`: navy `frigate/galleon/manowar`, or cargo with `rich`. Gated to the `boarding` upgrade ≥1 and `mapIndex>=1`.
- **Deck:** a **ship-hull-shaped** arena in WORLD space. `deckToWorld`/`worldToDeck` are exact inverses; `deckHalfWidthAt(sh, along)` is the single source of truth for the hull width profile (pointed bow, broad stern) — used by BOTH the movement clamp (`clampToDeck`) and the drawn silhouette. **Keep those two in sync.** Deck size, enemy count, and wave count scale with ship class (`sizeF`); `sh.camZoom` adapts the camera.
- **Fight:** marines fire muskets (count/HP/dmg scale with Boarding+Hull+Cannons+Crew); enemy boarders pour from hatches + stern doors in waves. Big prizes spawn a **Captain** mini-boss in the final wave; felling her **routs** the survivors (`fleeing` — they bolt overboard). Extras: **powder kegs** (proximity-lit AoE), grab-able **deck loot** (banked on win), off-screen enemy arrows.
- **Outcome:** clear all waves → sink her + **DOUBLE loot** (`killEntity` base reward + a 1× bonus + deck loot). Party wiped → survivors swing back, the enemy ship survives at current HP (finish with cannons), your hull takes a 12% hit + a short re-board cooldown (`e._boardCd`).
- **Win condition:** `waveIdx >= waves && no live enemies`. When counting kills for the HUD, **skip `e.dead`/`e.fleeing`** enemies in shot/target loops or "enemies left" over-counts.
- The two real ship sprites are hidden during boarding via `e._boarding` (a guard in the render entity loop; `G.ship` is in `G.entities`).

### Music engine (fully procedural — no audio files)
- `N` = note→frequency dict (naturals + `Fs`/`Cs` + `As4`, range ~G2–B5). Notes are `[freq, beats]` where `beats` are in quarter-note units and a tune's `beat` field = seconds per quarter.
- `SHANTIES` (songbook) play via `playShanty`/`stopShanty` on `SND.shantyBus` (self-looping).
- `THEMES` (background) = `{boss, shore}`, play via `playTheme`/`stopTheme`/`musicTone` on a **separate** `SND.musicBus` so they never collide with the songbook. `updateMusic()` (each frame) picks the theme by state: boss alive → Caribbean Adventure; shore/boarding → Roguish Captain; else silent.
- **Adding tunes (the user supplies ABC notation):** convert with — duration `beats = ABC_eighths / 2`; `beat = 60/Q` (for `Q:1/4=Q`) or `40/Q` (for 6/8 `Q:3/8=Q`); ABC uppercase letters = octave 4, lowercase = octave 5, `,` down / `'` up an octave; apply the key signature to every letter (e.g. K:Dm → `B`=B♭=`As`). If a needed pitch isn't in `N`, add it or transpose to a key that fits (the dict has no general flats). Verify each measure sums to the meter and every `N.x` resolves (a typo'd note is silent at runtime, not a syntax error).

### Save / load, endless, dev menu
- `saveGame`/`loadGame` use `localStorage` (works on Pages and in Capacitor). `saveLifetime` persists `G.stats`. Adding a new persisted field? Add it to the save object AND the load defaults.
- **Open Seas (endless):** `updateEndless` ramps a `threat` from time survived + notoriety; spawns escalating fleets/storms and cycles `ENDLESS_BOSSES`.
- **Dev menu:** open by **triple-tapping the map-name pill** (top of screen). Spawns enemies/cargo/bosses/forts/hazards/seas, grants upgrades, toggles god-mode (`G.dev.invincible/oneShot`). Great for testing a feature without grinding to it.

---

## 7. Conventions & gotchas (read before editing)

1. **Sync the iOS copy.** After any `index.html` change: `cp index.html www/index.html`. Mirror new files in `assets/` → `www/assets/`. The splash/app icon `leave-her-johnny-icon-1024.png` must also exist inside `www/` (it's referenced root-relative; the iOS bundle root is `www/`).
2. **`node --check` catches SYNTAX only, never runtime.** A real bug (e.g. a too-broad `replace_all` once rewrote a function's body into infinite recursion) passes the syntax check and crashes at runtime. For logic changes, also *reason about / simulate* runtime: stub the function in `node -e` and call it, or open in a browser. Watch for self-reference, recursion, undefined globals.
3. **Never `replace_all` a code pattern that also appears inside the function it routes to.** Scope replacements or edit call sites individually.
4. **Island silhouette consistency** — see §6 World gen. Anything placed on/near land must use `shoreAt(angle)*fraction` and the quadratic blend, or it lands in the water.
5. **iOS AudioContext** backgrounds to state `'interrupted'` (not `'suspended'`). Resume on *any* non-`'running'` state; on return, **rebuild** the context (`rebuildAudio` — close + new context + ambient, reuse decoded buffers; it also drops `SND.musicBus`/`_themeLoopId` so the theme restarts). A resumed zombie context is silent.
6. **No `backdrop-filter`** over the animating canvas — it broke tap hit-testing on iOS. Overlays use solid/translucent fills.
7. **`git push` under the sandbox fails** with "Could not resolve host: github.com" (the Bash sandbox blocks git's network). Push with the sandbox disabled, or from a normal shell / PowerShell. (PowerShell wraps git's stderr as a red "error" even on success — check for the `old..new  main -> main` line to confirm.)
8. **Performance:** it's a single canvas redrawn every frame. Keep per-frame allocations modest; reuse particle/array patterns already in the code.

---

## 8. Build & ship pipeline (iOS → TestFlight, from Windows, no Mac)

Apple team: **Scheidel Holdings LLC**. Bundle id: **`com.scheidelholdings.leaveherjohnny`**. Repos under `github.com/getarsenal`.

- The iOS native project in `ios/` was cloned from a proven template ("Dont-Touch-My-Boats"), **SPM flavor, no CocoaPods**. `capacitor.config.ts` has `webDir: 'www'`.
- `package.json` MUST match the native project: **Capacitor `^8.3.4`**, **`@capacitor/haptics`**, and a **`typescript`** devDep (needed to read `capacitor.config.ts`). `cap sync ios` regenerates `Package.swift` from npm, so npm and native must agree.
- **Code signing (this is the answer — don't repeat the multi-build saga):** this account signs from **provisioning profiles UPLOADED into Codemagic's code-signing UI**, NOT via App Store Connect API fetch. `codemagic.yaml`'s `ios_signing:` block (`distribution_type: app_store` + the bundle id) signs from those uploaded profiles.
  - For a NEW app/bundle id you must: register the App ID + app record; create an **App Store distribution provisioning profile** in the Apple Developer portal using the **shared distribution certificate** the other apps use (already in Codemagic — don't re-upload it); **download that `.mobileprovision` and upload it into Codemagic** with a reference name. LHJ used reference name "Leave her Johnny Reference" (cert "Leave Her Johnny App Store"). Green check = good.
  - Dead ends that wasted builds, do NOT repeat: the explicit `app-store-connect fetch-signing-files --create` CLI (no `CERTIFICATE_PRIVATE_KEY` on this account → 0 certs); and assuming `ios_signing:` fetches via API (it doesn't — it uses the uploaded profile).
- **Deploy = push to `main`.** `codemagic.yaml` triggers on push to `main`, runs `npm install` → `cap sync ios` → `xcode-project use-profiles` → set build number from timestamp → `xcode-project build-ipa` → publish to TestFlight. So the normal release flow is just: edit → sync www → commit → push `main` → Codemagic builds → TestFlight.

---

## 9. The safe edit workflow (do this every change)

```bash
# 1. Edit index.html

# 2. Syntax-check just the <script> block
START=$(grep -n '^<script>' index.html | head -1 | cut -d: -f1)
END=$(grep -n '^</script>' index.html | head -1 | cut -d: -f1)
sed -n "$((START+1)),$((END-1))p" index.html > _check.js
node --check _check.js && echo "SYNTAX OK"
rm -f _check.js

# 3. For logic changes: runtime-simulate. Extract the relevant functions,
#    stub the globals they use (G, input, sfx, dist, clamp, lerp, TAU, …),
#    and call them in `node -e` to verify behavior (this caught real bugs
#    in the boarding feature). Or just open index.html in a browser.

# 4. Sync the iOS bundle
cp index.html www/index.html

# 5. Commit + push (push needs the sandbox disabled — see §7)
git add -A && git commit -m "…"
git push origin main      # → Codemagic builds → TestFlight
```

---

## 10. Quick identifier glossary (grep these)

`G` (global state) · `loop` (rAF frame) · `render` · `wx`/`wy` (world→screen) · `genWorld`/`startMap`/`MAPS` · `islandEdge`/`shoreAt` (silhouette — keep in sync) · `makeShip`/`makeNavy`/`makeCargo`/`NAVY_CLASSES`/`CARGO_TYPES` · `fireCannons`/`updateCannonballs`/`dealDamage`/`killEntity` · `BOSS_STATS`/`spawnBossOf`/`updateBoss` · `G.world.hazards`/`updateHazards` · `startShoreParty`/`updateShore`/`drawShore` · `startBoarding`/`updateBoarding`/`drawBoarding`/`isBoardable`/`deckHalfWidthAt`/`clampToDeck` · `UP`/`G.upgrades` (add `boarding` key everywhere) · `SND`/`tone`/`sfx`/`initAudio`/`rebuildAudio` · `N`/`SHANTIES`/`playShanty` · `THEMES`/`playTheme`/`updateMusic` · `checkPortPrompt` (proximity prompts) · `updateEndless` · `saveGame`/`loadGame` · dev menu = triple-tap the map-name pill.

---

*Generated as a project handoff. The source of truth is always `index.html` — when this doc and the code disagree, the code wins; please update this doc.*
