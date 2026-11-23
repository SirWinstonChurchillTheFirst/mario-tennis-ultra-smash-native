# Mario Tennis: Ultra Smash (Wii U): native build, work in progress

An attempt to run the Wii U game as a native PC program by static recompilation: the game's PowerPC code is
translated to C++ ahead of time and runs on a reimplementation of the Wii U's system libraries. No emulator.

> **No game files are included.** You need your own dump of the game (Europe, title `00050000101A3600`, v16),
> unpacked to `code/`, `content/`, `meta/`. The translated C++ and the program built from it contain the game's
> code: do not share them. Not affiliated with or endorsed by Nintendo.

## Selbst bauen / Build it yourself

One command builds the native program from **your own dump** of Mario Tennis: Ultra Smash. Nothing from the game is in this repository
or downloaded by the scripts: the game's code (main.rpx) is translated to C++ and compiled on your computer.
*Ein Befehl baut das native Programm aus deinem eigenen Dump; nichts vom Spiel ist hier enthalten oder wird geladen.*

**You need / Du brauchst**
- your own dump of the game, **unpacked**: a folder with `code/`, `content/`, `meta/` (as Cemu or a Wii U dumper
  such as Dumpling gives it; `.wua`/`.wud` images must be unpacked first). Tested with Europe
  (00050000101A3600); other versions are translated from your files the same way and may work.
- Windows 10/11 64-bit or Linux x86-64; a CPU with AVX2/FMA (Intel Haswell, AMD Zen or newer); a GPU with a Vulkan driver.
- internet for the build tools and the pinned sources (Smash4PC `f145e7f`, ZArchive, addrlib, Vulkan-Headers from GitHub).
- about **1 GB** free in this folder (plus about 3 GB for MSYS2 on Windows) and 8 GB of RAM or more.
- time: about **3-8 minutes** for the first build on an 8-core PC; a second run only redoes what changed.

**Windows** (PowerShell):

```powershell
git clone https://github.com/SirWinstonChurchillTheFirst/mario-tennis-ultra-smash-native
cd mario-tennis-ultra-smash-native
powershell -ExecutionPolicy Bypass -File .\setup-windows.ps1 -GameDir "D:\Games\Mario Tennis: Ultra Smash"
.\play\Start.bat
```

It installs [MSYS2](https://www.msys2.org) with winget if it is missing and the build packages into its CLANG64
environment with pacman (clang, lld, cmake, ninja, Python, git, SDL3, Vulkan loader + headers, shaderc, zlib, zstd,
xz, DejaVu fonts). Without `-GameDir` it asks for the folder. `-Jobs N` limits the parallel compile jobs; the build
runs at low priority so the PC stays usable.

**Linux** (Debian/Ubuntu, Arch, Fedora; others: install the same packages yourself):

```bash
git clone https://github.com/SirWinstonChurchillTheFirst/mario-tennis-ultra-smash-native
cd mario-tennis-ultra-smash-native
./setup-linux.sh "/path/to/Mario Tennis: Ultra Smash"
./play/start.sh
```

Missing packages are installed with sudo (apt: clang cmake ninja-build git python3 pkg-config zlib1g-dev libzstd-dev
libssl-dev libvulkan-dev libshaderc-dev fonts-dejavu-core, and libsdl3-dev where the distribution has it; without it
SDL 3.2.26 is built from source, e.g. on Ubuntu 24.04). `--jobs N` limits the compile jobs.

**What the setup does** (`tools/native-build.sh`, the same on both systems): checks your dump; clones
[Smash4PC](https://github.com/josiahg2502-glitch/Smash4PC) at `f145e7f` and the three pinned dependencies; applies
`port/upstream-local.patch` (all runtime/recompiler changes, made by the `port/patch_*.py` scripts in order); builds the
recompiler `cafe-recomp`; translates your `code/`; compiles runtime + translated game; and puts `cafegame`, a stand-in
font and the starter into `play/`. Everything it makes stays in `build/` and `play/` (both ignored by git): they hold
code translated from the game — **keep them for yourself, do not share them.**

The Wii U system font is not part of a game dump: DejaVu Sans stands in (`play/fonts/CafeStd.ttf`); put your console's
own `CafeStd.ttf` there to replace it. Saves go to `play/save/`. The system language follows your PC
(`CAFE_LANGUAGE=de|en|fr|...` overrides it).

**Getestet / tested (23 Nov 2026):** For this game the Windows setup's translation step was run on the tested dump: it gives the same code as the build that runs on the Batocera console (only the newer recompiler's `dcbf` lines differ); the complete build of this game through the scripts was not run yet. The same scripts were run end to end from an empty folder on 23 November 2026: Lost Levels and Mario Kart 8 on Windows 11, Lost Levels on Ubuntu 26.04 and on a fresh Ubuntu 24.04 (SDL3 built from source there), Mario Party 10 on Windows 11.

## Update 22 November 2026, 18:50: the match is upright on the console (now the default there)

On the small AMD console box (Radeon Vega, Mesa RADV, 60 Hz TV) the match is upright in two runs (140 s of play), 60 frames/s. Build: the 60 Hz experiments (`patch_snapshot_constants.py`, `patch_swap_retire.py`) OFF by default (`patch_60hz_off.py`; with them on, the console got worse and one 3D World run ended in a GPU device lost), the write-back guard (`patch_writeback_guard.py`) on. Which of these differences rights the picture is not isolated yet.

## Update 22 November 2026, evening: upright on the PC at 60 frames a second

Two runtime causes, both only at 60 frames a second (a TV) and hidden at 144 Hz by frame interpolation: (1) the GPU read each draw's uniform blocks from guest memory when it executed, after the title had rewritten them for later passes; now they are copied when the draw is recorded (`port/patch_snapshot_constants.py`). (2) At a swap the title went on once the previous frame's GPU work was done but before its small render targets were written back to guest memory; now they are written back first (`port/patch_swap_retire.py`). PC with `TTT2_FPS=60`: upright, 59 frames/s. Not yet tested on the console.

## Update 22 November 2026, afternoon: right brightness; match turned again on the console

Brightness: same fix as Super Mario 3D World (`port/patch_scan_srgb.py`: the scan buffers are declared sRGB, the frame is linear; the copy now encodes as the console does).

The match picture is upside down again on the small AMD console box although it was upright on the PC; the "fix" below was timing luck. It is reproducible on the PC: written-back render targets delayed by 30 ms (`TTT2_WRITEBACK_DELAY_MS=30`, `port/patch_writeback_delay.py`) or no write-backs at all (`TTT2_NO_WRITEBACK=1`) give the upside-down match. So the title depends on render-target data arriving in guest memory in time; which data and how is still open. A guard that drops write-backs over memory the CPU changed meanwhile (`port/patch_writeback_guard.py`) never triggered, so it is not an overwrite.

## Update 22 November 2026: the match picture is upright

The 3D scene of a match was drawn upside down (two post-processing passes each turned it by 90 degrees). It is upright now, with the same runtime change that brought Super Mario 3D World to 60 frames a second on a small AMD box: the staging memory the runtime copies small render targets back through is now cached (`port/patch_writeback_cached.py`). A run of the same build with the old, uncached memory showed the match upside down again; with the change it is upright (Windows, RTX 3070 Ti, scripted presses into a match). The tone-map pass's input textures were upright in both cases, so the turn came from data the title reads back; which value exactly is not shown. Not checked: a whole match, a real controller, sound by ear.

## Status (19 November 2026, morning): menus and character select are right, the match picture is turned by 180 degrees

**Update of the morning (`port/patch_night4.py`):** the character panels of the select screen are right now. Two causes: the game clears its 16-bit float depth target through a 16-bit depth view of the same memory (the runtime cleared another image, old characters stayed), and the runtime wrote to every bound render target although the lighting shaders output to one (the G-buffer got undefined values, the characters were coloured noise). In a match the scene itself is rendered right (the G-buffer picture is upright, with the right characters); the tone-mapping pass and the anti-aliasing pass each turn the picture by 90 degrees, so the result is upside down. Those two passes draw rectangle lists with the same shaders as the (right) select screen; the cause is not found yet. What looked like "garbage characters" in the match was Mario seen from behind in the turned picture.

The older status below is otherwise unchanged.

Tested on one PC (Windows 11, GeForce RTX 3070 Ti, 1280x720), with scripted button presses only.

| Step | State |
| --- | --- |
| Translate `main.rpx` (16,912 functions, 1.41 million instructions; the file has no function names) | works |
| Build with the runtime | works on Windows (clang, MSYS2) and Linux |
| Controller notice, title screen, mode select, character select, court select, screen setup, tutorial card | shown |
| Texts | fixed: all letters and button texts are shown (`patch_dcflush.py`, `patch_rects.py`) |
| Starting the match | works since `patch_target_remake.py`; the match runs at 60 frames/s with score display |
| 3D picture in the match | **wrong**: the whole scene is shown rotated by 180 degrees, and the characters are untextured or covered in coloured noise (also on the character select screen). Some of their vertex shaders fail to translate ("ALU clause runs past the end of the program"). Cause not found |
| Sound | present in a recording of the output (not listened to). Reverb, chorus and delay are not computed |
| Speed | the game's 60 frames/s are held in menus and match; the renderer needs under 5 ms per frame on that card |
| Controller, saving, amiibo, online | **not tested** (amiibo and online answer "not available") |

## What is here

Only what was added for this game and its siblings, on top of
[Smash 4 PC](https://github.com/josiahg2502-glitch/Smash4PC) at `f145e7f` (MIT), which is built on
[Tekken Tag 2 Ultimate Edition](https://github.com/Petexy/Tekken-Tag-2-Ultimate-Edition)'s cafe-recomp and
Wii U runtime by Petexy (MIT). All credit for the recompiler and the runtime goes to them.

- `port/upstream-local.patch`: all changes to that runtime and recompiler. `port/patch_*.py` are the scripts that
  made them, each with the reason in its header. What this game needed:
  - `patch_threads.py`: default threads for cores 0 and 2 and `OSRunThread` (the game runs its jobs on them);
  - `patch_threads2.py`, `patch_threads3.py`: a thread that suspends itself waits; a thread resumed by a more
    urgent thread of its core does not run ahead of it (this is not a scheduler);
  - `patch_more_os.py`: the newer sound library names `sndcore2`/`snduser2`, more voice functions, frame heaps;
  - `patch_fetch_1010102.py`: vertex data format 10_10_10_2 with conversion in the shader translator
    (`patch_fetch_1010102_order.py` adds a run-time switch for the other bit order; which one is right is not known);
  - `patch_rects.py`, `patch_rects2.py`: the rectangle-list primitive (three vertices, the console adds the fourth).
    The runtime skipped these draws, 89 of 120 in a menu frame. The translated vertex shader now builds the fourth corner;
  - `patch_dcflush.py`: `DCFlushRange` tells the renderer that the CPU wrote memory. The game draws each letter into a
    texture on first use and only flushes the cache, so later letters never appeared;
  - `patch_target_remake.py`: a render target remade at a smaller size copied its old picture past the new image
    (the device fault at match start); format 1_5_5_5 no longer needs a Vulkan extension the runtime does not enable;
  - `patch_vte_off.py`: draws whose vertices are window pixels already (viewport transform off).
- `port/os_extra.cpp`: system functions other titles need, among them the multi-core task queues (`MP*`; the game
  fills the queue itself and expects `MPRunTasksFromTaskQ` to drain it), unit heaps, the working directory.
- `port/game_main.cpp`: a game-neutral entry program.
- `setup-windows.ps1`, `setup-linux.sh`, `tools/native-build.sh`, `game.conf`: the one-command build (above).
  `port/iter.sh` is the build-and-run loop used while porting.
- `tools/run_gpu.sh`, `tools/wincap_loop.py`: run the Windows build with scripted presses, record sound and frame
  times, photograph the window.
- `tools/rpxinfo.py`: prints facts about an RPX.

## License

The files here are under the MIT license, as the projects they build on. Third-party code keeps its own license.
