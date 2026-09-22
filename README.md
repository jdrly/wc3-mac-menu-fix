# wc3-mac-menu-fix

Makes the Warcraft III: Reforged main menu usable on Apple Silicon Macs by letting its embedded browser render on the GPU.

## The problem

Warcraft III: Reforged ships for macOS as an x86_64-only build, so on Apple Silicon it runs under Rosetta 2. The game itself copes with that reasonably well. The main menu does not.

The menu is not drawn by the game engine. It is a web page rendered by a bundled Chromium 83 (`BlizzardBrowser.app`, a CEF build from 2020) and copied into the game every frame. Blizzard launches that browser with three switches baked into its binary:

```
--disable-gpu
--disable-gpu-compositing
--disable-gpu-shader-disk-cache
```

With those set, Chromium rasterizes and composites the entire menu on the CPU with SwiftShader. On Apple Silicon that CPU work is also being translated by Rosetta, and it happens at the full game resolution. At 3840x2160 that is 8.3 million pixels per frame, in software, under emulation. The result is a menu that runs at a handful of frames per second while the browser's GPU helper process pins a core.

Measured on an M5 Max, macOS 27, game build 3.0.0.24268, 3840x2160 fullscreen, sitting on the main menu:

| process | before | after |
|---|---|---|
| BlizzardBrowser Helper (GPU) | ~88% CPU, SwiftShader loaded | ~19% CPU, Apple Metal-backed OpenGL loaded |
| BlizzardBrowser Helper (Renderer) | ~25% CPU | ~15% CPU |

## The fix

`wc3-menu-fix` renames the three switches inside the `BlizzardBrowser` binary to same-length names that Chromium does not recognise (for example `disable-gpu` becomes `disable-gpx`), so they are ignored. Chromium then picks its normal hardware path, which on macOS goes through Apple's Metal-backed OpenGL driver. The bundle is re-signed ad hoc so macOS keeps allowing it to run.

The patch changes 3 strings in one file. It does not touch the game executable, game data, network code or anything Battle.net verifies at login. The original binary and its signature seal are backed up before patching.

The menu is still an off-screen browser, so each frame is read back from the GPU and copied into the game. That path stays on the CPU, which is why the result is "smooth" rather than "free". It is still a night-and-day difference.

## Requirements

- Apple Silicon Mac (it works on Intel Macs too, but there was no Rosetta problem to solve there)
- Warcraft III: Reforged installed through Battle.net
- Xcode Command Line Tools are **not** required. The script uses `perl` and `codesign`, which ship with macOS.

## Install

```bash
git clone https://github.com/jdrly/wc3-mac-menu-fix.git
cd wc3-mac-menu-fix
```

Optionally put the script on your PATH:

```bash
cp wc3-menu-fix ~/.local/bin/
```

## Use

Close the game, then:

```bash
./wc3-menu-fix apply
```

Launch the game from Battle.net as usual. To check what state things are in, including whether a running game's menu browser is on the GPU or on SwiftShader:

```bash
./wc3-menu-fix status
```

To put the original back:

```bash
./wc3-menu-fix revert
```

If the game is not in `/Applications/Warcraft III`, point the script at it:

```bash
WC3_DIR="/Volumes/Games/Warcraft III" ./wc3-menu-fix apply
```

## After a game update

Battle.net replaces the browser binary whenever it patches the game, and "Scan and Repair" replaces it too. When the menu suddenly feels slow again, that is what happened. Close the game and run `apply` again. The script notices when the binary is already patched and does nothing in that case.

## Known issues

- Some menu animations render imperfectly on the GPU path with this old Chromium build. Buttons, navigation and matchmaking are unaffected.
- Battle.net does not verify this file before launch, so the patch survives day to day, but there is no guarantee that stays true forever.
- This only helps the menu. Measured during play on a campaign map, the browser processes sit at 1 to 3% CPU with or without the patch, so in-game performance is the game engine and is unaffected by this script.

## How it was found

Sampling the game on the menu showed the browser's GPU helper process at close to a full core with `libswiftshader_libGLESv2.dylib` loaded and `--use-gl=swiftshader-webgl` on its command line, while the actual GPU sat mostly idle. The parent browser process was being started with `--width=3840 --height=2160`, matching the game resolution. `strings` on the `BlizzardBrowser` binary showed the `disable-gpu*` switches as plain NUL-terminated strings, which made a same-length rename the least invasive change possible.

## Disclaimer

Not affiliated with or endorsed by Blizzard Entertainment. You are modifying a file inside a game you licensed; do it at your own risk. Everything here is reversible with `revert` or with Battle.net's "Scan and Repair".

## License

MIT

## Contact

Jan Drlý, jd@jandrly.cz. Issues and pull requests are welcome, especially reports from other Macs, macOS versions or game builds.
