# Mario Tennis: Ultra Smash (Wii U): native build, work in progress

An attempt to run the Wii U game as a native PC program by static recompilation: the game's PowerPC code is
translated to C++ ahead of time and runs on a reimplementation of the Wii U's system libraries. No emulator.

> **No game files are included.** You need your own dump of the game (Europe, title `00050000101A3600`, v16),
> unpacked to `code/`, `content/`, `meta/`. The translated C++ and the program built from it contain the game's
> code: do not share them. Not affiliated with or endorsed by Nintendo.

## Status (19 November 2026): menus work, a match does not start

Tested on one PC (Windows 11, GeForce RTX 3070 Ti, 1280x720), with scripted button presses only.

| Step | State |
| --- | --- |
| Translate `main.rpx` (16,912 functions, 1.41 million instructions; the file has no function names) | works |
| Build with the runtime | works on Windows (clang, MSYS2) and Linux |
| Controller notice, title screen, mode select, character select, court select, screen setup, tutorial card | shown |
| Texts | **faulty**: some letters are missing ("uic  tart" for "Quick Start"), and the text on highlighted buttons is missing |
| Starting the match | **fails**: the graphics driver reports a device fault (invalid write at address 0) and the program ends |
| Sound | present in a recording of the output (not listened to). Reverb, chorus and delay are not computed |
| Speed in the menus | the game's 60 frames/s are held; the renderer needs under 2.5 ms per frame on that card |
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
  - `patch_fetch_1010102.py`: vertex data format 10_10_10_2 with conversion in the shader translator.
- `port/os_extra.cpp`: system functions other titles need, among them the multi-core task queues (`MP*`; the game
  fills the queue itself and expects `MPRunTasksFromTaskQ` to drain it), unit heaps, the working directory.
- `port/game_main.cpp`: a game-neutral entry program.
- `port/setup.sh GAME_FOLDER NAME`: the Linux build steps in order (written down from the steps done by hand; the
  script as a whole has not been run). `port/run.sh` starts the result. `port/iter.sh` is the build-and-run loop.
- `tools/run_gpu.sh`, `tools/wincap_loop.py`: run the Windows build with scripted presses, record sound and frame
  times, photograph the window.
- `tools/rpxinfo.py`: prints facts about an RPX.

The system font is not part of a game dump either: copy your console's `CafeStd.ttf` to `build-NAME/fonts/`;
until then DejaVu Sans stands in.

## License

The files here are under the MIT license, as the projects they build on. Third-party code keeps its own license.
