# Visual, Palette, Typography & Transition Specification

Reference implementation: `https://github.com/Lithium023/16bit-video-forNeon` (`src/video/pixelSprites.ts`, `src/video/pixelTypography.ts`, `src/video/retroPixelRenderer.ts`, `src/video/sceneRenderers.ts`).

---

## 1. Virtual Pixel Canvas & Exact Integer Scaling

Always render pixel art on a dedicated offscreen buffer before blitting to the HD display/export canvas:

| Aspect Ratio | Virtual Pixel Resolution | HD 1080p Output | Integer Scale Factor |
| --- | --- | --- | --- |
| `16:9` Landscape | `384 × 216` | `1920 × 1080` | **Exact `5×`** |
| `9:16` Portrait | `216 × 384` | `1080 × 1920` | **Exact `5×`** |

- Always set `ctx.imageSmoothingEnabled = false` on **both** the virtual offscreen contexts (`bufA`, `bufB`, `bufDither`) and the final HD context right before blitting.
- Round all virtual pixel coordinates (`Math.round(x)`, `Math.round(y)`) inside drawing primitives (`pxRect`, `pxOutline`, `pxCircle`, `pxLine`, `drawSprite`) so no anti-aliased edge ever leaks into the pixel buffer.
- Maintain subtle **CRT scanlines** on the HD canvas after blitting: horizontal `1px` dark bands (`rgba(0, 0, 0, 0.055)`) every `Math.max(4, Math.round(h / 216))` pixels, followed by a gentle edge vignette (`0.14`).

---

## 2. Canonical 28-Color 16-Bit Palette (`PAL`)

Every scene draws from these harmonious, low-fatigue SNES/RPG tones:

```ts
export const PAL = {
  // Deep night & interior shadows (Studio, Starry Skies, Mainframes)
  void: '#10121E',
  navyDark: '#171B2C',
  navy: '#21273F',
  navyLight: '#313A59',
  slate: '#49557A',
  slateLight: '#7382A8',

  // Warm lamp / gold / amber highlights (Title, Breakthroughs, Dawn, Finale)
  cream: '#F6EED8',
  creamDim: '#D4C7A8',
  goldLight: '#FFE08A',
  gold: '#F2B44B',
  goldDark: '#C27E28',
  amber: '#965222',

  // Cozy coral / crimson accents (Warnings, Red Slashes, Dramatic Sunsets)
  coralLight: '#F58E7E',
  coral: '#E06055',
  coralDark: '#99363B',
  plum: '#4A2B45',
  dusk: '#6E3E52',

  // Cool teal / cyan / emerald (Data, Neural Nets, CRT Phosphor, Circuitry)
  tealLight: '#7ED4C0',
  teal: '#4AA092',
  tealDark: '#2A6360',
  pine: '#1E454B',

  // Ice / snow / arctic winter (Setbacks, Funding Cuts, AI Winters)
  iceWhite: '#ECF7FC',
  iceLight: '#A8DDF2',
  ice: '#6BB0D6',
  iceMid: '#3C6E99',
  iceDark: '#1E3A5C',
  iceDeep: '#11223B',

  // Game Boy Pocket olive monochrome (Plot-Twist Reversal Scenes)
  gbLight: '#D8E3C0',
  gbMid: '#9EB386',
  gbDark: '#486348',
  gbInk: '#1D2B1F',
} as const;
```

### Color Direction by Narrative Emotion
- **Cozy Prologue / Studio**: `#181C2E → #21253B → #2B2B3F` walls, warm `goldLight` desk lamp glow, `tealLight` CRT screen.
- **Discovery / Optimism / Dawn**: `navyDark → navy → plum → dusk → gold` skies with `goldLight` sparks and `cream` titles.
- **Plot Twist / Reversal ("但其实")**: Full monochrome Game Boy olive screen (`gbLight` / `gbMid` / `gbDark` / `gbInk`) for unmistakable instant visual contrast.
- **Crisis / Overload / Winter**: `iceDeep → iceDark → iceMid` skies, `iceWhite` falling snow/crystals, `coral` alert stamps.
- **Data / Deep Learning / Modern Era**: `void → navyDark → pine` with `tealLight` data streams, glowing `gold` neural synapses, and multi-modal `coralLight` / `tealLight` / `goldLight` accents.
- **Grand Finale**: Warm sunrise gradient (`navyDark → plum → dusk → amber → goldLight`), silhouetted human figures facing the horizon, golden star-arc sweep.

---

## 3. Authentic 16-Bit Dithering & Sprite Primitives

### 4×4 Ordered Bayer Matrix
```ts
export const BAYER_4X4 = [
  [0, 8, 2, 10],
  [12, 4, 14, 6],
  [3, 11, 1, 9],
  [15, 7, 13, 5],
];
export const bayerThreshold = (x: number, y: number) =>
  (BAYER_4X4[(y | 0) & 3][(x | 0) & 3] + 0.5) / 16;
```
- **Multi-Band Dithered Sky (`drawDitheredSky`)**: Render solid color bands, then add `4–12px` ordered checkerboard/Bayer transition rows immediately above each seam (`frac > 0.66`, `> 0.34`, else sparse).
- **Radial Pixel Halos (`drawDitheredGlow`)**: Sample `(dx, dy)` in `step = 2` (medium) or `3` (light) grids and plot single pixels where `(1 - d / r) > bayerThreshold(dx + cx, dy + cy) * 1.35`. Never draw blurred CSS/Canvas radial gradients on the virtual pixel buffer!
- **ASCII Grid Sprites (`drawSprite`)**: Define hero props/characters as rectangular `string[]` pixel grids (every row MUST have identical length — enforce with a unit test!), mapped through a per-sprite palette dictionary where `.` and space are transparent.
- **5×7 Pixel Bitmap Labels (`pxBitmapText`)**: Use for tiny diegetic in-world labels inside the `384×216` canvas (years `1956`, `1997`, `2026`, `COST $999K`, `CAT 99%`, `TEXT`, `VIDEO`, `AI`). Keep strictly uppercase ASCII/digits; never try to draw 5×7 Chinese characters!

---

## 4. Dual-Layer Typography Architecture

Load Google Fonts in `index.html`:
- Display / Subtitles: `Noto Sans SC` (weights 400, 500, 700, 900), `Outfit`
- Retro Pixel Latin / Numerals: `Silkscreen`, `JetBrains Mono`

### Layer A: In-Scene 16-Bit Pixel Kinetic Titles (`pixelTypography.ts`)
Drawn directly onto the HD canvas (`1920×1080`) **after** CRT scanlines so Chinese strokes stay razor-sharp while looking unmistakably 16-bit:
1. Rasterize each individual character onto a small offscreen canvas at:
   - `gridPx = 38` for giant headlines (`fontSize >= 90`)
   - `gridPx = 32` for medium titles
   - `gridPx = 24` for HUD badges (`drawPixelHudTag`)
2. Binarize alpha with **strict thresholds** (no gray anti-alias halo!):
   - CJK characters: `alpha >= 142` (for `<=24px`) or `148` (keeps tight internal counters open!)
   - ASCII characters (`Silkscreen`): `alpha >= 88` (keeps thin horizontal strokes solid)
3. Composite 3 pixel passes into the cached glyph sprite:
   - **Pass 1**: Stepped drop shadow (`+2,+2` for titles, `+1,+1` for badges)
   - **Pass 2**: 8-neighbor `1px` dark ink outline (`PAL.void`)
   - **Pass 3**: 3-band metallic 16-bit fill (`yf < 0.28` top highlight, middle tone, `yf > 0.78` bottom shade)
4. Cache in an LRU `Map` (max ~450 glyphs) and blit onto the HD canvas with `imageSmoothingEnabled = false` and per-character staggered `easeOutBack(rawP, 1.35)` pop-in + `easeOutExpo` vertical drop.

### Layer B: Clean Original Broadcast Subtitles (`sceneRenderers.ts`)
- Render at `y = h * 0.905` (landscape) / `h * 0.885` (portrait).
- Font: `700 ${Math.round(h * 0.032)}px "Noto Sans SC", sans-serif` in crisp `#F4F4F5`.
- Container: `rgba(7, 8, 11, 0.74)` rounded rect (`radius = 8`), hairline `rgba(245, 158, 11, 0.35)` border (`1.2px`), smooth `0.16s` ease-out fade-in and `0.12s` fade-out.
- **Rule**: Keep the lower-third safe zone (`y > 0.83 * h`) free of critical pixel focal points so subtitles never collide with key scene artwork.

---

## 5. Signature 9-Transition Library

All transitions interpolate between `bufA` (outgoing scene at `fromCam`) and `bufB` (incoming scene at `toCam`) using `easeInOutQuint(p)` over `0.36s–0.55s`, while crossfading in-scene typography (`outAlpha = smoothstep(0.46, 0.04, p)`, `inAlpha = smoothstep(0.32, 0.78, p)`):

1. **`aperture-open`**: CRT power-on horizontal scan beam expanding vertically into full frame (ideal for Scene 1 cold open).
2. **`whip-blinds`**: Zoom-through / portal push-in (zooms `bufA` forward while revealing `bufB` through expanding frame).
3. **`iris-flash`**: Golden circular RPG iris wipe expanding from screen center with a glowing `PAL.goldLight` ring edge.
4. **`razor-wipe`**: Continuous 60fps horizontal camera whip-pan left→right with subtle cream speedlines at peak velocity.
5. **`invert-strobe`**: Silky vertical CRT raster wipe top→bottom with a luminous pale-green/white scan beam (ideal for dramatic `"但其实"` plot twists without jarring strobe flicker).
6. **`camera-pullback`**: Authentic 16-bit **Ordered Bayer 4×4 Pixel Dissolve** (`composeBayerDissolve`) smoothly crossfading pixel-by-pixel between `bufA` and `bufB`.
7. **`ember-curtain`**: Continuous vertical camera tilt-up with warm amber seam glow (ideal for rising rockets, growing trees, dawn reveals).
8. **`gravity-shatter`**: Continuous vertical camera plunge-down with icy cyan seam glow (ideal for falling into winter/crises/collapses).
9. **`horizon-burst`**: Warm dawn horizon parting + Bayer dissolve + golden radial sunrise wash (ideal for major era awakenings and grand finale entry).
