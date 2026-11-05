# Game & Watch through MAME, on both panels: plan

Status: stage 1 done, stage 3 (integration) under way on `feature/game-and-watch`. Stage 1 started 2026-11-01. MVP built the same day (MAME master `ab8a0bb6`, SOURCES=hh_sm510.cpp:
176 drivers, 41 MB stripped; local patches for no fontconfig and for quitting on SDL_QUIT, `-DUSE_OZONE` for bgfx's
EGL without X11); card bundle in `tmp/gnw-test/card/`. **Stage 1: go** (see "Stage 1 results"). Replaces the route choice in `game-and-watch.md`, which stays as
the record of the alternatives.

## Goal

Every Game & Watch game on the RG DS Plus, in a way a non-technical user can set up: drop the files they have into a
folder, pick the game in the library.

- **Scope:** the 51 Nintendo Game & Watch units, single-screen and two-screen clamshell (DK II, Oil Panic, Donkey
  Kong, Green House…). Side-by-side two-screen games (Mario Bros.…) are nice-to-have. Other makers' LCD games
  (Tronica, Elektronika, Coleco tabletops…) come later, if at all.
- **Layout:** a clamshell game shows one LCD per panel, as the original. A single-screen game shows on the top panel.
- **Nothing of the games ships.** ROMs, the LCD SVGs in MAME's ROM zips and the artwork are all third-party
  (hydef's artwork: "personal use, not sold"). BananaOS ships the emulator, the launcher and layouts we write
  ourselves (positions only, never copied from an artwork zip's `default.lay`).

## Why MAME (decided 2026-11-01)

| Route | Files the user brings | Picture on a 1024×768 panel | Verdict |
|---|---|---|---|
| libretro `gw` core | `.mgw` (one file a game) | **blocky**: DK II's LCD is 276×186 in the `.mgw`, ~3.7× upscale | **tried and rejected** (below) |
| Yokoi | `.ykp`, built on a PC from MAME files (Python, Inkscape, Windows paths) | sharp (`--hires`: 1280×960 a screen) | too hard for most users |
| **MAME** | the MAME ROM zip and artwork zip, as they are | sharp: SVG LCDs drawn at panel size, 2463 px-wide scenery | **chosen** |

- **No conversion step.** MAME reads the same zips Yokoi's converter needs, directly.
- **GPL-2.0+**, so BananaOS can ship it.
- **Degrades gracefully:** without the artwork zip, MAME's built-in layout shows the bare LCDs.
- MAME already handles multi-screen machines (one view with both screens, or a window a screen). Nothing found shows
  it on both panels of an RG DS, so that part is new.

**The `gw` core, tried on the device (2026-11-02): rejected.** Built as a side-by-side comparison against MAME
(branch `feature/gw-core` off 0.5.2.2, never committed: a `libretro-gw` package at gw-libretro `91d599b`, Zlib; a
`[gameandwatch]` system with `ext = mgw` and `run-retroarch gw`; RomM's `g-and-w` slug mapped in the sync). Played with
the user's `.mgw` set from RomM: "buggy and horrible". One panel, blocky, no covers from the device's scrapers
(libretro's "Handheld Electronic Game" has ~16 Game & Watch boxes; TwilightBoxart has no such system). MAME stays the
only route; the `gw` core is no longer a fallback. The work is kept in a stash, "gw-core experiment (rejected)"
(`3e3ea0d`), should it ever be wanted for the non-Nintendo LCD games it also plays.

What the files are (checked 2026-11-01, DK II):
- ROM zip `gnw_dkong2.zip`: `jr-55_560` (4 KB, the Sharp CPU's program) plus `gnw_dkong2_top.svg` and
  `gnw_dkong2_bottom.svg` (the LCD segments).
- Artwork zip `gnw_dkong2.zip` (hydef, scans by algestam): `Screen-Top.png`/`Screen-Bottom.png` at 2463×1579 (plus
  no-shadow variants), `JR-55.png` (1767×2199, the unit with empty screens), `Backdrop.png`, `Gradient.png`,
  `default.lay` with views "Unit and Backdrop", "Unit Only", "Backgrounds Only" (+ No Shadow, Gap Fix).

## How it reaches both panels

BananaOS's SDL2 already has it: `buildroot/patches/sdl2/0001-banana-panels.patch` and `0002-banana-zero-copy-panels.patch`.
`SDL_BANANA_PANELS=stack` with `SDL_VIDEODRIVER=dummy` (software) or `offscreen` (GLES through an EGL pbuffer) makes
SDL's display the two panels together, the top one above: 1024×1536. Bottom-panel touches land on the window's lower
half; evdev input is found without udev. LÖVE apps such as eReaderDS run this way.

So MAME opens one 1024×1536 window, and a layout view puts screen 0 in the top 1024×768 and screen 1 in the bottom.

## Stages

### 1. Experiment (in `tmp/`, nothing in BananaOS changes)

1. Shallow-clone MAME into `tmp/mame`.
2. Cross-build a Game & Watch-only MAME (`SUBTARGET=gnw SOURCES=src/mame/handheld/hh_sm510.cpp`, adding sources
   until it links) with the `rgds-plus` toolchain and SDL2:
   - compiler `output/rgds-plus/host/bin/aarch64-linux-g++`, sysroot
     `output/rgds-plus/host/aarch64-buildroot-linux-gnu/sysroot`, `pkg-config` `output/rgds-plus/host/bin/pkg-config`
   - no X11, no desktop OpenGL, no Qt debugger.
3. Write a test layout for DK II: each screen on its own panel, the artwork's scenery behind it.
4. On the Plus over SSH, with the panels handed over the way the shelf apps get them: `SDL_VIDEODRIVER=dummy
   SDL_BANANA_PANELS=stack mame gnw_dkong2`. Check:
   - **speed**: full speed, CPU load (the deciding question; MAME redraws the SVG in software when segments change)
   - both panels, one screen each
   - buttons, sound, leaving it
5. Same for one single-screen game.

**Go/no-go:** full speed with headroom on DK II at panel size. If not: try `offscreen` (GLES), lower-res
rendering, or fall back to Yokoi with a user-supplied `.ykp`.

### Stage 1 results (2026-11-01, DK II on the Plus, the artwork view)

| Test | Video | Result |
|---|---|---|
| `soft` | `SDL_VIDEODRIVER=evdev`, `-video soft` | too slow, sound choppy: one core pinned (25% of the CPU, the main thread) |
| `gpu` | `SDL_VIDEODRIVER=offscreen`, `-video accel` (SDL's `opengles2` renderer on the Mali) | **full speed** ("Average speed: 99.98%", 37 s), smooth to the user; the main thread at ~51% of one core, the whole CPU 80% idle; clean exit (status 0) |

- Both panels one screen each with the artwork's scenery, the pad (SDL's game-controller line for GUID
  `190000004b4800000111000000010000`), sound (SDL's ALSA) all work.
- `-multithreading` no longer exists in MAME 0.289.
- MAME didn't handle `SDL_QUIT` at all, so `SIGTERM` (F + Start through quitwatch, and BananaOS's shutdown) did
  nothing: patched to `schedule_exit()`, which saves NVRAM. Worth offering upstream.
- Mesa logs "Failed to create //.cache for shader cache": `HOME` is `/` there; set `HOME` or
  `MESA_SHADER_CACHE_DISABLE` in the launcher.
- Still to try: a single-screen game; `gpu-bare` (no artwork) for comparison.
- **Generic results (2026-11-01, `TEST=pack-small`):** Ball 99.91%, DK Jr. 100%, DK II 99.91% (34 s), the main
  thread ~70–75% of a core. Took three fixes, all needed:
  - `GNW_SVG_MAXW` = panel width (the SVG LCD rasterised at ~1650 px wide was the cost; Ball 48% → 100%);
  - one-screen games keep the window spanning both panels and MAME draws into the top rows only
    (`BANANA_TARGET_HEIGHT`, `draw13.cpp`): a panel-sized window is scanned out to both panels by SDL's zero-copy
    path, and `SDL_BANANA_READBACK` is far too slow (Ball 13%);
  - `draw13.cpp`: a screen texture only cropped by the view (the packs' screen bounds reach past the view's edges)
    takes the plain copy and the GPU crops it, not the per-pixel cropping copy at half the speed.
- Pad (`ctrlr/banana.cfg`, `-ctrlr banana`): Start = Game A, Select = Game B, L = Time, R = Alarm. MAME's default
  had Select = Time and Game B on no button.
- Two-screen games, by MAME's screen tags (`-listxml`):
  - stacked (`screen_top`/`screen_bottom`, DK II): the view over both panels, one screen on each. Centring each
    screen on its panel (`BANANA_CENTER_SCREENS`, two clipped passes) was built and dropped: it cut the "Unit"
    views at the seam. DK II 99.91%, main thread ~64% of a core.
  - side by side (`screen_left`/`screen_right`, Mario Bros.): both on the top panel, as the pack draws them (the
    user's choice over splitting them across the panels), `GNW_SVG_MAXW` half a panel. 100%, main thread ~25%.
- No artwork (no zip, or one without a `.lay`): MAME's own layout, the same rules. MAME warns "requires external
  artwork" at start (every Game & Watch is flagged so): to hide, with our own note, at integration.
- Pad: Mario's right-hand up/down on X/B (MAME's default had Y/A).
- Later: art or text on the empty bottom panel of one-screen games (the scraped cover or the pack's unit photo).
- **Generic, no layouts of ours (`TEST=pack`):** the artwork zip's own "Background(s) Only" view
  (`-view Background`, a prefix match, the same in all hydef packs). Two screens: the window is both panels (DK II
  worked out: top screen y 108–753, bottom 781–1426, the seam at 768). One screen: `-window -resolution` one panel's
  size, which BananaOS's stack mode puts on the top panel (the bottom stays black). Screen count from `gnw -listxml`,
  panel size from DRM, so the original RG DS (640×480 panels) should work unchanged (untested: no device). Loses the
  console photo on the bottom panel; then no layouts, no per-game table and no `BANANA_LAYOUTPATH` patch.

### 2. Layouts

Not needed: the packs' own views and MAME's own layouts, placed by the rules above (dropped 2026-11-01).

### 3. Integration (branch `feature/game-and-watch`, from main `12474fb`)

Done 2026-11-01, not yet built as an image:
- `buildroot/package/mame-gnw`: MAME's 0.289 release (`mame0289`), `SOURCES=hh_sm510.cpp`, installed as
  `/usr/bin/mame-gnw`; at most 6 compilers (at 16 a 32 GB machine ran out of memory). Patches in
  `buildroot/patches/mame-gnw`: 0001 no fontconfig, 0002 SDL_QUIT quits and SIGUSR1 opens the menu, 0003
  `BANANA_SVG_MAXW`, 0004 draw13 (GPU cropping, `BANANA_TARGET_HEIGHT`, SIGUSR2 swaps the panels).
- `gw-hotkeys` (in the package): reads the pad; F + Start quits (SIGTERM, then SIGKILL after 5 s), F tapped on its own
  opens MAME's menu (SIGUSR1, a 100 ms Tab press: shorter, the UI missed it), L2 swaps the panels (SIGUSR2).
- `run-gw`: the rules above; `menu_pause 1` in MAME's ui.ini (the game paused in the menu). Card: ROM zips in
  `roms/gameandwatch`, artwork zips as downloaded in `roms/gameandwatch/artwork` (`flat = yes`); MAME's settings in
  `/data/mame-gnw`, high scores `saves/gameandwatch`, states `states/gameandwatch`, log `banana/logs/mame-gnw`.
- `[gameandwatch]` in systems.conf (`perf = performance`); pad mapping `/usr/share/banana/mame-gnw/banana.cfg`.
- A Game & Watch icon (Multi Screen, open, DK II's colours) in `ui/assets/overrides` (icons/ and systems.cfg).
- The menu's views: "Unit Only" fits the two panels well; "Unit and Backdrop" is a 4:3 frame (small in the tall
  window, by design of the packs). Default stays "Background".

- Names and covers (`gw-meta`, 2026-11-01): see "Menu names and art" below; at menu start (banana-ui-session) and
  after each game, in the background, waiting while a game plays; ~7 s a cover. Checked on the device: four games.
- Both cards: the artwork folder is the one beside the ROM (`<roms>/gameandwatch/artwork`, main or second card).

- MAME's menu, made for the device (patches 0005–0008, 2026-11-02): no warning screens; a short menu (Video Options,
  Machine Configuration, DIP Switches, Input Settings, Audio Mixer, Close Menu, Quit); on the top panel in
  two-screen games too; 16 lines a panel in the Pixel theme's Pixelify Sans (SDL_ttf kept without fontconfig: the
  built-in bitmap font scaled up was soft, and MAME's whole-step scaling sized it differently per game type).
- RomM: g-and-w → gameandwatch, each artwork zip with its game, checked by size (RomM's hashes of the files in a
  folder entry aren't the files'); test packages built from `tmp/all-prs-gw` (tmp/all-prs + this branch).

To do:
- The `-3` test package (MAME rebuilt with 0005–0008), then remove the card's temporary test override
  (`banana/systems.conf`, `/data/gwtest`).
- Hide MAME's "requires external artwork" warning when there's no artwork zip.
- Maybe: remember the view picked in the menu, and the panel swap, per game.
- Menu names and art (decided 2026-11-01): names, year and maker from MAME's own `-listxml` ("Game & Watch: Donkey
  Kong II", 1983, Nintendo), covers rendered by MAME from the user's pack (`-video none -seconds_to_run 3 -snapview
  "Unit Only"`: ~10 s a game on the Plus; the bare LCD without a pack), written as gamelist.xml and media/box by the
  scraper run. Other sources considered:
  - ScreenScraper: has a Game & Watch system and knows MAME short names, so it should cover all 51. BananaOS
    already stores a ScreenScraper account (`banana/accounts.conf`) but its scraper isn't written yet ("not
    available yet"); it needs API developer credentials for BananaOS (maintainer's call).
  - The unit photo in the user's own artwork zip (`JR-55.png`…), scaled into `roms/gw/media/box/<rom>.jpg`:
    all 51, offline, no download.
  - The current scraper (libretro-thumbnails "Handheld Electronic Game"): only 16 of the 51, and only through the
    name table.
- No issue first (the user's call, 2026-11-01).

## Open questions

- ~~Speed~~: answered, full speed on the GPU (stage 1 results). Software rendering is too slow.
- ~~Building without X11/GL~~: answered, it builds with `NO_X11=1 NO_OPENGL=1`, `-DUSE_OZONE` and no fontconfig.
  For a Buildroot package: add fontconfig to BananaOS instead of the patch, or keep the patch.
- **The user's artwork zips stay as downloaded**, so our layouts can't go inside them. Stock MAME looks for an
  external layout's images only in the folder or zip the layout came from (`render.cpp` `load_layout_file`, which
  passes no search path). Fix: a small patch so that layouts from BananaOS's own folder find their images along the
  whole `-artpath`, the way the built-in layouts already do (search path = art path, folder = game name). Then the
  layout ships with BananaOS and the images come from the untouched zip. Fallback with no patch: the launcher unzips
  the images into a cache on first run. Patch written 2026-11-01 (`render.cpp`, `$BANANA_LAYOUTPATH/<game>.lay`);
  the test bundle now has the artwork zips as downloaded, our layouts in `layouts/`.
- Where should MAME look for ROMs, artwork, layouts, cfg and nvram on the card (`/data`)?
- Which MAME version: the converter world uses 0.282; the user's sets must match the build's ROM hashes.
