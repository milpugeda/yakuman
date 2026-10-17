# Yakuman hand-off page

This folder is the whole of the future public repo `milpugeda/yakuman`. Its `index.html` is served at
https://milpugeda.github.io/yakuman/, the game's old address, and moves players to https://yakuman.pages.dev/ with their
progress.

## How it works

- All of a player's progress (profiles 1 to 3, saved runs, unlocks, settings) is in the browser's localStorage. That storage belongs to one origin, so the new address starts empty.
- The hand-off page reads only the game's own keys on `milpugeda.github.io`. That origin is shared by every Pages site under `milpugeda`, so other keys are never read. It changes nothing there, which keeps the old copy as a fallback.
- It packs the keys (JSON, deflated, base64url) and opens `https://yakuman.pages.dev/#import=...` with `location.replace`. The part after `#` is never sent to a server: not to GitHub, not to Cloudflare.
- The game removes `#import=...` from the address straight away (before anything else runs), then takes the progress in:
  - **This device has no progress on the new address:** it takes the progress in, reloads and shows "Progress Moved".
  - **This device already has progress there:** it asks the player, "Keep This Device's" or "Use Moved Progress". Closing the question keeps this device's progress.
  - **The same progress arrives again** (the old icon opened again): it is ignored quietly.
- Moved progress takes effect without asking only when it came from `https://milpugeda.github.io` (the page sends its origin as the referrer), or from `http://localhost` or `127.0.0.1` for local testing.
  - The check is by origin, so any of your other Pages sites on `milpugeda.github.io` would count too.
  - A `#import=` link from anywhere else always asks first.
  - The game also checks every key and value. Profiles, runs, setups and hints with any markup in them are dropped.
- **Transfer code.** The page also offers a transfer code (`YKMT:...`) that the player pastes into the game: Settings, Import, Load Save.
  - On an iPhone or iPad Home Screen icon, the page shows the code first, because those icons keep storage apart from Safari.
  - If the progress is ever too big for an address (more than 400,000 characters), the page shows only the code.
- Size: three heavy profiles, each with a saved run, come to about 27 KB in the address. Chrome takes up to 2 MB and Safari more.
- Without JavaScript the page is a plain link to the new address.
- **Two lists must match.** The keys that move are listed in `index.html` and again in `src/game.js` (MOVED PROGRESS). Change both together.

Testing: `tests/game/handoff.spec.js` serves this page from `http://127.0.0.1` and the game from `http://localhost`, which are two
origins with separate storage. A test sends the page to the local game by setting `window.__HANDOFF_TEST = { target, delay, maxFragment }`
before the page runs. Only `localhost` and `127.0.0.1` targets are accepted, so a link can never redirect anyone's progress
somewhere else.

## Go-live steps (in this order)

**0. Ship the importing game first.**
- Publish the game build that has MOVED PROGRESS to yakuman.pages.dev (`./publish.sh`).
- Check: open https://yakuman.pages.dev/#import=hello in a private window. It should show "Progress Not Moved", and the address should lose its `#import=...` at once.
- Then on your own device, try the real hand-off before anyone else does. Serve `handoff/` locally, or test it on a spare repo's Pages site.

**1. Rename the source repo.**
- On GitHub: milpugeda/yakuman, then Settings, then Repository name. Rename it to `yakuman-src`.
- From now until step 3 is live, https://milpugeda.github.io/yakuman/ shows a 404. Pages follows the new name, to `/yakuman-src/`, and GitHub does not redirect Pages addresses. Do steps 2 and 3 straight after.

**2. Point your local copy at the renamed repo, before anything else.**
- Run `git remote set-url origin https://github.com/milpugeda/yakuman-src.git`.
- Check: `git remote -v` shows yakuman-src.
- This matters. Once a new repo named `yakuman` exists, the old remote address leads to it, and the next `./publish.sh` would push the whole source into the public hand-off repo.
- Also update the comment at the top of `publish.sh` (it names milpugeda/yakuman). The source repo no longer needs `docs/` on GitHub Pages.

**3. Create the public hand-off repo.**
- Create a new public repo `milpugeda/yakuman` with this folder's contents at its root: `index.html`, `icons/`, `README.md`. For example:
  ```
  cp -R handoff /tmp/yakuman-handoff && cd /tmp/yakuman-handoff && git init -b main && git add -A && git commit -m "Hand-off page" && git remote add origin https://github.com/milpugeda/yakuman.git && git push -u origin main
  ```
- Turn on Pages: the repo's Settings, then Pages, then "Deploy from a branch", `main`, `/ (root)`.
- Check: after a minute or two, https://milpugeda.github.io/yakuman/ shows "Yakuman" with "The game has moved to yakuman.pages.dev".
  - With your own progress at the old address, it should land on yakuman.pages.dev with your runs, unlocks and saved run.
  - The address bar should show no `#import=`.
  - A second visit should not ask anything.
  - Try it on an iPhone in Safari, on an iPhone Home Screen icon (it should show the transfer code), and on Android.

**4. Make the source repo private.**
- Do this only once step 3 checks out. In yakuman-src, go to Settings, then Danger Zone, then Change visibility, then Private.
- On a free plan this also takes down its own Pages site (`/yakuman-src/`). That is fine, because the hand-off repo now serves `/yakuman/`.
- Check: https://milpugeda.github.io/yakuman/ still shows the hand-off page. `./publish.sh` still pushes to yakuman-src and deploys to Cloudflare.

## What testers do

- **In a browser:** nothing. Opening the old address brings them to the new one with their progress. Bookmarks to the old address keep working.
- **Home Screen icon on Android:** the old icon still works through the hand-off page. Open https://yakuman.pages.dev/ in Chrome, add it to the Home Screen, then remove the old icon. Chrome's storage is shared, so the progress is already there.
- **Home Screen icon on iPhone or iPad:** that icon has its own storage, apart from Safari's. Open the old icon, then:
  1. Tap Copy Transfer Code.
  2. Open yakuman.pages.dev in Safari and add it to the Home Screen.
  3. Open the new icon, go to Settings, then Import, paste the code and tap Load Save.
  4. Then remove the old icon.
  - **Verify this on a real iPhone before telling testers.** The simulator and Playwright can't show how separate the two storages really are.

## Rolling back

- **Hand-off page misbehaves:** in the `yakuman` repo, turn Pages off, or replace `index.html` with a plain link to yakuman.pages.dev. Nothing is lost: the page never deletes anything at the old address.
- **To bring the old game back at the old address:**
  1. Delete or rename the hand-off repo.
  2. Make yakuman-src public, rename it back to `yakuman`, and turn Pages back on from `docs/`.
  3. Set your remote back with `git remote set-url origin https://github.com/milpugeda/yakuman.git`.
  - The old address's storage was never touched, so testers find their progress as they left it there.
- **Progress someone played on yakuman.pages.dev after the move** stays there. To take it elsewhere, use Settings, then Export and Import, one profile at a time.
