<!-- TINS Specification v1.0 -->
<!-- ZS:COMPLEXITY:HIGH -->
<!-- ZS:PRIORITY:HIGH -->
<!-- ZS:PLATFORM:WEB -->
<!-- ZS:LANGUAGE:TYPESCRIPT -->

# VIDVEC

## Description

VIDVEC is a browser-based motion vector visualisation tool built with Vite, TypeScript, and React. It accepts a video file as input, processes it entirely client-side using the HTML5 Canvas API and WebAssembly-accelerated optical flow computation, and outputs a downloadable image sequence of directional line drawings rendered on a pure black field — one PNG per frame.

The primary use case is analysing talking-head video in side-by-side debate and multi-screen conference formats. Motion vectors rendered as directional lines on black reveal communication dynamics: speakers animate, non-speakers hold or micro-shift, and reaction patterns emerge spatially across the frame. The resulting sequence is designed for frame-accurate compositing over the original video in an external tool (e.g. FFmpeg, DaVinci Resolve, After Effects) using screen or add blend modes.

VIDVEC implements two motion extraction algorithms the user can switch between:

1. **Sparse Flow (default):** Lucas-Kanade optical flow tracking Shi-Tomasi corner features. Best for talking-head analysis — attaches to face and body feature points, producing clean, semantically meaningful vector trails.
2. **Flow Arrows:** Dense optical flow via Farnebäck algorithm downsampled to a regular grid, then rendered as directional lines. Inspired by flowvid's `flow_arrows` preset. Best for capturing full-frame motion fields including background.

All processing is done in the browser. No server, no upload, no dependencies beyond the bundled application.

---

## Functionality

### Application Layout

```
+================================================================+
|  VIDVEC                                          v1.0          |
+================================================================+
|                                                                |
|   +---------------------------+   +-------------------------+  |
|   |                           |   |   SETTINGS              |  |
|   |   VIDEO PREVIEW           |   |                         |  |
|   |   (canvas element)        |   |  Algorithm              |  |
|   |                           |   |  ○ Sparse (LK)  [●]    |  |
|   |   [scrub bar]             |   |  ○ Flow Arrows          |  |
|   |                           |   |                         |  |
|   |   frame 042 / 312         |   |  Max Features    [ 300] |  |
|   +---------------------------+   |  Min Magnitude   [ 0.5] |  |
|                                   |  Length Scale    [ 3.0] |  |
|   +---------------------------+   |  Max Line px     [  40] |  |
|   |                           |   |  Grid Step       [  16] |  |
|   |   VECTOR PREVIEW          |   |  Line Thickness  [   1] |  |
|   |   (canvas element)        |   |  Colour by Angle [ off] |  |
|   |   black field + lines     |   |  Output Res MP   [ 0.5] |  |
|   |                           |   |                         |  |
|   |   frame 042               |   |  +---------+            |  |
|   +---------------------------+   |  | PROCESS |            |  |
|                                   |  +---------+            |  |
|  Drop video here or [Choose File] |                         |  |
|                                   |  Progress: ████░░ 68%   |  |
|  [PROCESS]  [DOWNLOAD ZIP]        |  312 frames • 00:08 rem |  |
|                                   +-------------------------+  |
+================================================================+
```

### Core User Flow

1. User opens the app in any modern browser (Chrome, Firefox, Edge, Safari).
2. User drops a video file or clicks "Choose File" to load a local MP4, WebM, or MOV file.
3. The video preview canvas renders the first frame immediately. A scrub bar allows frame-by-frame inspection.
4. The vector preview canvas shows a live preview of the vector render for the current scrubbed frame using current settings.
5. User selects algorithm (Sparse LK or Flow Arrows) and tunes parameters via the settings panel. The vector preview updates in real time (debounced 200ms) as settings change.
6. User clicks PROCESS. The app processes all frames sequentially, updating a progress bar. The vector preview canvas shows each frame as it is rendered.
7. On completion, the DOWNLOAD ZIP button activates. Clicking it packages all PNG frames into a ZIP archive and triggers a browser download named `vidvec_[original-filename]_[timestamp].zip`.
8. The user can load a new video at any time, which resets all state.

### Video Loading

- Accepted formats: MP4 (H.264, H.265), WebM (VP8, VP9), MOV. Any format the browser's `<video>` element can decode.
- The video element is hidden; frames are read by drawing to an offscreen canvas via `ctx.drawImage(videoElement, ...)`.
- Maximum file size: 2 GB (browser memory-dependent; no enforced cap, but a warning displays above 500 MB).
- On load, display: filename, resolution, duration, frame count (estimated from `video.duration × detectedFPS`), and file size.
- FPS detection: read from `video.getVideoPlaybackQuality()` or estimate via frame-stepping (step 0.1s intervals and count unique frames across 1 second sample).

### Frame Stepping

Frames are extracted by setting `video.currentTime` and listening for the `seeked` event. Step size: `1 / fps` seconds. Each seek is awaited before processing:

```
seekToTime(t) → await 'seeked' event → drawImage to offscreen canvas → getImageData → process
```

### Scrub Bar

A range input (`<input type="range">`) below the video preview. Range: 0 to `frameCount - 1`. Dragging it seeks the video and immediately re-renders the vector preview for that frame using current settings. Displays `frame NNN / TTT` and timecode `HH:MM:SS.mmm`.

### Algorithm: Sparse LK (Lucas-Kanade + Shi-Tomasi)

This is the **default algorithm** and primary recommendation for talking-head content.

**Concept:** Detect strong corner features in the previous frame (Shi-Tomasi). Track those features into the current frame (Lucas-Kanade pyramidal). Draw a line from each feature's previous position to its new position.

**Implementation in the browser using `@techstark/opencv.js` (OpenCV.js WebAssembly build):**

```typescript
// Step 1: Detect corners in previous grayscale frame
const corners = new cv.Mat();
cv.goodFeaturesToTrack(
  prevGray,          // source grayscale Mat
  corners,           // output corners
  maxFeatures,       // max corners (default: 300)
  0.01,              // quality level
  10,                // min distance between corners
  new cv.Mat(),      // mask (none)
  3,                 // block size
  false,             // useHarrisDetector
  0.04               // k (Harris param, ignored)
);

// Step 2: Track corners into current frame with LK pyramidal
const nextPts  = new cv.Mat();
const status   = new cv.Mat();
const err      = new cv.Mat();
cv.calcOpticalFlowPyrLK(
  prevGray, currGray,
  corners, nextPts,
  status, err,
  new cv.Size(15, 15),   // winSize
  3,                      // maxLevel (pyramid levels)
  new cv.TermCriteria(
    cv.TermCriteria_EPS | cv.TermCriteria_COUNT, 10, 0.03
  )
);

// Step 3: Filter to only successfully tracked points (status == 1)
// Step 4: Draw lines from corners[i] → nextPts[i] on black canvas
```

**Vector rendering per tracked point:**

```
origin   = corners[i]          // (x0, y0) in source resolution
endpoint = nextPts[i]          // (x1, y1) in source resolution
dx = x1 - x0
dy = y1 - y0
magnitude = sqrt(dx² + dy²)

if magnitude < minMagnitude: skip

// Scale origin to output canvas
ox = x0 * (outW / srcW)
oy = y0 * (outH / srcH)

// Scale displacement by lengthScale, cap at maxLinePx
angle = atan2(dy, dx)
lineLen = clamp(magnitude * lengthScale, 1, maxLinePx)
ex = ox + cos(angle) * lineLen
ey = oy + sin(angle) * lineLen

// Draw line on black canvas
if colourByAngle:
  colour = hsvToRgb(angle_degrees / 360, 1.0, 1.0)
else:
  colour = white (255, 255, 255)

ctx.strokeStyle = colour
ctx.lineWidth   = lineThickness
ctx.beginPath(); ctx.moveTo(ox, oy); ctx.lineTo(ex, ey); ctx.stroke()
```

**Re-seeding:** Every `reseedInterval` frames (default: 30), re-detect corners in the current frame from scratch to capture newly entering features and drop lost tracks.

### Algorithm: Flow Arrows (Dense Farnebäck + Grid Sampling)

Inspired by flowvid's `flow_arrows` preset. Computes a dense optical flow field and samples it on a regular grid.

**Implementation using OpenCV.js:**

```typescript
const flow = new cv.Mat();
cv.calcOpticalFlowFarneback(
  prevGray, currGray,
  flow,
  0.5,   // pyrScale
  3,     // levels
  15,    // winSize
  3,     // iterations
  5,     // polyN
  1.2,   // polySigma
  0      // flags
);
// flow: CV_32FC2 Mat of shape (H, W, 2)
// flow.floatAt(y, x*2)   = dx
// flow.floatAt(y, x*2+1) = dy
```

**Grid sampling and rendering:**

```
for y in range(gridStep/2, srcH, gridStep):
  for x in range(gridStep/2, srcW, gridStep):
    dx = flow.floatAt(y, x*2)
    dy = flow.floatAt(y, x*2 + 1)
    magnitude = sqrt(dx² + dy²)
    if magnitude < minMagnitude: continue
    [render line as per Sparse LK rendering above]
```

### Output Canvas Sizing

Target output resolution is `outputResMegapixels × 1,000,000` pixels, aspect-ratio-preserving:

```typescript
const targetPixels = outputResMegapixels * 1_000_000;  // default: 500_000
const scale = Math.sqrt(targetPixels / (srcW * srcH));
const outW  = Math.round(srcW * scale / 2) * 2;   // round to even
const outH  = Math.round(srcH * scale / 2) * 2;
```

The output canvas is always pure black (`#000000`) before each frame's vectors are drawn.

### Frame Output and ZIP Packaging

Each processed frame is exported as PNG from the output canvas:

```typescript
const dataURL = outputCanvas.toDataURL('image/png');
// stored in memory as Blob, then added to ZIP
```

Frames are named `frame_000001.png` through `frame_NNNNNN.png` (6-digit zero-padded, 1-indexed).

ZIP packaging uses the `jszip` library:

```typescript
const zip = new JSZip();
frames.forEach((blob, i) => {
  zip.file(`frame_${String(i + 1).padStart(6, '0')}.png`, blob);
});
const zipBlob = await zip.generateAsync({ type: 'blob' });
saveAs(zipBlob, `vidvec_${baseName}_${timestamp}.zip`);
```

`FileSaver.js` handles the download trigger.

### Settings Panel — All Parameters

| Parameter | UI Element | Type | Default | Range / Options |
|---|---|---|---|---|
| `algorithm` | Radio group | `'sparse' \| 'flowArrows'` | `'sparse'` | — |
| `maxFeatures` | Number input | `number` | `300` | 10 – 2000 |
| `minMagnitude` | Number input | `number` | `0.5` | 0.0 – 10.0, step 0.1 |
| `lengthScale` | Number input | `number` | `3.0` | 0.5 – 20.0, step 0.5 |
| `maxLinePx` | Number input | `number` | `40` | 5 – 200 |
| `gridStep` | Number input | `number` | `16` | 4 – 64 (Flow Arrows only) |
| `lineThickness` | Number input | `number` | `1` | 1 – 5 |
| `colourByAngle` | Toggle switch | `boolean` | `false` | — |
| `outputResMegapixels` | Number input | `number` | `0.5` | 0.1 – 2.0, step 0.1 |
| `reseedInterval` | Number input | `number` | `30` | 5 – 120 (Sparse only) |

Settings state is persisted to `localStorage` under the key `vidvec_settings` and restored on app load.

`gridStep` and `reseedInterval` inputs are disabled and visually dimmed when the non-applicable algorithm is selected.

### Progress Display

During processing, show:
- A progress bar (filled left-to-right) as a percentage of frames completed
- Text: `Processing frame NNN / TTT • ~MM:SS remaining`
- Estimated time remaining: rolling average of per-frame processing time × remaining frames
- A CANCEL button that aborts the loop and leaves partial results downloadable

### Error States

| Condition | Behaviour |
|---|---|
| File not a supported video format | Red banner: "Unsupported file type. Please load an MP4, WebM, or MOV file." |
| Video has zero frames detected | Red banner: "Could not detect frames in this video. Try a different file." |
| File > 500 MB | Yellow warning banner: "Large file detected. Processing may be slow or run out of memory." — does not block |
| OpenCV.js fails to load | Red banner: "Motion analysis engine failed to load. Check your internet connection and reload." |
| Canvas memory error during processing | Red banner: "Out of memory at frame NNN. Download partial results." — DOWNLOAD button activates |
| User cancels processing | Blue info banner: "Processing cancelled at frame NNN. Download partial results." |

---

## Technical Implementation

### Stack

```
Vite 5.x          — build tool and dev server
React 18.x        — UI framework
TypeScript 5.x    — language
@techstark/opencv.js — OpenCV WebAssembly (optical flow)
jszip             — client-side ZIP creation
file-saver        — browser download trigger
Tailwind CSS 3.x  — utility-first styling
```

No backend. No state management library (React `useState` + `useReducer` is sufficient). No router (single-page app).

### Project Structure

```
vidvec/
├── index.html
├── vite.config.ts
├── tsconfig.json
├── tailwind.config.ts
├── package.json
├── public/
│   └── opencv.js            # OpenCV WASM build (copied from @techstark/opencv.js)
└── src/
    ├── main.tsx             # React root mount
    ├── App.tsx              # Root layout, state orchestration
    ├── types.ts             # All TypeScript types and interfaces
    ├── components/
    │   ├── DropZone.tsx     # File drop + input
    │   ├── VideoPreview.tsx # Video element + scrub bar
    │   ├── VectorPreview.tsx# Output canvas live preview
    │   ├── SettingsPanel.tsx# All parameter controls
    │   └── ProgressBar.tsx  # Processing progress UI
    ├── hooks/
    │   ├── useVideoLoader.ts   # Video file loading, frame count, FPS detection
    │   ├── useFrameStepper.ts  # Frame-by-frame seeking via currentTime + seeked
    │   └── useOpenCV.ts        # OpenCV.js WASM load state
    ├── lib/
    │   ├── opencvLoader.ts     # Dynamic script injection + ready promise
    │   ├── sparseFlow.ts       # LK + Shi-Tomasi implementation
    │   ├── denseFlow.ts        # Farnebäck + grid sampling implementation
    │   ├── renderer.ts         # Black canvas + line drawing
    │   ├── outputSizing.ts     # Aspect-ratio-preserving resolution calc
    │   └── zipExporter.ts      # JSZip + FileSaver integration
    └── styles/
        └── index.css
```

### TypeScript Types (`src/types.ts`)

```typescript
export type Algorithm = 'sparse' | 'flowArrows';

export interface Settings {
  algorithm:          Algorithm;
  maxFeatures:        number;   // default 300
  minMagnitude:       number;   // default 0.5
  lengthScale:        number;   // default 3.0
  maxLinePx:          number;   // default 40
  gridStep:           number;   // default 16
  lineThickness:      number;   // default 1
  colourByAngle:      boolean;  // default false
  outputResMegapixels: number;  // default 0.5
  reseedInterval:     number;   // default 30
}

export interface VideoInfo {
  file:       File;
  url:        string;         // object URL
  width:      number;
  height:     number;
  fps:        number;
  frameCount: number;
  duration:   number;         // seconds
  fileSize:   number;         // bytes
}

export interface ProcessingState {
  status:       'idle' | 'processing' | 'complete' | 'cancelled' | 'error';
  currentFrame: number;
  totalFrames:  number;
  startTime:    number | null;  // Date.now() at start
  error:        string | null;
}

export interface FrameBlob {
  index:  number;   // 0-based frame index
  blob:   Blob;     // PNG blob
}
```

### OpenCV Loading (`src/lib/opencvLoader.ts`)

OpenCV.js is loaded once asynchronously as a script tag pointing to `/opencv.js`. A module-level promise resolves when `window.cv` is available and `cv.getBuildInformation()` succeeds:

```typescript
let opencvReady: Promise<void> | null = null;

export function loadOpenCV(): Promise<void> {
  if (opencvReady) return opencvReady;
  opencvReady = new Promise((resolve, reject) => {
    const script = document.createElement('script');
    script.src = '/opencv.js';
    script.async = true;
    script.onload = () => {
      const cv = (window as any).cv;
      if (cv && cv.getBuildInformation) {
        cv['onRuntimeInitialized'] = () => resolve();
        // If already initialised:
        if (cv.Mat) resolve();
      } else {
        reject(new Error('OpenCV.js did not load correctly'));
      }
    };
    script.onerror = () => reject(new Error('Failed to fetch opencv.js'));
    document.head.appendChild(script);
  });
  return opencvReady;
}
```

`useOpenCV` hook wraps this in a `useEffect` and exposes `{ loaded: boolean, error: string | null }`.

### Frame Stepper (`src/hooks/useFrameStepper.ts`)

Exposes `seekToFrame(index: number): Promise<void>`. Internally:

```typescript
async function seekToFrame(index: number): Promise<void> {
  const t = index / videoInfo.fps;
  video.currentTime = t;
  await new Promise<void>((resolve) => {
    const onSeeked = () => {
      video.removeEventListener('seeked', onSeeked);
      resolve();
    };
    video.addEventListener('seeked', onSeeked);
  });
}
```

After seeking, the caller reads the frame by drawing the video element to an offscreen canvas and calling `getImageData`.

### Sparse Flow Implementation (`src/lib/sparseFlow.ts`)

Full implementation with proper OpenCV.js Mat lifecycle management (all Mats must be `.delete()`d after use to avoid WASM heap leaks):

```typescript
export interface SparseFlowState {
  prevGray: any;  // cv.Mat
  prevPts:  any;  // cv.Mat of detected corners
  framesSinceReseed: number;
}

export function initSparseFlow(): SparseFlowState {
  return { prevGray: null, prevPts: null, framesSinceReseed: 0 };
}

export function processSparseFrame(
  cv: any,
  state: SparseFlowState,
  currImageData: ImageData,
  settings: Settings,
  srcW: number,
  srcH: number
): { vectors: Vector2D[], nextState: SparseFlowState } {

  const currMat  = cv.matFromImageData(currImageData);
  const currGray = new cv.Mat();
  cv.cvtColor(currMat, currGray, cv.COLOR_RGBA2GRAY);
  currMat.delete();

  let vectors: Vector2D[] = [];
  let nextPts: any = null;

  const needsReseed = !state.prevGray
    || !state.prevPts
    || state.prevPts.rows === 0
    || state.framesSinceReseed >= settings.reseedInterval;

  if (needsReseed) {
    // Detect fresh corners
    if (state.prevPts) state.prevPts.delete();
    const corners = new cv.Mat();
    cv.goodFeaturesToTrack(
      currGray, corners,
      settings.maxFeatures, 0.01, 10,
      new cv.Mat(), 3, false, 0.04
    );
    // First frame or reseed: no vectors this frame, just store state
    if (state.prevGray) state.prevGray.delete();
    return {
      vectors: [],
      nextState: {
        prevGray: currGray,
        prevPts: corners,
        framesSinceReseed: 0
      }
    };
  }

  // Track previous points into current frame
  nextPts       = new cv.Mat();
  const status  = new cv.Mat();
  const err     = new cv.Mat();

  cv.calcOpticalFlowPyrLK(
    state.prevGray, currGray,
    state.prevPts, nextPts,
    status, err,
    new cv.Size(15, 15), 3,
    new cv.TermCriteria(
      cv.TermCriteria_EPS | cv.TermCriteria_COUNT, 10, 0.03
    )
  );

  // Extract successfully tracked vectors
  for (let i = 0; i < status.rows; i++) {
    if (status.data[i] !== 1) continue;
    const x0 = state.prevPts.data32F[i * 2];
    const y0 = state.prevPts.data32F[i * 2 + 1];
    const x1 = nextPts.data32F[i * 2];
    const y1 = nextPts.data32F[i * 2 + 1];
    vectors.push({ x: x0, y: y0, dx: x1 - x0, dy: y1 - y0 });
  }

  status.delete(); err.delete();
  state.prevGray.delete();

  return {
    vectors,
    nextState: {
      prevGray: currGray,
      prevPts: nextPts,
      framesSinceReseed: state.framesSinceReseed + 1
    }
  };
}

export interface Vector2D {
  x: number; y: number;    // origin in source pixels
  dx: number; dy: number;  // displacement in source pixels
}
```

### Dense Flow Implementation (`src/lib/denseFlow.ts`)

```typescript
export function processDenseFrame(
  cv: any,
  prevImageData: ImageData,
  currImageData: ImageData,
  settings: Settings,
  srcW: number,
  srcH: number
): Vector2D[] {

  const prevMat  = cv.matFromImageData(prevImageData);
  const currMat  = cv.matFromImageData(currImageData);
  const prevGray = new cv.Mat();
  const currGray = new cv.Mat();

  cv.cvtColor(prevMat, prevGray, cv.COLOR_RGBA2GRAY);
  cv.cvtColor(currMat, currGray, cv.COLOR_RGBA2GRAY);
  prevMat.delete(); currMat.delete();

  const flow = new cv.Mat();
  cv.calcOpticalFlowFarneback(
    prevGray, currGray, flow,
    0.5, 3, 15, 3, 5, 1.2, 0
  );
  prevGray.delete(); currGray.delete();

  const vectors: Vector2D[] = [];
  const step = settings.gridStep;

  for (let y = Math.floor(step / 2); y < srcH; y += step) {
    for (let x = Math.floor(step / 2); x < srcW; x += step) {
      const idx = (y * srcW + x) * 2;
      const dx  = flow.data32F[idx];
      const dy  = flow.data32F[idx + 1];
      const mag = Math.sqrt(dx * dx + dy * dy);
      if (mag >= settings.minMagnitude) {
        vectors.push({ x, y, dx, dy });
      }
    }
  }

  flow.delete();
  return vectors;
}
```

### Renderer (`src/lib/renderer.ts`)

Renders a `Vector2D[]` onto a black canvas at output resolution:

```typescript
export function renderVectors(
  ctx: CanvasRenderingContext2D,
  vectors: Vector2D[],
  settings: Settings,
  srcW: number, srcH: number,
  outW: number, outH: number
): void {
  // Clear to black
  ctx.fillStyle = '#000000';
  ctx.fillRect(0, 0, outW, outH);

  const scaleX = outW / srcW;
  const scaleY = outH / srcH;

  ctx.lineWidth = settings.lineThickness;

  for (const v of vectors) {
    const { x, y, dx, dy } = v;
    const mag = Math.sqrt(dx * dx + dy * dy);
    if (mag < settings.minMagnitude) continue;

    const angle   = Math.atan2(dy, dx);
    const lineLen = Math.min(mag * settings.lengthScale, settings.maxLinePx);

    const ox = x  * scaleX;
    const oy = y  * scaleY;
    const ex = ox + Math.cos(angle) * lineLen;
    const ey = oy + Math.sin(angle) * lineLen;

    if (settings.colourByAngle) {
      // Map angle [-π, π] → hue [0°, 360°]
      const hue = ((angle + Math.PI) / (2 * Math.PI)) * 360;
      ctx.strokeStyle = `hsl(${hue}, 100%, 60%)`;
    } else {
      ctx.strokeStyle = '#ffffff';
    }

    ctx.beginPath();
    ctx.moveTo(ox, oy);
    ctx.lineTo(ex, ey);
    ctx.stroke();
  }
}
```

### Processing Loop (`src/App.tsx` — processVideo function)

The main processing loop runs as an async function invoked on PROCESS click. It uses a `cancelRef` (`useRef<boolean>`) for cancellation:

```typescript
async function processVideo(): Promise<void> {
  cancelRef.current = false;
  setProcessingState({ status: 'processing', currentFrame: 0,
    totalFrames: videoInfo.frameCount, startTime: Date.now(), error: null });

  const frames: FrameBlob[] = [];
  const { outW, outH } = computeOutputSize(
    videoInfo.width, videoInfo.height, settings.outputResMegapixels
  );

  // Set up output canvas (offscreen for performance)
  const offscreen = document.createElement('canvas');
  offscreen.width  = outW;
  offscreen.height = outH;
  const ctx = offscreen.getContext('2d')!;

  // Source canvas for reading video frames
  const srcCanvas  = document.createElement('canvas');
  srcCanvas.width  = videoInfo.width;
  srcCanvas.height = videoInfo.height;
  const srcCtx = srcCanvas.getContext('2d')!;

  let sparseState = initSparseFlow();
  let prevImageData: ImageData | null = null;

  for (let i = 0; i < videoInfo.frameCount; i++) {
    if (cancelRef.current) {
      setProcessingState(s => ({ ...s, status: 'cancelled' }));
      break;
    }

    await seekToFrame(i);
    srcCtx.drawImage(videoRef.current!, 0, 0);
    const imageData = srcCtx.getImageData(0, 0, videoInfo.width, videoInfo.height);

    let vectors: Vector2D[] = [];

    if (settings.algorithm === 'sparse') {
      const result = processSparseFrame(
        cv, sparseState, imageData, settings, videoInfo.width, videoInfo.height
      );
      vectors     = result.vectors;
      sparseState = result.nextState;
    } else {
      if (prevImageData) {
        vectors = processDenseFrame(
          cv, prevImageData, imageData, settings, videoInfo.width, videoInfo.height
        );
      }
      prevImageData = imageData;
    }

    renderVectors(ctx, vectors, settings, videoInfo.width, videoInfo.height, outW, outH);

    // Update live preview
    livePreviewCtx.drawImage(offscreen, 0, 0,
      livePreviewCanvas.width, livePreviewCanvas.height);

    // Capture PNG blob
    const blob = await new Promise<Blob>((res) =>
      offscreen.toBlob(b => res(b!), 'image/png')
    );
    frames.push({ index: i, blob });

    setProcessingState(s => ({ ...s, currentFrame: i + 1 }));
  }

  // Clean up OpenCV Mats in sparse state
  if (sparseState.prevGray) sparseState.prevGray.delete();
  if (sparseState.prevPts)  sparseState.prevPts.delete();

  setFrameBlobs(frames);
  setProcessingState(s => ({
    ...s,
    status: cancelRef.current ? 'cancelled' : 'complete'
  }));
}
```

### ZIP Export (`src/lib/zipExporter.ts`)

```typescript
import JSZip from 'jszip';
import { saveAs } from 'file-saver';

export async function exportFramesAsZip(
  frames: FrameBlob[],
  baseName: string
): Promise<void> {
  const zip = new JSZip();
  for (const { index, blob } of frames) {
    const name = `frame_${String(index + 1).padStart(6, '0')}.png`;
    zip.file(name, blob);
  }
  const zipBlob = await zip.generateAsync({
    type: 'blob',
    compression: 'STORE'   // PNG is already compressed; STORE is fastest
  });
  const timestamp = new Date().toISOString().replace(/[:.]/g, '-').slice(0, 19);
  saveAs(zipBlob, `vidvec_${baseName}_${timestamp}.zip`);
}
```

### `vite.config.ts`

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  optimizeDeps: {
    exclude: ['@techstark/opencv.js']  // WASM — do not pre-bundle
  },
  server: {
    headers: {
      // Required for SharedArrayBuffer / WASM threading (if used)
      'Cross-Origin-Opener-Policy':   'same-origin',
      'Cross-Origin-Embedder-Policy': 'require-corp',
    }
  }
});
```

### `package.json` dependencies

```json
{
  "dependencies": {
    "react":                "^18.3.1",
    "react-dom":            "^18.3.1",
    "@techstark/opencv.js": "^4.10.0",
    "jszip":                "^3.10.1",
    "file-saver":           "^2.0.5"
  },
  "devDependencies": {
    "@vitejs/plugin-react":  "^4.3.1",
    "@types/react":          "^18.3.3",
    "@types/react-dom":      "^18.3.0",
    "@types/file-saver":     "^2.0.7",
    "typescript":            "^5.5.3",
    "vite":                  "^5.3.4",
    "tailwindcss":           "^3.4.4",
    "autoprefixer":          "^10.4.19",
    "postcss":               "^8.4.39"
  }
}
```

---

## Style Guide

**Colour palette:**

| Token | Value | Usage |
|---|---|---|
| Background | `#0a0a0a` | App background |
| Surface | `#141414` | Panel backgrounds |
| Border | `#2a2a2a` | Dividers, input borders |
| Accent | `#00ff88` | Active states, progress bar fill |
| Text primary | `#f0f0f0` | Labels, values |
| Text muted | `#666666` | Secondary labels |
| Error | `#ff4444` | Error banners |
| Warning | `#ffaa00` | Warning banners |
| Info | `#4488ff` | Info/cancel banners |

**Typography:** System monospace stack (`ui-monospace, 'Cascadia Code', 'Fira Mono', monospace`) throughout. This reinforces the technical, data-visualisation nature of the app.

**Canvas borders:** 1px solid `#2a2a2a`. Vector preview canvas has a subtle inner glow: `box-shadow: inset 0 0 20px rgba(0,255,136,0.05)`.

**Animations:** Progress bar fill transitions with `transition: width 80ms linear`. No other animations — the app is a tool, not a showcase.

---

## Performance Goals

- Frame processing time per frame: < 200ms on a mid-range laptop (2022+) for Sparse mode at 0.5MP output
- Dense Farnebäck mode: < 500ms per frame at 0.5MP
- ZIP generation for 300 frames: < 10 seconds
- OpenCV.js WASM load time: < 3 seconds on broadband (file is ~8MB)
- No UI freezing during processing — yield to event loop every frame via `await new Promise(r => setTimeout(r, 0))` if needed

---

## Accessibility Requirements

- All interactive controls have visible focus rings (2px solid accent colour)
- Settings inputs have associated `<label>` elements with `htmlFor`
- Progress bar uses `role="progressbar"` with `aria-valuenow`, `aria-valuemin`, `aria-valuemax`
- Drop zone uses `role="button"` with `aria-label="Drop video file or click to browse"`
- Error and status banners use `role="alert"` with `aria-live="assertive"` (errors) or `"polite"` (status)
- Colour-by-angle mode note: when enabled, direction is encoded by both angle and colour; the geometric encoding (line angle) remains functional without colour perception

---

## Testing Scenarios

1. **Load a 16:9 MP4 (1080p, 30fps, 10s)** → confirm frame count ~300, output canvas ~848×477
2. **Load a 9:16 portrait MP4 (1080×1920)** → confirm output canvas ~265×471
3. **Load a 4:3 video** → confirm aspect ratio preserved
4. **Scrub to frame 50** → vector preview updates, no processing triggered
5. **Change algorithm, tune settings** → vector preview re-renders within 200ms
6. **Process full 300-frame video, Sparse mode** → ZIP downloads, contains 300 PNGs, first frame black (no prev), frames 2–300 have vectors
7. **Process with Flow Arrows** → all frames 2–300 have grid vectors (frame 1 is black)
8. **Cancel mid-process at frame 150** → DOWNLOAD activates, ZIP contains 150 frames
9. **Colour by angle on** → lines are coloured, not white; hue varies with direction
10. **File > 500 MB** → yellow warning shown, processing still works
11. **Drag and drop video file** → equivalent to file picker
12. **Reload page** → settings restored from localStorage

---

## Extended Features (Optional)

These are not in scope for v1 but are natural extensions:

- **Region of interest masking:** Let the user draw rectangles over the video preview to restrict vector extraction to specific panel regions (e.g. left speaker only). Mask applied before corner detection or flow sampling.
- **Frame export as video:** Instead of PNG sequence ZIP, use the MediaRecorder API to encode the vector canvas stream directly to WebM.
- **Temporal accumulation:** Blend previous N frames' vectors (with decay) onto the current canvas to show motion trails.
- **Side-by-side preview mode:** Split the preview area to show source frame and vector frame simultaneously.
- **ffglitch JSON import:** Accept a JSON file from ffglitch's `ffedit` tool as an alternative to computed optical flow, using the codec's embedded motion vectors directly.

---

## Compositing Reference

The output PNG sequence is designed for frame-accurate overlay in external tools. Black pixels act as transparency when composited with **Screen** or **Add** blend modes.

**FFmpeg (screen blend over source video):**
```bash
ffmpeg -i source.mp4 \
  -framerate 30 -i frames/frame_%06d.png \
  -filter_complex \
    "[0:v]setsar=1[base]; \
     [1:v]setsar=1[vectors]; \
     [base][vectors]blend=all_mode=screen:all_opacity=0.85[out]" \
  -map "[out]" -c:v libx264 -crf 18 \
  output_composited.mp4
```

**DaVinci Resolve / After Effects:** Import PNG sequence as image sequence layer, set blend mode to Screen or Add, align frame 1 to video frame 1.

---

*TINS Specification — VIDVEC v1.0*
*Grounding document: motion-vector-expression-grounding.md*
*Algorithm references: flowvid (PyPI), OpenCV Lucas-Kanade + Shi-Tomasi sparse optical flow*
