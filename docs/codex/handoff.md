# Brush & Ink — handoff

## Objective
Interactive WebGPU brush-and-ink simulation as one self-contained `index.html` (MATERIAL STUDIES / NO. 010).

## Where things are
- `index.html` — the whole app (CSS + JS + WGSL). No build step, no dependencies besides Google Fonts.
- Preview: `.claude/launch.json` in the parent folder has server `brush-ink` → `python3 -m http.server 8917`.

## Architecture (index.html)
- **Brush (CPU, 240 Hz PBD)**: 72 bristles × 10 particles. Per step: pose spring (hand) → integrate → 4 iterations of
  bend-to-rest, weak global rest pull (guarantees recovery), wet cohesion, follow-the-leader length, paper/paperweight/ferrule
  collision → friction + disc-flattening spread for contacting hairs → safety reset (NaN / blow-up).
- **Ink per hair** `INK[i]`, refilled from belly reservoir `brush.res` at per-hair `BR.refill` rates → outer hairs dry first (갈필).
- **Splats**: contacting particles emit capsules (`addCap`), 4 physics steps = 1 sim tick; each tick = one additive splat
  render pass (rgba16f) + 2 compute iterations (A→B with splat, B→A).
- **Paper sim (GPU)**: 2134×1200 rgba32f ping-pong, texel = (surface water, suspended pigment, fixed pigment, dampness).
  Threshold (surface-tension) flow on 8 neighbours with fibre anisotropy, absorption, edge-enhanced evaporation, deposition.
  Only an "active rect" around recent splats is dispatched; it expires 32 s after the last splat.
- **Paper texture**: procedural compute at new-paper time (fibres, clouds, deckle mask, specks) → two rgba16f textures.
- **Render**: fullscreen ray-cast pass for table/felt/paper (writes frag_depth, SDF soft shadows from capsules + rounded box),
  meshes for brush and paperweight, instanced ribbons for bristles, optional skeleton lines. MSAA 4×.
- **Recording**: every physics step's shaped input (x, z, pressure, down) + events (dip, slider, preset) since new paper;
  replay restores a brush snapshot and recomputes from a cleared sheet. MediaRecorder MP4 (fallback WebM).
- **Save**: `window.claude.use('downloads')` when running as a claude.ai artifact (declare `capabilities: {downloads: true}`),
  otherwise an `<a download>` fallback.

## Verified (2026-09-28, Chrome/Electron preview, Apple GPU)
- Demo ensō, strokes at 5/15/40/90 cm/s and pressure 0.15–0.95, dry-brush streaks when ink runs low.
- All three presets, 米字格 grid, skeleton view, wide view (F), mobile portrait stacking (paper rotated).
- Pause, new paper, dip (D), recording → MP4 preview, timelapse (8× + skip idle) → auto preview, save path via stubbed
  downloads capability (saved + declined).
- Replay determinism: brush state identical; exported image mean pixel diff 0.008/255 vs. original.
- Bristles recover rest shape after hard scrubbing; no NaN.
- GPU+physics ≈10 ms/frame at 1440×900 (DPR capped to 3.2 MP canvas).

## Not verified
- Real pen pressure hardware and real touch devices (logic only exercised with synthetic pointer events).
- Browser without WebGPU (fallback message path not exercised), `prefers-reduced-motion` path.
- Downloads confirmation inside the live artifact viewer (published, not clicked through by me).

## iPad / Apple Pencil (2026-09-28)
- Pencil: `penPressure()` soft-knee curve × 필압 gain; tilt (altitude/azimuth or tiltX/Y) → handle lean, recorded per step
  (record stride `REC_F = 6`: x, z, pressure, down, leanX, leanZ; `LEAN_NONE = 9` means no pen lean).
- Coalesced pointer samples are queued and spread across the frame's physics steps (`feedPointer`).
- Palm rejection: touches ignored while the Pencil is down and 600 ms after; first Pencil use turns 손가락으로 쓰기 off.
- Safari gestures: touch-action none, touch-callout none, touchstart/touchmove/gesture* preventDefault on the canvas.
- 렌더 해상도 setting (자동/높게/보통/낮게 = 3.2/2.0/1.1 MP); 자동 starts at 보통 on touch devices and steps down when
  GPU time stays over 22 ms. Export texture is created lazily.
- Save on iPad (outside the claude.ai artifact) uses the share sheet when `navigator.canShare({files})`, else `<a download>`.
- Layout stacks for `(max-width: 820px), (orientation: portrait) and (max-width: 1100px)`; controls ≥ 44 px on coarse pointers.
- Needs device testing: real Pencil pressure/tilt feel, Pencil hover, Safari WebGPU canvas + captureStream recording,
  memory on older iPads, edge-swipe/Scribble behaviour.

## Testing tips
- The in-app Browser pane pauses `requestAnimationFrame` while hidden. `window.__brush.frame(t)` + `__brush.done()` let a
  script drive frames manually; `__brush.peel()` produces the top-down PNG for 1:1 inspection.
- Screenshots of the hidden pane can lag one frame; take two.

## Deploy
- Vercel project `brush-ink` → https://brush-ink.vercel.app (2026-09-28). Redeploy from this folder:
  `npx vercel@latest deploy --prod --yes`. `.vercelignore` keeps `docs/` out of the deploy.

## Next steps
- Try on a pen tablet / iPad and tune `shapeLive` pressure mapping.
- Published 2026-09-28 as a private claude.ai artifact with `capabilities: {downloads: true}`. To update: strip the
  doctype/html/head/body/meta wrapper from index.html into a copy and republish to the same artifact URL (kept in the local memory note, not here).
