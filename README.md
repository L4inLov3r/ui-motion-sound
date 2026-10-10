# ui-motion-sound

A Claude Code skill that builds a looping UI motion video from real product screens, then adds a sound layer made of real recordings, placed from the animation's own timing and checked by a measured QA.

*En français : une skill Claude Code qui fabrique une vidéo d'interface en boucle à partir de vrais écrans, puis lui ajoute une bande-son faite d'enregistrements réels, placée d'après la chronologie de l'animation et contrôlée par des mesures.*

## What it does

- **Picture.** One HTML page exposing `renderAt(t)` and `seek(t)`: every pixel comes from `t`. Closed-form springs, a real cursor driving every change, 120 BPM with one action per beat, rendered with Playwright at 60 fps with motion blur.
- **Sound.** Uses [opus-sound-layer](https://github.com/Bodila51/opus-sound-layer) at a pinned commit: a library of CC0 recordings, a cue sheet exported from the same timeline as the picture, a mixer, and a QA script that measures sync, loudness and audibility.
- **Gate.** Claude asks for the inputs first, shows a beat map and four stills, and builds only after you say go.
- **No self-congratulation.** Claude cannot watch frames move or hear audio, so it reports what the checks show. The final eye and ear check is yours.

## Requirements

- [Claude Code](https://claude.com/claude-code)
- `ffmpeg` on PATH
- [`uv`](https://docs.astral.sh/uv/)
- Playwright with Chromium
- Python with `numpy`

## Install

```bash
git clone https://github.com/L4inLov3r/ui-motion-sound ~/.claude/skills/ui-motion-sound
```

For a single project, clone it into `.claude/skills/ui-motion-sound` at the project root instead.

## Use

In Claude Code:

```
/ui-motion-sound a loop for our homepage, from the product in this repo
```

Claude will ask for 8 to 12 UI states, an accent colour, a 120 BPM track (or a click track), fonts, a logo, the last line of copy, and the format.

## What you get

- `out/final.mp4`
- `audio/kit/CREDITS.md`, the source of every sound in the library
- `LISTEN.md`, five timecodes to check by ear, plus the loop seam
- The QA result as printed: passes, fails, warnings

## Notes from a first real run

A 24-second, 1920×1080 loop of eleven screens. Things worth knowing before you start:

- **Measure cursor targets, never estimate them.** Positions typed from layout arithmetic put the cursor beside its target on 5 clicks out of 14. Make the timeline point at DOM elements, measure them with `getBoundingClientRect` once fonts are loaded, and add a check that the cursor tip is inside the element at the instant of each click. Do not click before the cursor has arrived.
- **`stop_before` and `build` are audible.** On hero cues they cut the music for 0.4 s and thin it for 2 s. With four heroes in 24 seconds, the first listener heard them as sound bugs. Ask before using them, and scan the mix for dropouts before delivering.
- **Force a full repaint in `renderAt`.** Chromium keeps the sub-pixel raster offset of composited layers (anything with `backdrop-filter`, for instance) from one frame to the next. After a long run the last frame differed from the first by half a pixel of text. Toggling `display` on the stage at the end of `renderAt` made the two frames byte-identical.
- **`backdrop-filter` blur leaves a faint sharp copy of what is behind it** in headless captures. If glass panels cover other elements, clip those elements to the panel's shape.
- **Leave margin for the audibility check.** In `calm` mode most support sounds measured 1.6 to 1.9 dB of lift against a 1.5 dB threshold. `punchy`, sounds with energy above 2 kHz, and not reusing one recording in very different musical contexts gave more room.
- **Render time.** About five minutes for 1,440 frames at four subframes each, with three browsers in parallel.

## Credits

- Sound layer: [Bodila51/opus-sound-layer](https://github.com/Bodila51/opus-sound-layer) (MIT), pinned at commit `52ac266a24b7dfd3cd06a28c34d5af360a8d7586`. The skill uses its scripts unchanged.
- Sounds: CC0 packs from [Kenney](https://kenney.nl) and recordings from Freesound, downloaded by `sfx.py kit`. Every source is listed in `audio/kit/CREDITS.md` after the first run.
