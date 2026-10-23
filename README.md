# In-game TimeCycle Editor

![The editor open in-game](screenshots/editor.png)

An ImGui panel that runs inside GTA: San Andreas, edits the game's live `CTimeCycle` array while you look at the scene, and writes the result back out as a drop-in `timecyc.dat`.

**Released:** June 2018 on GTAForums · **Platform:** GTA: San Andreas v1.0 (PC) · **Runtime:** MoonLoader + MoonImGui

## What it does

San Andreas keeps all of its lighting and atmosphere tuning — ambient colour, sky gradient top and bottom, sun core and corona, sun and sprite size, fog start, far clip, water colour and alpha, two post-processing passes, cloud alpha, directional multiplier — in a flat array called `CTimeCycle`.

Normally you edit that array as `timecyc.dat`, a large whitespace-delimited text file, and restart the game after every change. This script exposes thirty of those values as colour pickers and sliders over the running game, so you tune lighting against the actual scene, then serialise the array back into a correctly formatted `.dat`. Autosave every five minutes to `timecyc_bak.dat` means a bad experiment costs you at most five minutes.

## How it works

**Reaching the array.** `CTimeCycle` is not a symbol in the executable — each field is reached through its own pointer, read out of a hardcoded address and cast to a typed buffer:

```lua
m_nAmbientRed = ffi.cast("unsigned char*", memory.getuint32(0x55F4D6))
m_fFarClip    = ffi.cast("short*",         memory.getuint32(0x55F61B))
```

Every field is an array indexed `23 * hourIndex + weatherId` — `23` being the number of weather IDs, `hourIndex` the time-of-day keyframe. That single stride is why changing time or weather immediately reads the right slot.

**Eight-hour and twenty-four-hour timecycles.** Vanilla stores eight keyframes per weather; the `timecycle24.asi` plugin extends that to twenty-four. The script probes for that module at startup with `getModuleHandle`, switches `NUMHOURS`, and maps wall-clock hour onto the keyframe index accordingly:

```lua
if h < 5 then id = 0 end
if h == 5 then id = 1 end
if h == 6 then id = 2 end
if 7 <= h and h < 12 then id = 3 end
-- ...
```

So one build edits either layout, and the panel title reports which one it detected.

**Freezing the clock.** Editing lighting is pointless if the sun keeps moving. "Stop Clock" NOPs out the `CClock::Update` call site (five bytes of `0x90`) and pins the time to the sliders; unchecking restores the original instructions.

**The `.dat` writer is the hard part.** `timecyc.dat` is whitespace-delimited with a strict column order, a comment header at midnight and midday, and per-weather section banners. Get one column wrong and the game either refuses the file or produces garbage. `SaveToFile` reproduces all of it — headers, `// 7PM`-style time comments, column alignment — and applies the engine's own unit conversions: sun size, sprite size, sprite brightness and light-on-ground brightness are stored as bytes representing `(value - 0.5) / 10`, and the directional multiplier is a byte printed as `/ 100.0`, giving the 0.0–2.55 range the UI exposes.

## Requirements

- GTA: San Andreas v1.0 US (tested against the 1.0 builds commonly used by the modding community — the memory addresses are build-specific)
- [MoonLoader](https://blast.hk/moonloader/)
- [MoonImGui](https://blast.hk/threads/19292/)
- *Optional:* [`timecycle24.asi`](https://gtaforums.com/topic/892517-24h-timecycle/) — detected automatically if present

## Install

1. Copy `timecyc-editor.lua` into `GTA San Andreas/moonloader/` — **not** the `lib` folder.
2. Launch the game.
3. Press **CTRL + T** to open the panel.

## Usage

- **CTRL + T** toggles the panel. While it's open, player control is disabled so the camera doesn't wander.
- Weather dropdowns force current and next weather, so you can stage a transition and tune both sides of it.
- **Save Timecyc** writes `timecyc.dat` into `moonloader/`. Autosave writes `timecyc_bak.dat` every five minutes while the panel is open.
- **Reset Timecyc** calls the game's own `CTimeCycle::Initialise` to restore defaults.
- To fly around while editing, start the [debug camera](https://gtaforums.com/topic/896074-debug-menu/) before opening the panel.

## Known issues and things I'd change

- `imgui.Combo(..., #weather)` — `weather` is a 0-indexed table, and the length operator is undefined for that. In practice it returns 22, so `EXTRACOLOURS_2` can't be selected from the dropdown. Should be an explicit constant.
- `io.open` isn't checked before writing. If the working directory isn't writable, saving crashes instead of reporting an error.
- **The default window is too narrow at 16:9.** `Window.SizeX` is scaled by `sx/648`, but the label column shrinks with it — at 1366×768 labels like "Current Weather" and "Sprite Brightness" are clipped, which you can see in the screenshot above. The window is resizable, so it's cosmetic, but a truncated UI is the first thing a new user sees.
- The panel can't move the camera. It was requested on the release thread and never implemented; you have to use the debug camera.
- Globals (`bEnable`, `NUMHOURS`, `bTimecycle24Loaded`) leak into MoonLoader's shared environment rather than being local to the script.
- **The big one:** thirty separate hand-cast pointers is the wrong data model. Declaring the struct once with `ffi.cdef` and indexing it as a real C array would remove all forty-five address constants, make the `23 * hour + weather` stride explicit, and let the same code write the file and mutate memory.

## Links

- [GTAForums release thread](https://gtaforums.com/topic/910061-in-game-24h-timecycle-editor/)
- [`timecycle24.asi`](https://gtaforums.com/topic/892517-24h-timecycle/) — the plugin that enables 24-hour timecycles in the first place
