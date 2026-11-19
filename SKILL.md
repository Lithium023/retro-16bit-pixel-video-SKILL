---
name: retro-16bit-pixel-video
description: >-
  End-to-end production system and visual/audio constitution for creating broadcast-grade
  16-bit retro pixel-art motion-graphics explainer and documentary videos in React + HTML5
  Canvas with voiceover, SRT subtitle sync, seamless pixel transitions, 16-bit BGM + procedural
  SFX, studio NLE playback UI, and client-side 1920x1080 MP4 export. Use whenever the user asks
  to build, extend, restyle, or score a 16-bit / SNES / Sea-of-Stars-grade pixel-art video,
  narrated timeline animation, or Canvas-to-MP4 pixel film matching the reference standard at
  https://github.com/Lithium023/16bit-video-forNeon.
---

# 16-Bit Retro Pixel Video Production Skill

## Reference Blueprint

- **Canonical Reference Repository**: `https://github.com/Lithium023/16bit-video-forNeon`
- **Reference Benchmark**: 5m14.5s (314.5s) full-length documentary (*AI 发展史*), 55 hand-crafted 16-bit pixel scenes across 5 acts, 155 synced SRT cues, 9 seamless transitions, 3-track audio (Voiceover + 5-act 16-bit BGM with auto-ducking + 94 synchronized 16-bit SFX), and faster-than-real-time client-side **1920×1080 H.264/AAC MP4** export.
- **When starting a new project**: Mirror the module architecture and visual/audio/export standards documented in this skill and its `references/` files. If the user provides a new script, voiceover audio, or SRT file, keep the entire engine, aesthetic constitution, studio UI, and MP4 export pipeline identical while authoring new topic-specific chapters, pixel scenes, sprites, and sound cues.

---

## 1. Aesthetic Constitution (Non-Negotiable)

Think **SNES / Chrono Trigger / Sea of Stars / Celeste** meets **high-craft documentary motion graphics**—warm, rich, atmospheric, and crystal-clear. Never sterile wireframe minimalism, never messy low-res blur, never garish neon slop.

1. **Pure 16-Bit Virtual Pixel Grid (`384×216` Landscape / `216×384` Portrait)**:
   - Draw all pixel worlds, backgrounds, dithered skies, sprites, props, and icons onto an offscreen virtual canvas at **`384×216`** (16:9) or **`216×384`** (9:16).
   - Why `384×216`? It is an **exact 5× integer scale** to `1920×1080` (`384 × 5 = 1920`, `216 × 5 = 1080`). Blit to the HD output canvas with `ctx.imageSmoothingEnabled = false` so every pixel stays razor-sharp with zero shimmering or sub-pixel blur.
   - Lock idle/ambient pixel animations to a crisp retro cadence (`stepT = Math.floor(time * 8..10) / 10`) while camera pans, easing curves, and transitions run at silky **60fps** (30fps on export).

2. **Cohesive 28-Color Warm-Cinematic 16-Bit Palette**:
   - Build every scene from a disciplined shared palette (`PAL`) spanning 6 emotional families: Deep Night Void/Navy, Warm Lamp Gold/Cream, Cozy Coral/Plum, Cool Teal/Pine, Arctic Winter Ice, and Game Boy Olive Monochrome (reserved strictly for dramatic plot-twist hard cuts like `"但其实"`).
   - Never use flat untextured gradients. Use **4×4 Ordered Bayer Dithering** (`drawDitheredSky`, `drawDitheredGlow`) for sky seams, lamp halos, dawn bursts, and pixel dissolves.
   - See `references/visual-and-typography-spec.md` for exact hex tokens, Bayer matrix math, and sprite drawing primitives.

3. **The Golden Typography Law (Zero Muddy CJK Text)**:
   - **Never** draw small Chinese text onto the `384×216` low-res canvas and upscale it—dense Chinese characters (`最/魔/幻/澜/疆`) will smear into an unreadable blob ("糊成一坨").
   - **In-Scene Kinetic Titles & Badges (Dual-Layer HD Overlay)**: Render titles directly onto the HD canvas (`1920×1080`) using binary-snapped pixel glyph sprites rasterized at `38px` (large headlines), `32px` (sub-headlines), or `24px` (HUD tags), with an **8-neighbor 1-pixel dark ink outline**, **stepped drop shadow**, and **3-band metallic bevel fill** (top highlight / midtone / bottom shade), upscaled via nearest-neighbor (`imageSmoothingEnabled = false`).
   - **Lower-Third Broadcast Subtitles**: Keep subtitles in **clean, high-legibility original sans-serif typography** (`Noto Sans SC` / system sans bold inside a subtle dark translucent pill with hairline amber border at `y = 0.905 * h`). Do **not** pixelate the main subtitle strip unless explicitly requested.
   - **Never let dithered glows obscure critical labels**: Any key year, metric, or summit badge (`2026`, `1950 — FUTURE`) must sit inside a crisp high-contrast dark plaque with a golden/teal pixel border drawn **after** background glows.

4. **Zero Hard-Pop Entrances & Zero Accidental Hard Cuts**:
   - **Every scene boundary** must have an intentional transition from the 9-transition library (`aperture-open`, `whip-blinds`, `iris-flash`, `razor-wipe`, `invert-strobe`, `camera-pullback`, `ember-curtain`, `gravity-shatter`, `horizon-burst`) with smooth `fromCam → toCam` camera interpolation and typography crossfade. Even a dramatic reversal uses a deliberate CRT raster wipe (`invert-strobe`), never a jarring unmotivated jump.
   - **Every multi-element graphic inside a scene** (4 modality cards, 4 feature modules, 4 boxes, monolith steps, finale year banner) MUST animate in with purposeful choreography—staggered spring pop-ins, sweeping energy beams connecting hub → node, unfolding RPG plaques, or rolling odometer/counter ticks synced to the spoken words. Never just flip `if (localTime > X) drawBox()`.
   - **Every finale/ending** deserves a signature creative closing sequence (e.g., sunrise horizon bloom + characters looking forward + unfolding plaque + rolling year odometer `1950 → 2026 → FUTURE` + glowing star-arc sweep + warm harmonic chord).

---

## 2. Standard Production Workflow

Follow this exact sequence when creating a new 16-bit pixel video from scratch (or extending an existing one):

### Step 1: Ingest Script, Voiceover & SRT Timing
- Parse the user's real voiceover audio and `.srt` subtitle file (or generate an accurate timestamped SRT aligned to the voiceover).
- Normalize micro-gaps (`< 0.18s`) between adjacent cues so subtitles never flicker off for 2 frames.
- Establish `CLIP_DURATION` from the exact audio/SRT end time.

### Step 2: Storyboard & Beat Sheet (Pacing Blueprint)
- Divide the film into **3–6 Narrative Acts** (each with its own color temperature and BGM chapter) and **3–8 second scenes** (roughly 1–3 subtitle cues per visual beat).
- For long videos (>60s), build iteratively in **30-second chapters** (`retroPixelChapterTwo.ts`, `retroPixelChapterThree.ts`, etc.) so every shot gets genuine hand-crafted pixel art rather than recycled templates.
- Assign each scene in `timeline.ts`:
  - `index`, `id`, `title`, `quote`, `start`, `end` (strictly contiguous, zero gaps/overlaps covering `[0, CLIP_DURATION]`)
  - `transitionIn` + `transitionLabel`
  - `visualBeat` (concise editorial description for the storyboard panel)
  - `accent` hex color for the timeline strip.

### Step 3: Build Core Pixel Engine & Modular Chapter Renderers
Mirror the clean separation of concerns from `https://github.com/Lithium023/16bit-video-forNeon`:
- `src/video/types.ts` — `SrtCue`, `SceneCut`, `TransitionType`, `RenderSettings`, `ExportResolution`, `ExportStatus`.
- `src/video/srtParser.ts` — Deterministic SRT parser, `AUDIO_SRC`, `BGM_SRC`, `CLIP_DURATION`, `getActiveCue`.
- `src/video/timeline.ts` — Contiguous `SCENE_CUTS`, easing curves (`easeOutExpo`, `easeOutBack`, `easeOutCubic`, `easeInOutQuint`, `smoothstep`), `getStaggerProgress`, `getSceneAtTime`, `getActiveTransition`, synthetic fallback voice energy.
- `src/video/pixelSprites.ts` — `PAL`, `BAYER_4X4`, `pxRect`, `pxOutline`, `pxCircle`, `pxRing`, `pxLine`, `drawDitheredSky`, `drawDitheredGlow`, `drawSprite`, `pxBitmapText`, reusable pixel art sprites & environment helpers (`pxPineTree`, `pxCloud`, `pxSparkle`, `pxStarfield`).
- `src/video/pixelTypography.ts` — Cached binary-snapped pixel glyph rasterizer (`drawPixelString`, `drawPixelKineticTitle`, `drawPixelHudTag`).
- `src/video/retroPixelRenderer.ts` — Master virtual-buffer pool (`bufA`, `bufB`, `bufDither`), camera drift calculator, 9 transition compositors, CRT scanlines, typography handoff, and dispatcher to modular `retroPixelChapter*.ts` files.
- `src/video/sceneRenderers.ts` — Top-level `renderVideoFrame`: pixel scene → subtle vignette (`0.14`) → clean original lower-third subtitles → hairline amber progress bar → 0.25s tail fade.

### Step 4: Score the Full Film (3-Track Audio System)
Read `references/choreography-and-audio-spec.md` for full details:
1. **Track 1 — Voiceover**: Master narrative anchor (`volume = 1.0`, 0.25s tail fade).
2. **Track 2 — Multi-Act 16-Bit BGM**: Source genuine open-licensed (CC0 / CC BY 3.0, e.g., OpenGameArt / BoxCat Games / Kevin MacLeod / chiptune OSTs) 16-bit tracks matching each act's mood, stitch them into a single gapless BGM track with smooth crossfades matching the act timestamps, and apply **automatic voiceover ducking** (`duck = 1 - voiceEnergy * 0.22`) at ~`0.58 * bgmVolume`.
3. **Track 3 — Synchronized 16-Bit SFX (`retroSfxEngine.ts`)**: Combine automatic transition SFX for every scene cut + hand-placed musical beat SFX (pentatonic ascending blips for 4-card reveals, crystal clinks, slash buzzes, odometer ticks, and a warm finale major chord). All SFX synthesize cleanly via Web Audio (`BaseAudioContext` + optional `scheduledTime`) so live playback and offline MP4 export sound identical.

### Step 5: Studio NLE Preview & Client-Side 1080p MP4 Export
Read `references/mp4-export-and-qa-spec.md` for full details:
- **Studio Shell**: Dark obsidian workspace (`#07080B`), TopBar (brand title, chapter jump nav, single primary `导出 1080p MP4` action with live stage progress + instant cancel), Canvas stage (16:9 / 9:16 switch, play/pause/scrub/step/speed/mute/fullscreen), multi-track visual timeline (scene blocks + subtitle blocks + playhead), and right editorial panel (`分镜与转场` / `口播台词` / `画面调校` including `1080p` vs `720p` MP4 switch, BGM/SFX sliders, subtitle toggle, highlight jump buttons).
- **MP4 Export (`videoExporter.ts`)**: Use `mp4-muxer` + browser **WebCodecs (`VideoEncoder` H.264 + `AudioEncoder` AAC/Opus)** with offline direct-PCM audio mixing + cached per-SFX stamping, defaulting to **`1920×1080` @ 30fps, 14 Mbps** (`1080×1920` for portrait), with automatic fallback to Safari native `MediaRecorder` MP4.

### Step 6: Unit Tests & Visual Browser Verification (Mandatory)
- Decouple pure logic and test with `node:test` (`npx tsx --test`):
  1. SRT timestamp parsing & cue coverage (`srtParser.test.ts`)
  2. Contiguous scene cuts & transitions with zero gaps/overlaps (`timeline.test.ts`)
  3. Bayer dithering & rectangular sprite grid validation (`pixelSprites.test.ts`)
  4. Chronological SFX cue coverage & forward-only trigger window (`retroSfxEngine.test.ts`)
  5. Export resolution config, MP4 MIME priority, and voice energy timeline (`videoExporter.test.ts`)
- Support `?t=<seconds>` URL query param in `App.tsx` so automated headless browser checks can jump directly to any scene, transition midpoint, animated entrance beat, and ending frame for visual inspection.
- Verify live MP4 export reaches `渲染 MP4`, cancels cleanly back to idle with zero stale callbacks, and emits zero browser console errors.

---

## 3. Hard-Won Rules & Pitfalls to Never Repeat

| Pitfall | Root Cause | Mandatory Solution |
| --- | --- | --- |
| Chinese titles look muddy/clumped ("糊成一坨") | Rendering CJK onto `384×216` low-res canvas then upscaling | Render CJK titles on HD canvas via binary-thresholded `38px/32px` glyph sprites (`thresh = 142..148` for CJK, `88` for ASCII) with 1px ink outline + 3-band bevel; keep lower-third subtitles in clean original sans-serif |
| Hard cuts between late-stage scenes | Missing `transitionIn` or mismatched visual handoff | Every cut in `SCENE_CUTS` must define a purposeful `transitionIn`; pair thematic transitions (e.g. blooming flower dissolving into falling ice crystals → `gravity-shatter` plunge into winter) |
| Elements pop in abruptly | Drawing UI cards/boxes with raw `if (localTime > t)` booleans | Drive every element with `clamp01((localTime - delay) / dur)` + `easeOutBack` / `easeOutExpo`, beam connectors, plaque unfolds, or rolling counters |
| Climax label occluded or half-typed | Glow drawn on top of label, or progress derived from bounded segment counter | Draw labels inside framed dark plaques **after** glows; tie reveal progress directly to `localTime` rather than a capped curve parameter |
| Offline MP4 export stalls at 4% | Connecting 90+ SFX (~280 `BiquadFilterNode`s) to a single 314s `OfflineAudioContext` | Decode voice + BGM once, mix directly in stereo `Float32Array` PCM, render each unique short SFX in a tiny `0.55–1.55s` `OfflineAudioContext` (cached), and stamp into PCM |
| Offline MP4 audio encoding takes 12s+ | Feeding tiny `4096`-sample `AudioData` chunks with loose drain polling | Batch `AudioData` to 1-second (`48000`-sample) chunks (~315 chunks for 5m) and tighten drain fallback to `20ms` |
| Clicking Cancel during MP4 render leaves button stuck | `waitForEncoderDrain` resolves after `cancel()` and emits one last `onProgress` | Wrap `onProgress`/`onComplete`/`onError` in `!cancelled` guards and check `if (isCancelled()) return` immediately after every `await` |

---

## 4. Reference Files in This Skill

Load these references as needed during execution:
- `references/visual-and-typography-spec.md` — Exact `PAL` color palette, `384×216` canvas & integer scaling math, Bayer dithering, pixel typography rasterizer, and 9 transition recipes.
- `references/choreography-and-audio-spec.md` — Animated entrance patterns, 5-act BGM crossfade + ducking math, and 15 procedural 16-bit SFX recipes.
- `references/mp4-export-and-qa-spec.md` — 4-tier client-side MP4 export pipeline, fast offline PCM mixer, studio NLE UI spec, and unit + browser QA checklist.
