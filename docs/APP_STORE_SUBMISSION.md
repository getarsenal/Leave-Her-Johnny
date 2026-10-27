# Leave Her Johnny — App Store submission (version 2.0)

Last published: 1.2. This release is **2.2** (2.0 was approved; its train is closed) (`MARKETING_VERSION` in
`ios/App/App.xcodeproj/project.pbxproj`, `GAME_BUILD` in `index.html`). The
build number is set by Codemagic from the timestamp, so every push to `main`
is a new, higher build. Team: Scheidel Holdings LLC. Bundle id:
`com.scheidelholdings.leaveherjohnny`.

## What the repository now does for you (done in this pass)

| Item | State |
|---|---|
| `www/index.html` identical to `index.html`, `www/assets` mirrors `assets` | checked, in sync |
| Every asset the game references exists in the bundle with the exact case (iOS is case-sensitive) | checked, all 20 present |
| App icon 1024×1024, RGB, **no alpha** (alpha is rejected at upload) | checked |
| Fonts | bundled (`assets/fonts`, SIL OFL); the Google Fonts `@import` is gone, so the app looks right offline and under review |
| Privacy manifest `ios/App/App/PrivacyInfo.xcprivacy` | added and wired into the App target: no tracking, no collected data, UserDefaults reason CA92.1 (Capacitor's own storage) |
| Export compliance | `ITSAppUsesNonExemptEncryption = NO` in Info.plist: no encryption questionnaire per build |
| `UIRequiredDeviceCapabilities` | `arm64` (was the retired `armv7`) |
| Orientations | iPhone: portrait + both landscapes. iPad: all four, as iPad multitasking requires |
| Deployment target | iOS 15.0 |
| Script syntax | `node --check` clean; headless boot on an iPhone viewport with the network blocked: no page errors |
| Package.swift | path to the haptics plugin made portable (`cap sync` regenerates it anyway) |

## What only you can do, in App Store Connect

1. **Create version 2.0** under the app → "+ Version". Builds arrive from
   Codemagic/TestFlight after the push to `main`; pick the newest.
2. **What's New** — paste from "Release notes" below.
3. **Screenshots.** Required sets: **iPhone 6.9"** (1320×2868, the
   `store-card-6.9in-1320x2868.png` in the repo is one; add real gameplay
   shots) and, because the app supports iPad, **iPad 13"** (2064×2752 or
   2048×2732). The 6.7" set is optional now. The first screenshot is the
   image iMessage shows for an App Store link.
4. **App Privacy** → "Data Not Collected". That is true (no account, no
   analytics, no ads, no network calls) and matches the privacy policy.
5. **Age rating** questionnaire: Cartoon or Fantasy Violence = Infrequent/Mild
   (cannon and musket combat against ships, monsters and crews, flat-shaded,
   no blood). Everything else None. Expect 9+.
6. **URLs.** Support: `https://getarsenal.github.io/leaveherjohnny-support.html/`.
   Privacy policy: `https://getarsenal.github.io/leaveherjohnny-privacy.html/`.
   Both repositories exist and read correctly (contact
   getarsenal.app@gmail.com, no data collected). **Confirm both are live with
   GitHub Pages switched on** — I could not reach them from this sandbox.
   **Marketing URL:** the page's own `og:url` is `https://getarsenal.app/`,
   which is now Excessive Force's domain. Either point the marketing URL at
   the Pages URL of this repository or leave it blank.
7. **Review notes.** "No account or login. All progress is on-device. The
   game works offline. Tap NEW VOYAGE on the title screen to play; the
   coached first voyage explains the controls."
8. **Export compliance** — already answered by the plist key; if ASC still
   asks, the answer is "No" (no encryption beyond HTTPS/OS).
9. Submit for review. Pick "Manually release this version" if you want to
   time the launch.

## Release notes (What's New, 2.0)

Leave Her Johnny 2.0 is the biggest voyage yet.

- The Sea Cavern: a dark, lantern-lit naval dungeon.
- Fleet & allies: capture warships and sail with an escort.
- Ship companions: animal friends with passive buffs.
- Sea encounters and event cards: risk-and-reward choices at sea.
- Directional wind with a HUD weathervane.
- A real fog-of-war chart you fill in by sailing, with a pop-out minimap.
- Whaling, catchable seagulls and ship cosmetics in the Trim menu.
- A narrative spine: chapter cards, prologue and a richer victory epilogue.
- Ship dossier, settings and accessibility panel, a pause screen.
- A coached first voyage for new captains.
- Hand-drawn icons across the whole interface, better readability, and
  dozens of balance and bug fixes.

## Listing copy (if you want to refresh it)

**Subtitle (30):** Cozy pirate sailing adventure

**Promotional text (170):** Sail eight seas, raid cargo, storm enemy decks,
explore islands on foot and face sea monsters, in a warm, chunky,
hand-drawn world. No account, no ads, works offline.

**Description:**
Leave Her Johnny is a cozy top-down pirate adventure. Take the helm of your
own ship and sail eight themed seas, each with its own islands, forts,
merchants and a monster waiting at the end.

Raid cargo ships and outgun the navy. Board a crippled prize and fight for
her deck. Row ashore with a landing party to dig for treasure, free captives
and face what guards the island. Capture warships into a fleet, pick up
animal companions, buy upgrades at port, learn sea shanties, and sail on
into the endless Open Seas when the story is done.

Everything is on your device: no account, no ads, no tracking, and the game
plays offline.

**Keywords (100):** pirate,ship,sailing,naval,sea,adventure,cozy,treasure,
monster,kraken,cannon,arcade

**Category:** Games → Adventure (secondary: Action).

## How a release flows

1. Edit `index.html`, then `cp index.html www/index.html`.
2. Syntax check (see HANDOFF.md §9); open it in a browser.
3. Commit and push `main`. Codemagic builds, signs from the uploaded profile
   and uploads to TestFlight (10–20 minutes). Watch the build in Codemagic.
4. In App Store Connect the build appears under TestFlight, then can be
   attached to version 2.0.

## Known limits of this pass

- No Mac here, so the Xcode project was edited by hand and validated by
  parsing, not by building. Codemagic's build is the real test; if it fails
  in "Build IPA" with a PrivacyInfo or Info.plist complaint, revert the
  commit named below and push.
- The two Pages URLs and the marketing URL could not be fetched from this
  sandbox (outbound web blocked).
