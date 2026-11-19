# Client-Side 1080p MP4 Export, Studio UI & QA Specification

Reference implementation: `https://github.com/Lithium023/16bit-video-forNeon` (`src/video/videoExporter.ts`, `src/App.tsx`, `src/components/TopBar.tsx`, `src/components/EditorialPanel.tsx`, `src/components/TimelineBar.tsx`).

---

## 1. Client-Side Faster-Than-Real-Time MP4 Export (`mp4-muxer` + WebCodecs)

Install `mp4-muxer` (`npm install mp4-muxer`). Export runs 100% client-side in static Vite SPAs—no backend FFmpeg required.

### Default Export Specs (`getExportConfig`)
- **1080p Full HD (Default)**: `1920 × 1080` (16:9) or `1080 × 1920` (9:16), `30 fps`, video `14 Mbps`, audio `192 kbps` (`48kHz` stereo). Exact `5×` integer scale from `384 × 216`.
- **720p Standard (Optional)**: `1280 × 720` (16:9) or `720 × 1280` (9:16), `30 fps`, video `6.5 Mbps`, audio `160 kbps`.
- **Filename convention**: `<ProjectName>_16bit完整成片_<width>x<height>.mp4`.

### 4-Tier Browser Compatibility Pipeline
1. **Tier 1 (Primary — Chrome / Edge / Modern Desktop)**: Offline **WebCodecs `VideoEncoder` (H.264 `avc1.64002a` / `avc1.640028` / `avc1.4d002a` / `avc1.42002a`, `{ avc: { format: 'avc' } }`) + `AudioEncoder` (AAC `mp4a.40.2`, falling back to `opus`) + `mp4-muxer` (`fastStart: 'in-memory'`)**. Produces universally playable `.mp4` (QuickTime, Windows, Premiere, 剪映, Bilibili) faster than real time with zero dropped frames.
2. **Tier 2 (Safari / WebKit)**: Native `MediaRecorder` probing `video/mp4;codecs=avc1...` first (`selectSupportedMediaRecorderMime`).
3. **Tier 3 (Open-Source Chromium without H.264)**: Offline WebCodecs `VP9` + `Opus` muxed into ISO BMFF `.mp4` via `mp4-muxer`.
4. **Tier 4 (Final Safety Net)**: `MediaRecorder` `video/webm` only if zero MP4 encoders exist.

### 3 Critical Engineering Optimizations (Mandatory!)

1. **Fast Direct-PCM Offline Mixing + Cached Tiny Per-SFX Contexts**:
   - **NEVER** connect 90+ SFX cues (~280 `BiquadFilterNode`s) to a single 5-minute `OfflineAudioContext`—Chromium evaluates every filter across all ~15M samples and stalls for minutes!
   - Instead:
     1. Decode `AUDIO_SRC` and `BGM_SRC` in parallel at `48kHz` using a 1-sample `OfflineAudioContext`.
     2. Compute `voiceEnergies = computeVoiceEnergyTimeline(...)` once per video frame (reused for both canvas audio-reactivity AND smooth BGM ducking).
     3. Mix voiceover + ducked BGM + tail fades directly into two `Float32Array(totalAudioSamples)` arrays (`mixedL`, `mixedR`) in ~30ms.
     4. For each SFX cue, cache its rendered mono PCM by `${cue.kind}|${cue.freq ?? ''}|${cue.gain ?? ''}` using a tiny `0.55–1.55s` `OfflineAudioContext` (`playSfxCue(sfxCtx, sfxCtx.destination, cue, sfxVolume, 0)`), then stamp the cached float array directly into `mixedL`/`mixedR` at `Math.floor(cue.time * sampleRate)`.
     5. Apply a gentle `0.95` master ceiling clamp (`[-0.98, 0.98]`) so peaks never clip in AAC/Opus.

2. **1-Second Batched `AudioData` Encoding**:
   - Feed `AudioEncoder` with `audioChunkSize = profile.audioSampleRate` (`48000` samples = 1.0s per `AudioData`, ~315 calls for a 5m film instead of 3,685).
   - Drain when `encodeQueueSize > 24` using `waitForEncoderDrain` with a `20ms` fallback timeout.

3. **Race-Free Instant Cancellation**:
   - Inside `startVideoExport`, wrap `onProgress`, `onComplete`, and `onError` with `if (!cancelled)` guards (`safeProgress`, `safeComplete`, `safeError`).
   - Check `if (isCancelled()) return;` immediately after every `await` (especially after `waitForEncoderDrain`), and register encoder `.close()` cleanup so clicking Cancel immediately resets the UI to `导出 1080p MP4` with zero zombie progress ticks.

### Live Stage Progress Copy
Emit clear Chinese stages to the TopBar export pill:
`准备 MP4` → `混音音轨` → `合成音效` → `编码音频` → `渲染 MP4` → `封装 MP4` → `完成` (auto-triggers browser file download + keeps a reusable download banner in「画面调校」).

---

## 2. Studio NLE Workspace Layout

Keep the editing/preview workspace clean, dark (`#07080B`), zero telemetry clutter, zero multi-style switches once locked to 16-bit:

- **TopBar (`h-14`)**:
  - Left: Single-line title (`<主题> · 16-Bit 成片`)
  - Center: Direct chapter jump navigation with active amber bottom underline (`h-[2px] bg-amber-500`)
  - Right: Single primary amber CTA (`导出 1080p MP4` / `导出 720p MP4`), transforming into an active spinner + `{stage} · {pct}%` + `X` cancel button while exporting.
- **Left Stage + Multi-Track Timeline**:
  - Centered responsive Canvas preview (`16:9` or `9:16`) with floating subtle scene badge (`03 / 主标题落版 · 金色光圈扩镜`), mute/fullscreen controls, and hover play/pause overlay.
  - Transport bar: Play/Pause, Restart, Step -1s/+1s, tabular monospace timecode (`MM:SS.CC / MM:SS.CC`), playback rate (`0.75x / 1x / 1.25x`), aspect ratio toggle (`16:9 / 9:16`), mute, fullscreen.
  - Multi-track clickable/scrubbable timeline: Time ruler + colored Scene Cut blocks + SRT Subtitle blocks + amber vertical playhead.
  - Keyboard shortcuts: `Space` (Play/Pause), `ArrowLeft`/`ArrowRight` (±1s), `R` (Restart), `M` (Mute).
- **Right Editorial Panel (3 Tabs)**:
  1. `分镜与转场`: Scrollable list of all scenes with index, timestamp, spoken quote, transition label, and visual beat summary; clicking jumps immediately to that scene.
  2. `口播台词`: Full SRT cue list with click-to-seek and active highlight.
  3. `画面调校`: Aspect ratio cards (`16:9` / `9:16`), **导出分辨率 (MP4)** cards (`1080p 全高清` default / `720p 标准`), motion intensity slider, BGM volume slider, SFX volume slider + attribution note, subtitle toggle, highlight jump chips, and completed MP4 download banner.

---

## 3. Verification & QA Checklist (Before Declaring Done)

1. **Pure Logic Unit Tests (`npx tsx --test`)**:
   - `srtParser.test.ts`: Exact timestamp conversion, full cue count, micro-gap bridging.
   - `timeline.test.ts`: Zero gaps/overlaps across `[0, CLIP_DURATION]`, key story beat lookup, transition detection around cut boundaries, easing & stagger bounds.
   - `pixelSprites.test.ts`: Bayer threshold distribution `(0, 1)` and uniform row width for every sprite & 5×7 glyph.
   - `retroSfxEngine.test.ts`: Chronological `SOUND_CUES`, every scene covered, forward-only playback trigger (scrubs return `[]`).
   - `videoExporter.test.ts`: `1920×1080` & `1080×1920` default config, `1280×720` option, MP4 prioritized over WebM in `selectSupportedMediaRecorderMime`, normalized `[0, 1]` `computeVoiceEnergyTimeline`.
2. **Build & Type Check**: Co-run `npm run build` and `npm run lint` (`tsc --noEmit`) with zero errors.
3. **Real Browser Visual Inspection (via `?t=<sec>` query param at `1440×900`)**:
   - Spot-check opening shot, title card (verify crisp CJK pixel glyphs & clean original lower-third subtitles), mid-film plot twist / winter / climax scenes, **every multi-module entrance beat** (at start + mid + settled timestamps), and the final 3 seconds of the ending.
   - Click `导出 1080p MP4`, verify stages advance smoothly through `混音音轨` → `合成音效` → `编码音频` → `渲染 MP4`, click Cancel and verify instant reset to `导出 1080p MP4` with **0 console errors**.
