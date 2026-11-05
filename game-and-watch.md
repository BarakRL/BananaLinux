# Game & Watch on both panels: plan

Status: superseded 2026-11-01 by `game-and-watch-mame.md` (MAME, for a non-technical user's setup). Kept as the
record of the alternatives. Written 2026-10-31.

## Why

To play **Donkey Kong II** (a two-screen "Multi Screen" Game & Watch, JR-55; MAME name `gnw_dkong2`) on the RG DS
Plus, one LCD screen per panel, as the original.

- No official re-release keeps the two screens. Game & Watch Gallery 3 (GBC) has it as an unlockable but on one
  screen; the DS "Game & Watch Collection" releases are two-screen but have other games (Donkey Kong, Oil Panic,
  Green House).
- BananaOS ships no Game & Watch emulator: no MAME, no libretro `gw` core. Its RetroArch cores are NES, SNES,
  GB/GBC, GBA, Mega Drive, N64, PS1 and PICO-8, and RetroArch games use one panel.

## Routes at a glance

| Route | What | Both panels? | Games | Size |
|---|---|---|---|---|
| **A. `gw` core** | The libretro "Handheld Electronic (GW)" core in BananaOS's RetroArch: a `libretro-gw` Buildroot package plus a Game & Watch system (`roms/gw`) | **No**, both screens on one panel | `.mgw` files (the user has these) | small |
| **B. Span mode** | RetroArch sees one 1024×1536 display; BananaOS shows its top half on the top panel and its bottom half on the bottom (GammaOS's approach) | Yes, if the core's frame divides cleanly | `.mgw` with route A; also helps vertical (TATE) arcade and Vectrex | large |
| **C. Yokoi** | A native Game & Watch emulator with a two-screen SDL2 frontend (below) | Yes, by design | MAME ROM, artwork and screenshot, converted to a `.ykp` | medium |

**Decision (user, 2026-10-31): route C, the standalone Yokoi app.** It's two-screen by design (no frame to split,
no per-game crops), public domain, and leaves RetroArch alone, so nothing else on the handheld can break. B is the
riskiest, since it changes how RetroArch reaches the panels for everything. **A stays the fallback** (one panel,
`.mgw` files, small) if C stalls. The cost of C: MAME files converted on a PC instead of the `.mgw` files the user
has, and Yokoi has to handle input, sound, volume keys and the return to the menu itself.

(Only for B: run the `gw` core with the DK II `.mgw` and screenshot a frame. Stacked screens alone would split
trivially; console artwork between them would need per-game crops.)

## What GammaOS does (checked 2026-10-31)

There's no Game & Watch-specific feature. GammaOS (Android) has an OS-level "span"/"dual stack" mode: an app sees
both screens as one tall screen, so RetroArch draws one tall picture over both panels. An r/ANBERNIC thread ("RG DS
Retroarch with Dual Screens") suggests it for `gw`, with "adjust the layout to see if it'll work"; it never confirms
it worked. The same thread says Vectrex and CPS1 TATE modes span both screens that way. GammaOS's image ships
RetroArch's standard `gw_libretro.info`, but nothing found shows `gw`-specific two-screen handling.

## How BananaOS shows things today (why B is work)

- RetroArch: `video_driver = "gl"` on **one** connector (`video_monitor_index`: 2 the top, 1 the bottom;
  `run-retroarch:312`, `/etc/banana/retroarch.cfg`). `banana-leaseshim` (`ui/src/leaseshim.c`) hands a game exactly
  one panel through a DRM lease, so the menu keeps the other: the opposite of spanning.
- The DS emulators drive both panels themselves: KMS, both panels in one atomic commit (`docs/EMULATORS.md:109`).
  That's the model for B's presenter, or for a minimal libretro frontend just for `gw`.

## Route C details: Yokoi

- Source: https://github.com/RetroValou/Yokoi_Game_-_Watch_emulator_3ds. An emulator of the Sharp SM5A / SM510 /
  SM511 / SM512 CPUs in the Game & Watch units. **Public domain** ("Free to use, No attribution required"), so
  BananaOS could ship it.
- Donkey Kong II is in its game list (`CONVERT_ROM/GNW_LIST.md`).
- Besides 3DS and Android it has a **Linux SDL2 frontend** (`linux/`, made for PortMaster):
  - Needs only SDL2 (≥ 2.0.10) and the C++ standard library at runtime. PNG decoding and the font are vendored.
    C++20 to build.
  - Uses the Android/RGDS ROM-pack format (`yokoi_pack_rgds.ykp`).
  - Has a multi-screen layout and caches textures per display.
  - Layout: `core/` (runtime, app, audio), `input/` (controller, pointer), `render/`.
  - Cross-build script: `linux/cross-build-aarch64.sh`.
- There's also a port for the Anbernic **stock OS**: https://github.com/PolluxPT/Yokoi-RGDSPlus. It has useful
  ideas (RG DS input, volume keys, screen swap, save slots), but **it has no license**, so its code can't go into
  BananaOS. Use it as a reference for behaviour only.

## What BananaOS has for it

- SDL2 2.32 and glibc 2.41, newer than the stock-OS port needs (2.38).
- Some apps already draw on both panels: LÖVE apps such as eReaderDS, through a "zero-copy panel patch for LÖVE
  apps on both panels" (README credits). Find that patch and how those apps are launched; it's the closest model
  for C, and maybe for B. (Not `banana-leaseshim`: that gives a program **one** panel through a DRM lease.)
- A Buildroot sysroot with SDL2 in `output/rgds-plus/` to build against.

## Games

The user supplies them. Nothing of the games goes into BananaOS or a PR. Yokoi's `CONVERT_ROM` scripts (target
`rgds`: `convert_original.py --target rgds`, `convert_3ds.py --target rgds`) turn **MAME** data, tested with MAME
0.282, into `yokoi_pack_rgds.ykp`. The inputs, all inside `CONVERT_ROM/` and named after the MAME set
(`gnw_dkong2` for Donkey Kong II):
- the MAME ROM **zip** in `rom/roms/` (`gnw_dkong2.zip`);
- the MAME artwork **zip** in `rom/artwork/` (`gnw_dkong2.zip`);
- a screenshot of the console artwork in `rom/console/` (`gnw_dkong2.png`), taken in MAME itself (F12 while the game
  shows its artwork; MAME doesn't ship these).

The converter runs on a PC (Python plus `requirements.txt`, and Inkscape for the SVG screens; tex3ds is only for
3DS). It could run in the container too, with Inkscape added to `Dockerfile.banana` on the host. Only the `.ykp`
goes on the handheld.

**`.mgw` files don't work with Yokoi.** They're the libretro `gw` core's format (MADrigal's simulators). That core
would be the other route: a RetroArch core added to BananaOS, with both screens stacked on one panel unless that's
changed too.

## Getting started (facts checked 2026-10-31)

- **Toolchain and SDL2 to build against**, all from the `rgds-plus` image build (present while `output/` is):
  - compiler: `output/rgds-plus/host/bin/aarch64-linux-g++` (Bootlin glibc, C++20 capable)
  - sysroot: `output/rgds-plus/host/aarch64-buildroot-linux-gnu/sysroot`
  - SDL2 headers: `output/rgds-plus/staging/usr/include/SDL2/`
  - `pkg-config`: `output/rgds-plus/host/bin/pkg-config`, which reports the sysroot's SDL2
- **Getting it onto the handheld:** copy the binary and the ROM pack to the main card's exFAT partition (`/data`), or
  use SSH: the user's public key goes in `banana/ssh/authorized_keys` on the main card, then Settings › Network ›
  SSH on, then `ssh root@<address>` (the address is in Settings › Wi-Fi's description; `docs/FIRST-BOOT.md:76`).
  The menu holds the panels (DRM master) while it runs, so a test from SSH has to get them first. See how
  `run-retroarch` and the shelf apps hand the panels over, in `docs/EMULATORS.md` ("closed its display").
- **Git:** start from an up-to-date `main` (`git checkout main && git pull --ff-only`) on a feature branch. Yokoi's
  own clone goes in `tmp/`, which is gitignored. This plan lives on the `plans` branch, checked out at `plans/`;
  update it there and push with `git push` from `plans/`.
- **Only the RG DS Plus matters.** Build `rgds-plus` only.

## Steps

1. **Build it for BananaOS.** Clone Yokoi into `tmp/` (gitignored, outside BananaLinux's tree) and build `linux/`
   against the `rgds-plus` toolchain and sysroot. Nothing in BananaOS changes.
2. **Make a ROM pack** with Donkey Kong II (the user's own MAME files, via `CONVERT_ROM`).
3. **Test over SSH** on the Plus. Check:
   - both panels, one screen each
   - the buttons (BananaOS's joypad)
   - touch
   - sound and the volume keys
   - leaving it cleanly back to the menu
4. **Adapt what doesn't work.** Likely the controller mapping, and the two panels the way the LÖVE apps get them
   (see "What BananaOS has for it"). Keep the changes small, and offer useful ones back to Yokoi upstream.
5. **If it all works, decide how BananaOS gets it.** Open an issue on JorreFog/BananaLinux first; it's the
   maintainer's call:
   - a shelf/Store **app**, like eReaderDS; or
   - a Buildroot package plus a Game & Watch system in the library (`systems.conf`, a `roms/gw` folder).

## Open questions

- Does Yokoi's Linux frontend open one window per display, or one window over both? The answer decides how much of
  step 4 is needed.
- How does it find its ROM pack and keep saves? It should fit the card's layout (`/data`).
- Does its audio work with BananaOS's ALSA setup and volume keys?
