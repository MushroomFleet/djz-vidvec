# VIDVEC

**Browser-based motion vector visualization tool.** Drop in a video, extract optical flow frame-by-frame, and export the results as PNG sequences — entirely client-side, no server required.

### [Try the Live Demo](https://scuffedepoch.com/motion-vector/)

VIDVEC computes motion vectors between consecutive video frames using OpenCV.js (WASM) and renders them as directional line overlays on a black canvas. The output is useful for motion analysis, video research, VFX reference, AI training data preparation, and anyone who needs to see how pixels move between frames.

Everything runs in your browser — no uploads, no server, no data leaves your machine.

---

## Features

- **Drag-and-drop video loading** — supports MP4, WebM, and MOV
- **Two optical flow algorithms:**
  - **Sparse (Lucas-Kanade)** — tracks feature points across frames using `goodFeaturesToTrack` + `calcOpticalFlowPyrLK`, with configurable feature count and periodic re-seeding
  - **Dense (Farneback)** — computes per-pixel flow fields via `calcOpticalFlowFarneback` and samples on a configurable grid
- **Live preview** — scrub through the video timeline and see vector output for any frame before committing to a full render
- **Colour modes** — white vectors on black, or HSL-mapped by motion angle for directional visualization
- **Configurable output resolution** — scale output independently from source (0.1–2.0 megapixels)
- **Batched ZIP export** — output PNGs are bundled into ~250 MB ZIP files that download automatically during processing, keeping memory usage low and downloads reliable
- **Automatic FPS detection** — samples the video at 120 Hz resolution and snaps to common framerates (23.976, 24, 25, 29.97, 30, 50, 59.94, 60)
- **Persistent settings** — all parameters saved to `localStorage` and restored between sessions
- **Fully client-side** — all computation runs in the browser via OpenCV.js WASM; no data leaves your machine

## How It Works

```
Video File
    │
    ▼
┌─────────────────┐
│  Frame Stepper   │  Seeks to each frame using HTMLVideoElement
│  (useFrameStepper)│  and reads pixel data via Canvas2D
└────────┬────────┘
         │ ImageData (frame N-1, frame N)
         ▼
┌─────────────────┐
│  Optical Flow    │  Sparse: Lucas-Kanade pyramid tracking
│  (sparseFlow /   │  Dense:  Farneback polynomial expansion
│   denseFlow)     │  → outputs Vector2D[] {x, y, dx, dy}
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Renderer        │  Draws vectors as lines on a black canvas
│  (renderer.ts)   │  Scales source→output coordinates
│                  │  Optional HSL colouring by angle
└────────┬────────┘
         │ Canvas frame (PNG blob)
         ▼
┌─────────────────┐
│  Batch Exporter  │  Accumulates blobs up to 250 MB
│  (zipExporter.ts)│  then zips + downloads automatically
│                  │  Repeats until all frames are exported
└─────────────────┘
```

## Settings

| Parameter | Range | Description |
|---|---|---|
| **Algorithm** | Sparse / Flow Arrows | Sparse tracks feature points; Flow Arrows uses dense grid sampling |
| **Max Features** | 10–2000 | Number of corners to track (Sparse only) |
| **Min Magnitude** | 0–10 | Ignore vectors below this pixel displacement |
| **Length Scale** | 0.5–20 | Multiplier for rendered line length |
| **Max Line px** | 5–200 | Hard cap on line length in output pixels |
| **Grid Step** | 4–64 | Sampling interval for dense flow (Flow Arrows only) |
| **Line Thickness** | 1–5 | Stroke width in pixels |
| **Colour by Angle** | on/off | White lines vs HSL-mapped by direction |
| **Output Res MP** | 0.1–2.0 | Output resolution in megapixels (aspect ratio preserved) |
| **Reseed Interval** | 5–120 | Re-detect features every N frames (Sparse only) |

## Usage

1. Open the [live demo](https://scuffedepoch.com/motion-vector/)
2. Drop a video file (MP4, WebM, or MOV) onto the page — or click to browse
3. Scrub the timeline to preview vector output for any frame
4. Adjust settings in the right panel (algorithm, feature count, line style, output resolution, etc.)
5. Click **Process** to render all frames
6. ZIP files download automatically in ~250 MB batches as processing runs

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | React 18 |
| Language | TypeScript 5 |
| Build | Vite 5 |
| Styling | Tailwind CSS 3 |
| Computer Vision | OpenCV.js 4.12 (WASM) |
| ZIP Export | JSZip + FileSaver |

## Output Format

Each processed frame is exported as a PNG image:
- **Filename pattern:** `frame_000001.png`, `frame_000002.png`, ...
- **Background:** solid black (`#000000`)
- **Vectors:** white lines (or HSL-coloured) from origin point in direction of motion
- **Resolution:** determined by the Output Res MP setting

ZIP files are named `vidvec_{videoname}_{timestamp}.zip` with `_part002`, `_part003` suffixes for multi-part exports.

## License

This project is provided as-is for research and creative use.

---

## 📚 Citation

### Academic Citation

If you use this codebase in your research or project, please cite:

```bibtex
@software{djz_vidvec,
  title = {VIDVEC: Browser-Based Motion Vector Visualization Tool},
  author = {Drift Johnson},
  year = {2025},
  url = {https://github.com/MushroomFleet/djz-vidvec},
  version = {1.0.0}
}
```

### Donate:

[![Ko-Fi](https://cdn.ko-fi.com/cdn/kofi3.png?v=3)](https://ko-fi.com/driftjohnson)

---

## Support This Project

If you found this useful, please **star the repo** — it helps others discover it!

[![Star on GitHub](https://img.shields.io/github/stars/MushroomFleet/djz-vidvec?style=social)](https://github.com/MushroomFleet/djz-vidvec)