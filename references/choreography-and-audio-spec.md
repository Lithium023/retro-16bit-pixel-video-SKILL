# Motion Choreography, Pacing & 3-Track Audio Specification

Reference implementation: `https://github.com/Lithium023/16bit-video-forNeon` (`src/video/timeline.ts`, `src/video/retroPixelChapterEight.ts`, `src/video/retroSfxEngine.ts`).

---

## 1. Scene Pacing & Composition Discipline

- **Beat Cadence**: Average **3.5–7.5 seconds per scene** (1–3 SRT cues). Short dramatic reversals (`"但其实"`, `"昙花一现"`) can be `0.8–2.0s`; major narrative tableaux (`Transformer + ChatGPT`, `World Tree`, `Finale`) run `6–11s` with multi-stage internal reveals timed to exact spoken phrases.
- **Ambient Motion**: Every pixel scene must feel alive even during holds:
  - Parallax drifting clouds/stars (`pxStarfield`, `pxCloud`)
  - Subtle camera Ken Burns drift (`scale 1.00 → 1.04`, gentle `dx/dy` pan scaled by `motionIntensity`)
  - Audio-reactive subtle accents (`energy` drives waveform bars, CRT phosphor pulse, fireplace/ember flicker, neural pulse brightness)
  - Stepped pixel animations (`Math.floor(time * 8)` for walk cycles, steam curls, blinking LEDs, bobbing cursors).

---

## 2. Zero Hard-Pop Entrance Choreography Patterns

Whenever a scene introduces multiple modules, cards, pillars, or a closing emblem, **never** toggle visibility with raw `if (localTime > t)`. Always pick from these animated entrance recipes:

1. **Hub-to-Card Energy Beam + Spring Pop-In** *(e.g., 4 Multimodal Cards: TEXT / IMAGE / VIDEO / AUDIO)*:
   - Central hub activates first (`easeOutExpo`).
   - For each satellite card `i`, draw an animated pixel energy beam growing from hub → card (`beamP = clamp01((localTime - delay) / 0.25)`), followed by the card popping in with `scale = lerp(0.4, 1.0, easeOutBack(cardP))`, pixel slide offset, and a brief white/gold flash highlight border.
2. **Staggered Orbital / Radial Bloom** *(e.g., breaking out of an old paradigm into 4 new capabilities)*:
   - First animate the old paradigm exiting (e.g., dramatic red diagonal slash + shake + fade-out).
   - Then bloom each new module outward along its vector from center (`lerp(center, target, easeOutBack(p))`) synced to the exact timestamps when the narrator speaks each item!
3. **Organic Growth + Ripening Fruit Badges** *(e.g., spark growing into a World Tree with 4 industry badges)*:
   - Trunk grows upward (`height * easeOutCubic(p)`), canopy expands in layered pixel puffs, then each badge sprouts outward from its branch tip with a tiny stem + sparkle burst (`pxSparkle`) staggered by `0.35–0.40s`.
4. **Rising Architectural Monoliths / Timeline Curve**:
   - Monoliths rise sequentially from the ground plane (`y = baseY + (1 - easeOutBack(p)) * h`) with rising pixel dust sparks and crowning icons that lock in at apex.
   - Curves draw progressively along segments (`segProgress`), with valley crystals precipitating during downturns and a glowing golden beacon + **high-contrast framed badge** locking at the summit (progress driven directly by `localTime` so it always reaches 100%).
5. **Grand Finale Closing Sequence** *(never a static title card!)*:
   - Sunrise horizon light + drifting star dust + human silhouettes witnessing the dawn.
   - Center RPG plaque unfolds horizontally (`plaqueW * easeOutExpo(p)`) with corner rivets locking in.
   - Rolling **year odometer** (`1950 → 1970 → 1997 → 2016 → 2026`) synced to 5 ascending tick SFX, culminating in a bright flash reveal of **`FUTURE`** (or the film's final motif) with a sweeping golden star-pen arc and lingering sparkles.

---

## 3. Three-Track Audio Architecture

A silent or monotone-beeping pixel video feels amateur. Always ship all three balanced tracks:

### Track 1: Voiceover (`AUDIO_SRC`)
- Master clock during live playback (`volume = 1.0`).
- Include stall-resilient fallback in the React playback loop: if the browser audio clock stalls `> 180ms` (e.g., headless autoplay restrictions), seamlessly fall back to `performance.now()` visual clock.
- Smooth `0.25s` tail fade at `CLIP_DURATION - 0.25`.

### Track 2: Multi-Act 16-Bit BGM (`BGM_SRC`)
- **Licensing**: Always source genuinely open-licensed 16-bit / chiptune tracks (CC0 or CC BY 3.0, e.g., OpenGameArt / BoxCat Games *Nameless: The Hackers RPG* OST) and document attribution cleanly in the Controls panel.
- **Act-Matched Stitching**: Pick 3–5 distinct moods matching the documentary's emotional arc (e.g., Wonder/Prologue → Curious Retro Lab → Melancholy Winter → Rising Momentum → Heroic Grand Finale) and crossfade them (`1.5–2.5s` smooth equal-power fades) into a single pre-mixed `public/assets/...-bgm.mp3` matching `CLIP_DURATION`.
- **Dynamic Voiceover Ducking**:
  - Default UI mix: `bgmVolume = 0.36`, effective base gain = `bgmVolume * 0.58`.
  - Duck smoothly whenever voiceover is active: `duck = 1 - voiceEnergy * 0.22`.
  - Smooth `1.5s` musical tail fade-out at the end of the film.

### Track 3: Procedural 16-Bit Web Audio SFX (`retroSfxEngine.ts`)
Synthesize warm, non-fatiguing SNES-era sound effects directly in TypeScript via Web Audio API—zero external SFX downloads needed, sample-accurate sync, and tiny bundle size:

1. **Low-Pass Warmth Rule**: Route every square/sawtooth/triangle oscillator through a `4200Hz` low-pass `BiquadFilterNode` with a soft `10–15ms` attack and exponential release so chiptune SFX sound warm and musical rather than harsh/piercing.
2. **Stepped 16-Bit Noise Buffer**: Cache a `0.8s` quantized pseudo-random noise buffer (`sample-and-hold` every 3 samples quantized to 16 levels) per `BaseAudioContext` via `WeakMap<BaseAudioContext, AudioBuffer>` for authentic retro whooshes, winter gusts, and CRT static.
3. **Dual-Mode Signature (`BaseAudioContext` + `scheduledTime`)**:
   ```ts
   export function playSfxCue(
     ctx: BaseAudioContext,
     destination: AudioNode,
     cue: SfxCue,
     masterSfxVolume: number,
     scheduledTime?: number
   ): void
   ```
   - Live playback calls `playSfxCue(audioCtx, audioCtx.destination, cue, sfxVolume)` when `collectTriggeredSfxCues(prevT, nextT)` catches cues in `(prevT, nextT]` (ignoring scrubs/seeks where `delta <= 0 || delta > 0.35`).
   - Offline MP4 export calls `playSfxCue(sfxCtx, sfxCtx.destination, cue, sfxVolume, 0)` inside tiny cached per-SFX `OfflineAudioContext`s!
4. **Core 15-Preset SFX Vocabulary**:
   - **Transitions (auto-mapped from `scene.transitionIn`)**:
     - `crt-boot` (`aperture-open`): warm triangle thump `110→52Hz` + phosphor sine chime + octave square blip
     - `zoom-whoosh` (`whip-blinds`): rising bandpass noise `340→2800Hz` + triangle sweep
     - `iris-chime` (`iris-flash`): 3-note ascending major arpeggio
     - `swoosh` (`razor-wipe`): crisp bandpass noise `520→2400Hz`
     - `tape-rewind` (`invert-strobe`): fast upward pitch zip + filtered static
     - `winter-gust` (`gravity-shatter`): cold resonant low-bandpass wind sweep + descending minor tone
     - `ember-swell` (`ember-curtain`): warm rising fifth chord shimmer
     - `horizon-bloom` (`horizon-burst`): lush open-fifth + octave warm swell
     - `pixel-shimmer` (`camera-pullback`): delicate crystalline sparkle pair
   - **Signature Story & Animation Beats (hand-timed to frame animations)**:
     - `blip-note`: pentatonic ascending notes (`C5 / D5 / E5 / G5 / A5`) for 4-card / 4-pillar staggered reveals
     - `crystal-clink`: bright bell harmonics (`880Hz + 1320Hz`) for Go stones, tree fruit badges, winter crystals
     - `slash-buzz`: gritty downward saw/square swipe for red X rejections or system overloads
     - `power-surge`: rising synth sweep for rockets, GPUs, breakthroughs
     - `tick`: crisp short mechanical clicks ascending in pitch (`660 → 1046.5Hz`) for odometer/year counters
     - `finale-chord`: rich `1.4s` major-9th chiptune chord swell for the final reveal!
