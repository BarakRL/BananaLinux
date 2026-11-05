# Game & Watch on both panels (Yokoi): plan

Status: idea, not started. Written 2026-10-31.

## Why

To play **Donkey Kong II** (a two-screen "Multi Screen" Game & Watch, JR-55) on the RG DS Plus, one LCD screen per
panel, as the original.

- No official re-release keeps the two screens. Game & Watch Gallery 3 (GBC) has it as an unlockable but on one
  screen; the DS "Game & Watch Collection" releases are two-screen but have other games (Donkey Kong, Oil Panic,
  Green House).
- BananaOS ships no Game & Watch emulator: no MAME, no libretro `gw` core. Its RetroArch cores are NES, SNES,
  GB/GBC, GBA, Mega Drive, N64, PS1 and PICO-8, and RetroArch games use one panel.

## The emulator: Yokoi

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
- SDL apps across both panels already work: the shelf apps (eReaderDS etc.), through `banana-leaseshim`
  (`ui/src/leaseshim.c`). Check how those apps are launched and given both panels.
- A Buildroot sysroot with SDL2 in `output/rgds-plus/` to build against.

## Games

The user supplies them. MAME's Game & Watch ROMs and artwork are converted with Yokoi's `CONVERT_ROM` scripts
(Android/RGDS target) into `yokoi_pack_rgds.ykp`. Nothing of the games goes into BananaOS or a PR.

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
4. **Adapt what doesn't work.** Likely the controller mapping, and the two panels through `banana-leaseshim` as the
   shelf apps do. Keep the changes small, and offer useful ones back to Yokoi upstream.
5. **If it all works, decide how BananaOS gets it.** Open an issue on JorreFog/BananaLinux first; it's the
   maintainer's call:
   - a shelf/Store **app**, like eReaderDS; or
   - a Buildroot package plus a Game & Watch system in the library (`systems.conf`, a `roms/gw` folder).

## Open questions

- Does Yokoi's Linux frontend open one window per display, or one window over both? The answer decides how much of
  step 4 is needed.
- How does it find its ROM pack and keep saves? It should fit the card's layout (`/data`).
- Does its audio work with BananaOS's ALSA setup and volume keys?
