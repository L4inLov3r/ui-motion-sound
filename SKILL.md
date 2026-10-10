---
name: ui-motion-sound
description: Build a looping SaaS UI motion video from real product screens (renderAt(t), springs, cursor-driven, 120 BPM) and add a real-recording sound layer with measured QA via opus-sound-layer.
---

# UI motion + sound layer

One pipeline, two halves: a deterministic UI motion loop built frame by frame from `t`, then a sound layer of real recordings placed from the animation's own timing and checked by a QA script.

You can't watch your frames move and you can't hear your audio. So: build every frame from `t`, check stills, place sounds from the timeline, look at waveforms, and let the QA decide. A human does the final eye and ear check. Never call the motion or the sound "good"; report what the checks show.

## Requirements

Check these before starting and say what is missing:

- `ffmpeg` on PATH
- `uv` (runs the sound scripts and installs their Python packages)
- Playwright with Chromium (frame capture)
- Python with `numpy` (beat grid)

## Step 0: Inputs and gate

Ask the user first, in one message:

- 8 to 12 UI states of the product (screens, components or Figma frames)
- black and white plus one accent colour
- a 120 BPM track, or a click track (`track.wav`)
- fonts, logo, and the last line of copy
- format: 1440x1440 by default, 1920x1080 if they ask

Then show the beat map and 4 stills: the opening state, two mid states and the loop seam. **Build only after the user says go.**

## Step 1: Direction

- One continuous morph. No cuts, no fades between states.
- A real cursor drives every change (hover, click, drag, type) and never leaves the frame.
- The camera zooms so each state fills the frame, then pulls back.
- Springs with mass and a tiny overshoot.
- Banned: bouncy easing, particles, glows, gradients on UI, dead time, a template look.

## Step 2: Structure

- 120 BPM, one action per beat (0.5 s).
- Button > loader > check > panel > list > chart > ... > back to the opening frame.
- Hold the hero state 2 beats. Never 4 beats without motion.

## Step 3: Build the picture

- One HTML file exposing `renderAt(t)` and `seek(t)`: every pixel comes from `t`.
- No CSS transitions, no timers, no `will-change` on scaled elements (it blurs text).
- Closed-form springs, summed when a target changes mid-flight. UI damping 0.7 to 0.8, settle in 0.3 to 0.45 s. Camera damping about 1.0.
- Snap every action to a beat grid computed from the audio with numpy.
- Keep one timeline data structure (list of actions with start time, spring parameters, target element). Both the renderer and the cue export in Step 6 read from it.
- Render with Playwright, 4 subframes per frame for motion blur, 60 fps, into `frames/%05d.png`.

Checks before encoding:

- Last frame equals first frame, cursor included.
- Text is sharp at 1x after every zoom.
- `seek(7.25)` called twice gives identical pixels.

## Step 4: Encode the picture MUTED

Do not mux the track here. The sound layer does the final mux.

```
ffmpeg -framerate 60 -i frames/%05d.png -c:v libx264 -crf 14 -pix_fmt yuv420p out/video.mp4
```

## Step 5: Sound layer setup

Clone the audited version, pinned:

```
git clone https://github.com/Bodila51/opus-sound-layer sound-layer
git -C sound-layer checkout 52ac266a24b7dfd3cd06a28c34d5af360a8d7586
```

Use its scripts as they are. Never rewrite them, never loosen a threshold to make a check pass, and do not update to a newer commit without telling the user (only this commit was reviewed).

Build the library (first run downloads about 1,100 CC0 sounds; give it a 10-minute timeout or run it in the background):

```
uv run sound-layer/scripts/sfx.py kit
```

Keep the default output folder `audio/kit`. The script deletes and rebuilds files inside it, so never point `--out` at a folder holding the user's own files.

## Step 6: Cues

Print the format with `uv run sound-layer/scripts/mix.py --schema`, then export `audio/cues.json` from the Step 3 timeline. Never type frame numbers by hand.

Top-level fields for this pipeline:

- `"fps": 60`, `"duration_s"`: the loop length
- `"energy"`: `punchy` or `calm`
- `"music": {"file": "track.wav", "start_s": 0, "loop": true, "fade_out_s": 0}` so the loop seam has no fade. Use a short fade only if the user wants a one-shot video instead of a loop.

Per cue:

- Sound only on real actions. Not every beat needs a sound; a cursor hover usually does not.
- Tag `visual` from the spring math, since there are no cuts in this film:
  - `move`: frame of peak speed
  - `land`: frame where motion has effectively stopped (the settle, under about 5% of its peak speed), not the first time the spring crosses its target
  - `appear`: frame an element pops in
- `weight`: `support` by default. One `hero` per about 4 s (logo, main reveal, the check that confirms). Hero cues use `"hit+boom"` (or `"thump+tap"` for a soft film) with `stop_before` and `build`, and a `riser` or `reverse` cue peaking on the same frame.
- `pan` follows the element's horizontal position, within -0.6 to 0.6.

## Step 7: Choose sounds

Pick each sound yourself:

```
uv run sound-layer/scripts/sfx.py browse --type <type> --like "<material, weight, mood>"
```

Look at `audio/browse.png`, then put the chosen id in the cue (`"sound": "click/<id>"`). Types: hit boom riser reverse transition whoosh swish glitch ui click key tick tap pop thump paper chime shimmer zap.

- Keep one sonic family for the whole film (for a clean SaaS UI: soft clicks, taps, light swishes, one chime). Vary repeats.
- Never synthesize hits, clicks or whooshes. No meme sounds.
- Nothing fits: `sfx.py search "<material action>" --type <type>`, check with `sfx.py sheet`, then `sfx.py add FILE --type <type> --credit "<source, licence>"`. Only CC0 or a licence that clearly allows the use; tell the user about any sound taken from outside the library.

## Step 8: Mix, mux, QA

```
uv run sound-layer/scripts/mix.py audio/cues.json
ffmpeg -i out/video.mp4 -i audio/mix.wav -map 0:v -map 1:a -c:v copy -c:a aac -b:a 256k -shortest out/final.mp4
uv run sound-layer/scripts/qa.py out/final.mp4
```

- Fix every FAIL by changing cues or sounds, at most 2 rounds. If a FAIL remains, report it as it is.
- Warnings on support cues are common here because the cursor keeps the picture moving; list them, do not hide them.
- Open `audio/qa.png` and describe what you see at each hero cue.

## Step 9: Deliver

- `out/final.mp4`
- `audio/kit/CREDITS.md`
- `LISTEN.md`: 5 timecodes for the user to check by ear, plus the loop seam
- The QA result as printed: passes, fails, warnings. No verdict on how it sounds or looks.
