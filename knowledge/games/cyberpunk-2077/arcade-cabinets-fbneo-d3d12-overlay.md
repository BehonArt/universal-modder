---
kind: game
title: 'Playable arcade cabinets in Cyberpunk 2077: a RED4ext plugin running FinalBurn Neo, drawn through a D3D12 overlay'
game: Cyberpunk 2077
games_also: []
game_version: 'Steam PC 2.31 (WolvenKit GameVersion 2310), Codeware 1.20.5, redscript 0.5.31'
platform: windows
engine: redengine
route: native-hook
tools:
- RED4ext
- RedLib
- Codeware
- Audioware
- redscript 0.5.31
- redscript-cli 1.0-preview.22 (decompiler)
- WolvenKit.CLI 9.0.1
- FinalBurn Neo
- MSVC 14.51
anti_cheat: none (single player; no online features touched)
status: in-progress
agents:
- Claude Code (Sonnet 5.5)
humans:
- BehonArt
date: '2026-10-16'
links: []
tags:
- cyberpunk2077
- red4ext
- redscript
- codeware
- arcade
- emulator
- d3d12-hook
- agent-testing
---
# Playable arcade cabinets in Cyberpunk 2077: a RED4ext plugin running FinalBurn Neo, drawn through a D3D12 overlay

> Four classic arcade games (Pac-Man, Donkey Kong, Galaga, Dig Dug) run inside Cyberpunk 2077's arcade cabinets.
> A RED4ext plugin hosts FinalBurn Neo, the picture is drawn by hooking the game's D3D12 swapchain, and
> redscript wires cabinets, input, saved high scores and Eddie rewards. The emulator, overlay, input bridge and
> scores were verified in the real game; the per-game cabinet reskins were built but not yet seen in game.

## Setup
- Cyberpunk 2077 Steam PC, game 2.31. Mods: RED4ext, RedLib (native classes), Codeware, redscript 0.5.31,
  ArchiveXL, TweakXL, Audioware, entSpawner (to place a test cabinet).
- Toolchain: Visual Studio build tools (MSVC 14.51) for the plugin; WolvenKit.CLI 9.0.1 for archives;
  redscript-cli 1.0-preview.22 to decompile the game's scripts, and 0.5.31 (the version the game ships) to
  type-check your own.
- Users supply their own ROM zips. Nothing ROM-derived or game-derived is part of the mod.

## Route and why
- **No released mod adds a new game to the city's cabinets.** The proven pattern is a CET-Lua mod that overrides
  `ArcadeMachine.SetupMinigame`, hides the cabinet's attract video, and draws into the cabinet's ink widget
  tree. This project follows the same hook points, but runs the game natively.
- **Emulator in the plugin:** FinalBurn Neo compiled into a RED4ext/RedLib plugin. A hand-written Z80 emulator
  came first (it passed ZEXDOC/ZEXALL and was pixel-exact against a tile renderer) and is kept as a fallback
  that draws the screen as ink tiles. FBNeo replaced it so four games share one core and ROM versions can be
  recognised by CRC.
- **Picture:** uploading a texture every frame through ink was the fallback; the main path hooks the
  `IDXGISwapChain` and `ID3D12CommandQueue` vtables with RED4ext's hooking API and draws a point-sampled quad
  right before `Present`.

## How the game works (what we had to learn)
- Since 2.0 the vanilla arcade games run on native `gameuiarcade*` controllers (Roach Race, Trauma Drama,
  Tank) set up from TweakDB `Arcade*_Record`s. The `ArcadeMinigame` enum is native, so a new game type has to
  be scripted rather than added as data.
- `ArcadeMachine.SetupMinigame` picks the game from the cabinet's attract video. Play starts through
  `BeginArcadeMinigameUI` (a `lockedCamera` workspot), then a menu event switches to a full-screen menu entry in
  `base\gameplay\gui\fullscreen\menu.inkmenu`.
- The cabinet screen is a 320x240 (4:3) root widget with a `main_display` video child. Hiding that child and
  parenting your own canvas works without touching archives.
- Every arcade cabinet in the world uses one mesh (`game_machine.mesh`). Appearances (`pacman`, `dkong`, ...)
  can be added to a patched copy of the mesh with new materials. Both outer side panels share one UV island, so
  side art shows mirrored on the second side. The patched mesh overrides the vanilla path, so another mod
  replacing it conflicts.
- A native overlay is feasible: probe a throwaway device and swapchain to read the real dxgi/d3d12 vtables
  (Present is slot 8, Present1 22, SetColorSpace1 38; the queue's ExecuteCommandLists is slot 10), hook them,
  and draw on the last DIRECT queue the presenting thread submitted to. HDR works by hooking the colour space
  call: scRGB (FP16) and HDR10 PQ were both handled, with paper white at 200 nits. A 3-slot fence ring and
  "skip the frame instead of stalling" keep it safe.
- The game's HUD virtual window (Codeware `inkHUDLayer`) is not guaranteed to be 3840x2160. Size layouts from
  `host.GetSize()`.

## Build steps
1. Build FinalBurn Neo as a static library with only the drivers you need (compile the driver files, their CPUs
   and sound cores, and zlib), generating `driverlist.h` from FBNeo's `gamelist.pl` over those driver files.
   Required defines: `/DLSB_FIRST` (see Gotchas), `/D_UNICODE /DUNICODE`. Stub the burner front end.
2. Set `nBurnSoundLen` and `pBurnSoundOut` **before** `BurnDrvInit`, because drivers size their sound buffers
   at init.
3. Identify ROM sets by file CRC and size across all zips in the folder (merged or split sets), preferring
   parent over official clone over bootleg. Save NVRAM (`.nv`) and high scores (`.hi`, via a MAME `hiscore.dat`
   trimmed to your sets) in the plugin's own folder.
4. Expose a RedLib native class (Load, Step, SetInput, GetScore) plus overlay natives (HasLiveDisplay, ShowLive).
5. In redscript, wrap `ArcadeMachine.SetupMinigame`, the PS `IsPlayable`/`GetActions`/`OnBeginArcadeMinigameUI`
   methods and `ArcadeMachineInkGameController.TurnOn`; declare the native class OUTSIDE any `module` block.
6. Install the DLL under `red4ext/plugins/<name>/`, scripts under `r6/scripts/<name>/`, any archive under
   `archive/pc/mod/`.

## Verification
- **Emulator oracle:** the tile renderer was compared against the emulator's own frames; 0 differing pixels on
  five frames. The Z80 core passed ZEXDOC and ZEXALL.
- **In game (verified):** overlay hooks installed and first draw logged, 60 fps, SDR colour space; all four
  games ran; score readback matched the on-screen scores (Pac-Man 550, Galaga 360, Dig Dug 70) using addresses
  found by matching RAM dumps against screenshots. A test bridge drove the game from outside (below).
- **Not verified:** the per-game cabinet reskins and screen stills; the Eddie payout firing; HDR on a real HDR
  display; coexistence with other D3D12 hooks (CET overlay, Steam overlay, frame generation) beyond the test PC.

## Gotchas
1. **Symptom.** RED4ext refused to load the plugin: "Missing native class 'Mod.ClassName'". **Cause:** a native
   class declared inside a redscript `module X` is looked up as `X.ClassName`. **Fix:** keep native
   declarations outside modules.
2. **Symptom.** A UI atlas never loads and the screen shows placeholder squares. **Cause:** images must be
   attached to the widget tree before `SetAtlasResource` is called. **Fix:** attach first, then set the atlas.
3. **Symptom.** A per-frame tick stops after one frame. **Cause:** `DelayCallbackNextFrame` does not repeat.
   **Fix:** reschedule with `DelayCallback(cb, 0.001, false)`.
4. **Symptom.** Transparent regions of ink images add light (pink background, halos, invisible dots).
   **Cause:** the UI blends textures as premultiplied alpha. **Fix:** write colour 0 wherever alpha is 0.
5. **Symptom.** Every FBNeo driver crashed in `ZetReadOp` on frame 0. **Cause:** `LSB_FIRST` was not defined, so
   Z80 register pairs were byte-swapped. **Fix:** compile with `/DLSB_FIRST`.
6. **Symptom.** The game crashes (access violation in the script VM) when a second large chart is selected.
   **Cause:** one function with ~6500 array-literal elements in a `switch` overflows redscript's 16-bit jump
   offsets. **Fix:** keep generated code bodies small; store data as comma-separated strings and `StrSplit`
   them.
7. **Symptom.** Hotkeys do nothing, or open other mods' menus. **Cause:** F8 and F10 belong to other popular
   mods and `]` opens the CET overlay. **Fix:** pick keys such as Page Down, and check against installed mods.
8. **Symptom.** Codeware `Input/Key` callbacks never fire during agent-driven tests. **Cause:** they do not
   receive keys injected by `SendInput`, although the game's own actions (e.g. M for the map) do. **Fix:** give
   the mod a file bridge for tests, e.g. a text file of commands polled every 0.25 s.
9. **Symptom.** Steam launch stops at the REDlauncher waiting for a Play click. **Fix:** launch
   `bin\x64\Cyberpunk2077.exe --launcher-skip -skipStartScreen` directly (`um win launch`).
10. **Symptom.** Simulated mouse movement is ignored in menus. **Cause:** the game reads raw input. **Fix:** use
    scan-code keys (arrows + Enter) with `um win drive`.
11. **Symptom.** `Bool == Bool` and indexing a temporary (`StrSplit(x)[1]`) do not compile. **Fix:**
    `NotEquals`/a local variable. (redscript 0.5.31.)
12. **Symptom.** Persistent file storage errors on the second request. **Cause:** `RedFileSystem` storage can only
    be requested once per run. **Fix:** request it once from a `ScriptableService` `OnLoad`.
13. **Symptom.** Audio and video drift apart when the emulator is silent. **Cause:** `sound_update` pushes no
    samples while sound is disabled. **Fix:** push silence on those frames.
14. **Symptom.** Vertical arcade games look sideways. **Cause:** vertical drivers draw the raster rotated.
    **Fix:** rotate 90 degrees; boards flagged FLIPPED then need no extra half turn.
15. **Symptom.** Windows idle time reads near zero right after an automated test. **Cause:** injected input
    resets the idle timer. **Fix:** do not use idle time to decide whether the human is at the PC after you
    have driven the game.

## Assets
- Cabinet textures are baked by script into a private archive built with WolvenKit.CLI. Cabinets use generic
  colours and decals, with no publisher logos. Marquee art uses texture u 0..0.868, so true-aspect marquees
  need the 1.2x stretch compensated.
- Nothing ROM-derived is committed, shipped or described here beyond RAM addresses used for score readback.

## Cost and time
About a week of part-time sessions (Oct 9 to Oct 16, 2026).

## Open questions
- Does the overlay coexist with CET, the Steam overlay and DLSS/FSR frame generation?
- Is the Eddie payout paid once per save or once ever? (Currently per save.)
- Per-game screen stills for the attract screen; the reskinned cabinets have not been seen in game yet.
