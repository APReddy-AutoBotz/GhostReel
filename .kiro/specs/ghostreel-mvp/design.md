# Design Document: GhostReel MVP

## Overview

GhostReel is a browser-based sound design application that revives Daphne Oram's Oramics concept for independent filmmakers. The application allows users to draw sound on parallel film strips locked to picture, place synthesized cues at precise frames, save sound identities for reuse across shots, and export synchronized audio.

### Core User Workflow

1. Load a reference video clip (up to 60 seconds at 24 fps)
2. Draw or modify waveform, pitch, and intensity on parallel strips
3. Place cue markers (start, hit, end) on picture frames
4. Hear the synthesized sound synchronized to picture
5. Save sound identities for reuse across scenes
6. Revise cue timing while preserving the hit anchor
7. Export synchronized stereo WAV audio
8. Verify sync by loading the exported WAV back into the app

### Design Principles

- **Direct Manipulation:** Drawing changes immediately affect synthesized sound
- **Frame Precision:** All timing expressed in frames with sample-accurate export
- **Visual Lock:** Strips and picture timelines scroll together under a fixed playhead
- **Predictable Revision:** Moving start/end refits curves while preserving hit position
- **No Hidden State:** All sound parameters visible on strips and saveable

### Technology Stack

- **React 18+** with TypeScript for UI components and state management
- **Vite** for build tooling and development server
- **Web Audio API** for synthesis, mixing, and offline rendering
- **HTML5 Video Element** for picture playback
- **Canvas 2D** for drawable strips
- **No server-side processing:** all work happens in the browser

## Architecture

### High-Level Component Structure

```
┌─────────────────────────────────────────────────────────┐
│                    Application Shell                     │
│  - Project state management                              │
│  - File I/O (open video, save/load project, export WAV) │
│  - Global controls (play/stop, volume, undo/redo)       │
└────────────┬────────────────────────────────────────────┘
             │
    ┌────────┴────────┐
    │                 │
┌───▼──────────┐  ┌──▼─────────────────────────────────┐
│ Video        │  │ Audio Engine                       │
│ Component    │  │ - Synthesis                        │
│              │  │ - Mixing                           │
│ - Playback   │  │ - Offline rendering                │
│ - Scrubbing  │  │ - Real-time playback               │
│ - Timeline   │  │                                    │
└───┬──────────┘  └──┬─────────────────────────────────┘
    │                │
    │                │
┌───▼────────────────▼─────────────────────────────────┐
│              Strip Editor Canvas                     │
│  - Three strips per layer (waveform, pitch, intensity)│
│  - Marker placement (start, hit, end)                │
│  - Drawing interaction                               │
│  - Visual timeline sync                              │
└──────────────────────────────────────────────────────┘
    │
┌───▼──────────────────────────────────────────────────┐
│           Identity & Snapshot Manager                │
│  - Identity shelf                                    │
│  - Snapshot storage and comparison                   │
│  - Recipe templates                                  │
└──────────────────────────────────────────────────────┘
```

### Component Responsibilities

#### Application Shell
- Maintains the global project state
- Coordinates playback between video and audio
- Handles file operations (video open, project save/load, WAV export)
- Manages undo/redo stack
- Provides transport controls and volume

#### Video Component
- Wraps HTML5 video element
- Exposes current playback time and frame position
- Responds to play/pause/seek commands
- Emits timeupdate events for strip synchronization
- Reports video duration and dimensions

#### Audio Engine
- **Synthesizer:** Generates audio from waveform, pitch, and intensity curves
- **Mixer:** Combines multiple layers with level controls
- **Real-time Renderer:** Connects to Web Audio destination for playback
- **Offline Renderer:** Produces WAV export using OfflineAudioContext
- **Safety:** Enforces gain limits, applies fade envelopes, prevents clicks

#### Strip Editor Canvas
- Renders three strips per layer (waveform, pitch, intensity)
- Handles mouse/touch drawing interactions
- Displays cue markers (start, hit, end) on strips and timeline
- Scrolls strips in sync with video playback
- Supports selection, editing, and undo of drawn points

#### Identity & Snapshot Manager
- Maintains identity shelf (saved sound recipes)
- Creates, updates, and deletes identities
- Applies identities to cues (copies data, fits curves to cue duration)
- Saves and loads cue snapshots for A/B comparison
- Provides three starter recipes (tension pulse, energy build, stylized presence)

### State Management Strategy

Use React Context + useReducer for global state, structured as:

```typescript
interface AppState {
  project: Project;
  playback: PlaybackState;
  ui: UIState;
  undoStack: ProjectSnapshot[];
  redoStack: ProjectSnapshot[];
}

interface PlaybackState {
  isPlaying: boolean;
  currentFrame: number;
  volume: number; // 0.0 to 1.0
  mode: 'synthesis' | 'verifyExport';
  exportedAudioBuffer?: AudioBuffer;
}

interface UIState {
  selectedLayer: number; // 0, 1, 2
  selectedStrip: 'waveform' | 'pitch' | 'intensity' | null;
  comparisonActive: boolean;
  tourDismissed: boolean;
  creditNoteDismissed: boolean;
}
```

Local component state for transient interactions (mouse position, hover state, drag in progress).

## Components and Interfaces

### Core Data Types

```typescript
// Project is the root saveable structure
interface Project {
  version: '1.0';
  identityShelf: Identity[];
  scenes: Scene[]; // max 2
  activeSceneIndex: number;
}

// Scene references video and contains cues
interface Scene {
  videoSource: VideoSource;
  frameRate: 24; // only 24 fps supported in MVP
  durationFrames: number;
  layers: Layer[]; // max 3
}

// VideoSource distinguishes bundled from user files
type VideoSource = 
  | { type: 'bundled'; name: string } 
  | { type: 'user'; fileName: string };

// Layer groups one cue with its settings
interface Layer {
  cue: Cue | null;
  level: number; // 0.0 to 1.0
  muted: boolean;
  snapshot: Cue | null; // for A/B comparison
}

// Cue is a placed sound performance
interface Cue {
  identityId: string; // references Identity
  startFrame: number;
  hitFrame: number;
  endFrame: number;
  waveform: Curve; // single-cycle waveform shape
  pitch: Curve; // frequency over time
  intensity: Curve; // amplitude envelope over time
}

// Identity is a reusable sound recipe (no timing)
interface Identity {
  id: string; // UUID
  name: string;
  waveform: Curve;
  defaultPitch: Curve; // normalized 0-1 range
  defaultIntensity: Curve; // normalized 0-1 range
  recipeSettings: RecipeSettings;
}

// RecipeSettings capture synthesis parameters
interface RecipeSettings {
  baseFrequency: number; // Hz, default pitch anchor
  frequencyRange: [number, number]; // Hz, min/max for pitch curve
  gainRange: [number, number]; // linear multiplier for intensity curve
}

// Curve is a drawable, editable path
interface Curve {
  points: Point[];
  interpolation: 'linear' | 'cubic'; // linear for MVP
}

interface Point {
  x: number; // normalized 0-1 (relative to curve domain)
  y: number; // normalized 0-1 (relative to curve range)
}
```

### Curve Anchoring Model

Cues have three anchor frames: start, hit, and end. Pitch and intensity curves are divided into two segments:

1. **Pre-hit segment:** start frame to hit frame
2. **Post-hit segment:** hit frame to end frame

When markers move:

- **Move start marker:** Refit pre-hit segment to new duration, post-hit segment unchanged
- **Move end marker:** Refit post-hit segment to new duration, pre-hit segment unchanged
- **Move hit marker:** Move entire cue (all three markers), preserving both segment durations
- **Waveform:** Always copied unchanged from identity; no anchoring

**Implementation:** Store curves as normalized (0-1 domain), anchor at 0.0 (start), 0.5 (hit), 1.0 (end). When refitting, map only the affected half of the curve to the new frame range.

```typescript
interface CurveSegments {
  preHit: Point[]; // x in [0, 0.5]
  postHit: Point[]; // x in [0.5, 1.0]
}

function splitCurveAtHit(curve: Curve): CurveSegments {
  const preHit = curve.points.filter(p => p.x <= 0.5);
  const postHit = curve.points.filter(p => p.x >= 0.5);
  return { preHit, postHit };
}

function refitPreHit(
  original: CurveSegments,
  newStartFrame: number,
  hitFrame: number,
  endFrame: number
): Curve {
  // Refit pre-hit segment to [newStartFrame, hitFrame]
  // Keep post-hit segment unchanged on [hitFrame, endFrame]
  // Return merged curve
}
```

### Audio Engine Interface

```typescript
interface AudioEngine {
  // Real-time synthesis
  startPlayback(scene: Scene, startFrame: number): void;
  stopPlayback(): void;
  setVolume(volume: number): void;
  
  // Offline rendering
  renderToBuffer(scene: Scene): Promise<AudioBuffer>;
  
  // Cleanup
  dispose(): void;
}

class WebAudioEngine implements AudioEngine {
  private context: AudioContext;
  private offlineContext?: OfflineAudioContext;
  private sources: AudioBufferSourceNode[];
  private gainNode: GainNode;
  private limiter: DynamicsCompressorNode;
  
  // ... implementation
}
```

### Synthesizer Design

**Waveform Interpretation:**

- User draws a single-cycle waveform on a canvas (e.g., 512 pixels wide)
- Sample this curve at a fixed resolution (e.g., 2048 samples) to create a PeriodicWave
- Use `AudioContext.createPeriodicWave()` with real/imaginary coefficients derived from the drawn shape
- This creates a repeating waveform with controlled harmonic content

**Alternative (simpler for MVP):** Store the drawn waveform as a single-cycle AudioBuffer and use looped AudioBufferSourceNode.

```typescript
function createWaveformBuffer(
  curve: Curve, 
  sampleRate: number
): AudioBuffer {
  const cycleLength = 2048; // samples per cycle
  const buffer = new AudioBuffer({
    length: cycleLength,
    sampleRate: sampleRate,
    numberOfChannels: 1
  });
  
  const channel = buffer.getChannelData(0);
  for (let i = 0; i < cycleLength; i++) {
    const x = i / cycleLength; // 0 to 1
    channel[i] = evaluateCurve(curve, x);
  }
  
  return buffer;
}
```

**Pitch Control:**

- Pitch curve maps to `AudioBufferSourceNode.playbackRate`
- Curve's y-axis maps to frequency range defined in RecipeSettings
- Sample pitch curve at the cue's frame rate, apply values via `setValueAtTime()`

```typescript
function applyPitchCurve(
  source: AudioBufferSourceNode,
  pitchCurve: Curve,
  recipeSettings: RecipeSettings,
  startTime: number,
  durationFrames: number,
  frameRate: number
): void {
  const [minFreq, maxFreq] = recipeSettings.frequencyRange;
  const frameDuration = 1 / frameRate;
  
  for (let frame = 0; frame < durationFrames; frame++) {
    const t = startTime + (frame * frameDuration);
    const x = frame / durationFrames; // 0 to 1
    const y = evaluateCurve(pitchCurve, x); // 0 to 1
    const freq = minFreq + y * (maxFreq - minFreq);
    
    // playbackRate relative to base frequency embedded in waveform
    const baseFreq = recipeSettings.baseFrequency;
    const rate = freq / baseFreq;
    
    source.playbackRate.setValueAtTime(rate, t);
  }
}
```

**Intensity Control:**

- Intensity curve maps to `GainNode.gain`
- Apply fade-in (5ms) and fade-out (20ms) envelopes to prevent clicks
- Sample intensity curve at frame rate

**Click Prevention:**

```typescript
function applyIntensityCurve(
  gainNode: GainNode,
  intensityCurve: Curve,
  recipeSettings: RecipeSettings,
  startTime: number,
  durationFrames: number,
  frameRate: number
): void {
  const [minGain, maxGain] = recipeSettings.gainRange;
  const frameDuration = 1 / frameRate;
  const fadeInTime = 0.005; // 5ms
  const fadeOutTime = 0.020; // 20ms
  
  gainNode.gain.setValueAtTime(0, startTime);
  gainNode.gain.linearRampToValueAtTime(
    evaluateIntensityAt(intensityCurve, 0, minGain, maxGain),
    startTime + fadeInTime
  );
  
  for (let frame = 0; frame < durationFrames; frame++) {
    const t = startTime + (frame * frameDuration);
    const x = frame / durationFrames;
    const y = evaluateCurve(intensityCurve, x);
    const gain = minGain + y * (maxGain - minGain);
    
    gainNode.gain.setValueAtTime(gain, t);
  }
  
  const endTime = startTime + (durationFrames * frameDuration);
  gainNode.gain.setValueAtTime(
    evaluateIntensityAt(intensityCurve, 1.0, minGain, maxGain),
    endTime - fadeOutTime
  );
  gainNode.gain.linearRampToValueAtTime(0, endTime);
}
```

**Mixing:**

```typescript
function mixLayers(scene: Scene, context: BaseAudioContext, startTime: number): void {
  const masterGain = context.createGain();
  const limiter = context.createDynamicsCompressor();
  
  // Configure limiter to enforce -6 dBFS ceiling
  limiter.threshold.setValueAtTime(-6, startTime);
  limiter.knee.setValueAtTime(0, startTime);
  limiter.ratio.setValueAtTime(20, startTime);
  limiter.attack.setValueAtTime(0.001, startTime);
  limiter.release.setValueAtTime(0.01, startTime);
  
  masterGain.connect(limiter);
  limiter.connect(context.destination);
  
  for (const layer of scene.layers) {
    if (layer.cue && !layer.muted) {
      const layerGain = context.createGain();
      layerGain.gain.setValueAtTime(layer.level, startTime);
      
      synthesizeCue(layer.cue, context, startTime, layerGain);
      layerGain.connect(masterGain);
    }
  }
}
```

### Peak Limiting Strategy (addresses REQ-G1.6)

Use a DynamicsCompressorNode configured as a brick-wall limiter:

- **Threshold:** -6 dB
- **Ratio:** 20:1 (effectively infinite)
- **Knee:** 0 dB (hard knee)
- **Attack:** 1 ms
- **Release:** 10 ms

This guarantees exported audio never exceeds -6 dBFS peak while preserving transient detail within the threshold.

**Verification:** After offline rendering, scan the output buffer to confirm peak level:

```typescript
function verifyPeakLevel(buffer: AudioBuffer): number {
  let peak = 0;
  for (let ch = 0; ch < buffer.numberOfChannels; ch++) {
    const samples = buffer.getChannelData(ch);
    for (let i = 0; i < samples.length; i++) {
      peak = Math.max(peak, Math.abs(samples[i]));
    }
  }
  return 20 * Math.log10(peak); // dBFS
}
```

## Data Models

### Project File Format (JSON)

```json
{
  "version": "1.0",
  "identityShelf": [
    {
      "id": "uuid-1",
      "name": "Tension Pulse",
      "waveform": {
        "points": [{"x": 0, "y": 0}, {"x": 0.5, "y": 1}, {"x": 1, "y": 0}],
        "interpolation": "linear"
      },
      "defaultPitch": {
        "points": [{"x": 0, "y": 0.3}, {"x": 0.5, "y": 0.5}, {"x": 1, "y": 0.3}],
        "interpolation": "linear"
      },
      "defaultIntensity": {
        "points": [{"x": 0, "y": 0}, {"x": 0.5, "y": 1}, {"x": 1, "y": 0}],
        "interpolation": "linear"
      },
      "recipeSettings": {
        "baseFrequency": 110,
        "frequencyRange": [80, 440],
        "gainRange": [0, 0.5]
      }
    }
  ],
  "scenes": [
    {
      "videoSource": {
        "type": "bundled",
        "name": "corridor_approach"
      },
      "frameRate": 24,
      "durationFrames": 240,
      "layers": [
        {
          "cue": {
            "identityId": "uuid-1",
            "startFrame": 60,
            "hitFrame": 120,
            "endFrame": 180,
            "waveform": { /* copied from identity */ },
            "pitch": { /* fitted to 60 frames */ },
            "intensity": { /* fitted to 60 frames */ }
          },
          "level": 0.8,
          "muted": false,
          "snapshot": null
        },
        {
          "cue": null,
          "level": 1.0,
          "muted": false,
          "snapshot": null
        },
        {
          "cue": null,
          "level": 1.0,
          "muted": false,
          "snapshot": null
        }
      ]
    }
  ],
  "activeSceneIndex": 0
}
```

**Serialization:**
- Save: `JSON.stringify(project, null, 2)`, trigger download via Blob URL
- Load: `JSON.parse()`, validate version and structure, reconstruct state

**Validation on Load:**
- Check `version === '1.0'`
- Verify all referenced identity IDs exist
- Clamp frame numbers to scene duration
- Substitute safe defaults for invalid curve data

### WAV File Format (PCM Export)

**Format:**
- **Sample Rate:** 48000 Hz
- **Bit Depth:** 24-bit PCM (little-endian)
- **Channels:** 2 (stereo, both carrying same signal)
- **Duration:** Exactly scene duration in frames
- **Start:** Sample 0 = frame 0 = time 0.0

**Encoding:**

```typescript
function encodeWAV(buffer: AudioBuffer): ArrayBuffer {
  const numChannels = 2;
  const sampleRate = 48000;
  const bitsPerSample = 24;
  const bytesPerSample = bitsPerSample / 8;
  const numFrames = buffer.length;
  
  const dataSize = numFrames * numChannels * bytesPerSample;
  const headerSize = 44;
  const fileSize = headerSize + dataSize;
  
  const arrayBuffer = new ArrayBuffer(fileSize);
  const view = new DataView(arrayBuffer);
  
  // RIFF header
  writeString(view, 0, 'RIFF');
  view.setUint32(4, fileSize - 8, true);
  writeString(view, 8, 'WAVE');
  
  // fmt chunk
  writeString(view, 12, 'fmt ');
  view.setUint32(16, 16, true); // fmt chunk size
  view.setUint16(20, 1, true); // PCM format
  view.setUint16(22, numChannels, true);
  view.setUint32(24, sampleRate, true);
  view.setUint32(28, sampleRate * numChannels * bytesPerSample, true); // byte rate
  view.setUint16(32, numChannels * bytesPerSample, true); // block align
  view.setUint16(34, bitsPerSample, true);
  
  // data chunk
  writeString(view, 36, 'data');
  view.setUint32(40, dataSize, true);
  
  // Interleave samples
  const monoData = buffer.getChannelData(0);
  let offset = 44;
  
  for (let i = 0; i < numFrames; i++) {
    const sample = Math.max(-1, Math.min(1, monoData[i]));
    const intSample = Math.round(sample * 0x7FFFFF);
    
    // Write left channel (24-bit little-endian)
    view.setUint8(offset++, (intSample) & 0xFF);
    view.setUint8(offset++, (intSample >> 8) & 0xFF);
    view.setUint8(offset++, (intSample >> 16) & 0xFF);
    
    // Write right channel (same as left)
    view.setUint8(offset++, (intSample) & 0xFF);
    view.setUint8(offset++, (intSample >> 8) & 0xFF);
    view.setUint8(offset++, (intSample >> 16) & 0xFF);
  }
  
  return arrayBuffer;
}
```

### Timeline Mapping

**Fundamental Conversion:**

At 24 fps and 48 kHz:
- 1 frame = 1/24 second = 0.041667 seconds
- 1 frame = 2000 samples

**Frame to Sample:**
```typescript
function frameToSample(frame: number): number {
  return frame * 2000;
}
```

**Sample to Frame:**
```typescript
function sampleToFrame(sample: number): number {
  return Math.floor(sample / 2000);
}
```

**Frame to Time:**
```typescript
function frameToTime(frame: number): number {
  return frame / 24.0;
}
```

**Time to Frame:**
```typescript
function timeToFrame(time: number): number {
  return Math.floor(time * 24.0);
}
```

**Hit Position Display:**
```typescript
function formatHitPosition(hitFrame: number): string {
  const sample = frameToSample(hitFrame);
  return `Frame ${hitFrame} (Sample ${sample})`;
}
```

**Scene Duration:**
```typescript
function calculateSceneDuration(durationFrames: number): {
  frames: number;
  seconds: number;
  samples: number;
} {
  return {
    frames: durationFrames,
    seconds: durationFrames / 24.0,
    samples: frameToSample(durationFrames)
  };
}
```

### Visual Timeline Synchronization

**Challenge:** HTML5 video `currentTime` has limited precision (~15ms typical). Seeking may not land exactly on requested frame.

**Approach:**

1. **Playback:** Use video `timeupdate` event (fires ~4x per second) to update strip scroll position. Accept minor jitter; strips follow video, not the other way around.

2. **Scrubbing:** When user drags timeline scrubber, update both video `currentTime` and strip scroll immediately. Video will seek to nearest keyframe; strips show requested position.

3. **Marker Placement:** User clicks on strip or timeline; snap to nearest frame boundary. Store frame number, display at exact pixel position.

**Strip Scroll Calculation:**

```typescript
interface StripViewport {
  widthPixels: number; // canvas width
  visibleDuration: number; // seconds visible in viewport
  scrollPosition: number; // seconds, left edge of viewport
}

function updateStripScroll(
  currentFrame: number,
  viewport: StripViewport
): number {
  const currentTime = frameToTime(currentFrame);
  const centerTime = currentTime;
  const newScrollPosition = centerTime - (viewport.visibleDuration / 2);
  
  return Math.max(0, newScrollPosition);
}

function drawStrip(
  curve: Curve,
  viewport: StripViewport,
  canvas: HTMLCanvasElement
): void {
  const ctx = canvas.getContext('2d');
  const startTime = viewport.scrollPosition;
  const endTime = startTime + viewport.visibleDuration;
  
  // Map curve's normalized x (0-1) to visible time range
  // Draw only points within viewport
  // ...
}
```

**Limitation Disclosure:**

The design acknowledges browser video seek is not frame-accurate. We provide frame-accurate *export* and *marker placement*, but not frame-accurate *playback preview*. This is acceptable for scene-by-scene work where final sync is verified via export.

## Error Handling

### Audio Safety

**Stop Within 100ms (REQ-B3.3):**

```typescript
class AudioEngine {
  private activeNodes: AudioNode[] = [];
  
  stopPlayback(): void {
    const now = this.context.currentTime;
    const stopTime = now + 0.01; // 10ms ramp down
    
    this.activeNodes.forEach(node => {
      if ('stop' in node) {
        (node as AudioScheduledSourceNode).stop(stopTime);
      }
    });
    
    this.gainNode.gain.setValueAtTime(this.gainNode.gain.value, now);
    this.gainNode.gain.linearRampToValueAtTime(0, stopTime);
    
    // Clear references after stop time
    setTimeout(() => {
      this.activeNodes = [];
    }, 100);
  }
}
```

**Invalid Parameter Handling:**

```typescript
function sanitizeGain(value: number): number {
  if (!isFinite(value) || isNaN(value)) return 0;
  return Math.max(0, Math.min(1, value));
}

function sanitizeFrequency(value: number): number {
  if (!isFinite(value) || isNaN(value)) return 440;
  return Math.max(20, Math.min(20000, value)); // human hearing range
}
```

**Unstable Output Prevention:**

- Clamp all curve evaluations to [0, 1]
- Validate frequency range before applying
- Check for NaN/Infinity before setting Web Audio parameters
- If synthesis throws, fall back to silence rather than crash

### File Operations

**Video Load Failure:**

```typescript
async function loadVideo(file: File): Promise<VideoMetadata> {
  const url = URL.createObjectURL(file);
  const video = document.createElement('video');
  
  return new Promise((resolve, reject) => {
    video.addEventListener('loadedmetadata', () => {
      const durationSeconds = video.duration;
      const durationFrames = Math.floor(durationSeconds * 24);
      
      if (durationFrames > 60 * 24) {
        reject(new Error('Video exceeds 60 second limit'));
        return;
      }
      
      resolve({
        url,
        durationFrames,
        durationSeconds,
        width: video.videoWidth,
        height: video.videoHeight
      });
    });
    
    video.addEventListener('error', () => {
      reject(new Error(`Cannot decode video: ${video.error?.message}`));
    });
    
    video.src = url;
  });
}
```

**Project Load Validation:**

```typescript
function validateProject(data: unknown): Project {
  if (typeof data !== 'object' || data === null) {
    throw new Error('Invalid project file: not an object');
  }
  
  const obj = data as Record<string, unknown>;
  
  if (obj.version !== '1.0') {
    throw new Error(`Unsupported project version: ${obj.version}`);
  }
  
  // Validate structure, substitute defaults where needed
  const project: Project = {
    version: '1.0',
    identityShelf: validateIdentityShelf(obj.identityShelf),
    scenes: validateScenes(obj.scenes, obj.identityShelf),
    activeSceneIndex: typeof obj.activeSceneIndex === 'number' 
      ? obj.activeSceneIndex 
      : 0
  };
  
  return project;
}
```

**Export Failure Recovery:**

```typescript
async function exportWAV(scene: Scene): Promise<Blob> {
  try {
    const buffer = await renderOffline(scene);
    const wav = encodeWAV(buffer);
    
    // Verify before returning
    const peakDB = verifyPeakLevel(buffer);
    if (peakDB > -6.0) {
      console.warn(`Peak level ${peakDB.toFixed(1)} dB exceeds -6 dBFS`);
    }
    
    return new Blob([wav], { type: 'audio/wav' });
  } catch (error) {
    throw new Error(`Export failed: ${error.message}`);
  }
}
```

### User-Facing Error Messages

- **Cannot decode video:** "The selected video file cannot be decoded. Please use a supported format (MP4, WebM)."
- **Video too long:** "The video exceeds the 60 second limit. Current duration: X seconds."
- **Frame rate mismatch:** "Please confirm the video is 24 fps. Other frame rates are not supported in this version."
- **Export failed:** "Audio export failed. Please check that your scene has at least one unmuted cue and try again."
- **Load failed:** "Cannot open project file. The file may be corrupted or from an incompatible version."

### State Recovery

**Undo/Redo Implementation:**

```typescript
interface ProjectSnapshot {
  timestamp: number;
  state: Project;
}

class UndoManager {
  private undoStack: ProjectSnapshot[] = [];
  private redoStack: ProjectSnapshot[] = [];
  private maxStackSize = 50;
  
  pushState(project: Project): void {
    this.undoStack.push({
      timestamp: Date.now(),
      state: deepClone(project)
    });
    
    if (this.undoStack.length > this.maxStackSize) {
      this.undoStack.shift();
    }
    
    this.redoStack = []; // clear redo on new action
  }
  
  undo(currentProject: Project): Project | null {
    if (this.undoStack.length === 0) return null;
    
    this.redoStack.push({
      timestamp: Date.now(),
      state: deepClone(currentProject)
    });
    
    const snapshot = this.undoStack.pop()!;
    return snapshot.state;
  }
  
  redo(currentProject: Project): Project | null {
    if (this.redoStack.length === 0) return null;
    
    this.undoStack.push({
      timestamp: Date.now(),
      state: deepClone(currentProject)
    });
    
    const snapshot = this.redoStack.pop()!;
    return snapshot.state;
  }
}
```

**Actions that push undo state:**
- Draw on strip
- Move marker
- Change layer level
- Apply identity
- Save snapshot
- Delete cue

**Actions that do NOT push undo state:**
- Play/pause
- Scrub timeline
- Change volume
- Toggle mute
- A/B comparison toggle

## Testing Strategy

### Unit Tests

**Specific examples and edge cases:**

- Timeline conversion functions (frame/sample/time)
- Curve evaluation at boundaries (x=0, x=1, x=0.5)
- Waveform buffer creation from simple curves
- WAV header encoding with known parameters
- Project serialization with minimal and maximal data
- Marker movement with boundary constraints
- Peak level verification with known test signals

**Integration points:**

- Video element interaction (load, play, seek events)
- Web Audio context lifecycle (real-time and offline)
- File I/O (save project, load project, export WAV)
- Canvas drawing and mouse interaction

**Error conditions:**

- Invalid video file
- Video exceeds duration limit
- Corrupt project file
- Missing identity reference
- Out-of-bounds frame numbers

### Property-Based Tests

Property-based testing is appropriate for GhostReel because:

- Core logic involves pure functions (timeline conversions, curve evaluation, data transformations)
- Serialization and deserialization should round-trip reliably
- Curve refitting has universal properties (anchor preservation)
- Audio rendering has invariants (peak limits, duration accuracy)

**Property test library:** fast-check (TypeScript)

**Property test configuration:**
- Minimum 100 iterations per property test
- Each test tagged with comment referencing design property
- Tag format: `// Feature: ghostreel-mvp, Property {number}: {property_text}`

**Testing approach:**

- **Unit tests** verify specific examples, edge cases, and error conditions
- **Property tests** verify universal properties across randomized inputs
- Together they provide comprehensive coverage

(Correctness Properties section will be written after prework analysis)

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Timeline Conversion Round Trip

*For any* frame number in the valid range [0, 1440] (60 seconds at 24 fps), converting frame → sample → frame SHALL return the original frame number.

**Validates: Requirements REQ-C1.3, REQ-C1.4, REQ-C1.5**

### Property 2: Project Serialization Round Trip

*For any* valid project state, serializing to JSON and then deserializing SHALL produce an equivalent project with the same identities, cues, markers, and settings.

**Validates: Requirements REQ-F1.1, REQ-F1.2, REQ-F1.3**

### Property 3: Move Start Preserves Hit and Post-Hit

*For any* cue with start < hit < end, moving the start frame to a new position within [0, hit) SHALL preserve the hit frame position and all curve points in the post-hit segment (hit to end).

**Validates: Requirements REQ-C3.1, REQ-C3.2, REQ-C3.6**

### Property 4: Move End Preserves Hit and Pre-Hit

*For any* cue with start < hit < end, moving the end frame to a new position within (hit, sceneDuration] SHALL preserve the hit frame position and all curve points in the pre-hit segment (start to hit).

**Validates: Requirements REQ-C3.3**

### Property 5: Move Hit Preserves Segment Durations

*For any* cue with start < hit < end, moving the hit frame SHALL move all three markers together such that (hit - start) and (end - hit) remain unchanged.

**Validates: Requirements REQ-C3.4**

### Property 6: Waveform Unchanged by Marker Movement

*For any* cue and any marker movement (start, hit, or end), the waveform curve SHALL remain identical to its original state.

**Validates: Requirements REQ-C3.6**

### Property 7: Identity Application Round Trip

*For any* saved identity, applying it to a cue and then saving that cue's waveform as a new identity SHALL produce a waveform with equivalent curve points.

**Validates: Requirements REQ-D1.4, REQ-D2.1**

### Property 8: Layer Independence

*For any* scene with multiple layers, editing one layer's cue (curves, markers, level) SHALL leave all other layers' cue data and settings unchanged.

**Validates: Requirements REQ-C2.4, REQ-C3.6**

### Property 9: Peak Level Ceiling

*For any* scene with valid cues, the exported WAV file SHALL have a peak level at or below -6 dBFS when measured across all samples in all channels.

**Validates: Requirements REQ-G1.6**

### Property 10: Export Duration Accuracy

*For any* scene with duration D frames, the exported WAV file SHALL contain exactly D × 2000 samples, matching the scene duration precisely.

**Validates: Requirements REQ-G1.4, REQ-C1.4**

### Property 11: Curve Evaluation Range

*For any* curve and any x value in [0, 1], evaluating the curve SHALL return a y value in [0, 1], regardless of the curve's control points.

**Validates: Requirements REQ-B3.5 (safe parameter values)**

### Property 12: Mute Excludes Layer

*For any* scene with at least one muted layer, the exported audio SHALL be equivalent to exporting with that layer removed from the scene.

**Validates: Requirements REQ-C2.3, REQ-G1.2**

### Property 13: Snapshot Comparison Preserves Current State

*For any* layer with a saved snapshot, toggling A/B comparison on and off SHALL return the current cue to its exact state before comparison began.

**Validates: Requirements REQ-E1.3, REQ-E1.4**

### Property 14: Identity Editing Does Not Affect Existing Cues

*For any* cue created from an identity, subsequently editing and re-saving that identity SHALL leave the existing cue's waveform, pitch, and intensity curves unchanged.

**Validates: Requirements REQ-D2.3**

### Property 15: Stop Clears All Audio

*For any* playback state with synthesized audio running, calling stop SHALL result in all audio ceasing within 100ms, and subsequent play/stop cycles SHALL never leave audio running.

**Validates: Requirements REQ-B3.3**

## Implementation Risks and Mitigation

### High-Risk Areas

#### 1. Browser Audio Latency and Synchronization

**Risk:** Web Audio playback and video playback use separate clocks. Drift may cause audio/video desynchronization during real-time preview.

**Mitigation:**
- Use video `timeupdate` as the synchronization source
- Accept that preview has jitter; prioritize export accuracy
- Restart audio synthesis if drift exceeds 100ms
- Document that frame-accurate preview is not guaranteed; only export is sample-accurate

#### 2. Curve Refitting Complexity

**Risk:** The anchored curve refitting logic (moving start refits pre-hit, moving end refits post-hit) is complex and error-prone.

**Mitigation:**
- Implement property tests for all marker movement scenarios
- Visualize before/after curves in development UI
- Test with extreme cases (1-frame segments, full scene spans)
- Provide undo so users can recover from unexpected refitting

#### 3. Memory Limits for 60-Second Clips

**Risk:** Holding decoded video frames and offline-rendered audio in memory may exceed browser limits on lower-end hardware.

**Mitigation:**
- Measure memory usage on target hardware (test on laptop with 8GB RAM)
- Use OfflineAudioContext which handles buffers internally
- Release video URL objects when loading new clips
- Document observed limits in README

**Measured limit to document:** Maximum tested clip duration and resolution (e.g., "Tested with 60-second 1080p video on 8GB RAM Windows laptop, Chrome 120").

#### 4. WAV Export Correctness

**Risk:** Incorrect WAV header or sample encoding will produce files that audio editors reject or import incorrectly.

**Mitigation:**
- Test exported WAV files in DaVinci Resolve, Audacity, and Pro Tools (if available)
- Validate WAV header with hex dump
- Implement automated test: encode known buffer, decode, compare samples
- Use established libraries if hand-coding proves unreliable (e.g., wav-encoder npm package)

#### 5. Web Audio API Scheduling Precision

**Risk:** `setValueAtTime()` scheduling may have precision limits or rounding that cause audible artifacts.

**Mitigation:**
- Sample curves at frame rate (24 fps = 41.67ms intervals), acceptable for parameter automation
- Use `linearRampToValueAtTime()` for gain to smooth transitions
- Test with high-frequency pitch modulation to detect artifacts
- Document observed precision limits

### Medium-Risk Areas

#### 6. Canvas Drawing Performance

**Risk:** Real-time strip rendering during video playback may drop frames on slower hardware.

**Mitigation:**
- Render strips at lower frequency than video frames (e.g., 10 fps)
- Use requestAnimationFrame for drawing
- Simplify strip rendering (fewer path points, simpler styling)
- Profile on target hardware, optimize hot paths

#### 7. Undo Stack Memory

**Risk:** Deep-cloning entire project state 50 times may consume excessive memory.

**Mitigation:**
- Clone only the active scene, not the entire project
- Compress undo states (delta encoding) if memory becomes an issue
- Limit undo stack to 20 entries if necessary
- Measure memory impact during testing

#### 8. Video Seek Reliability

**Risk:** Some video codecs (e.g., H.264 with long GOP) may seek to incorrect frames or produce visual glitches.

**Mitigation:**
- Recommend keyframe-rich encoding for reference videos
- Test with both H.264 and VP9 codecs
- Document observed seek behavior
- Prioritize export accuracy over preview accuracy

### Mitigation Summary

The highest-risk items (audio/video sync, curve refitting, WAV export) are addressed through:

1. **Property-based testing** for curve refitting and export correctness
2. **Multi-tool validation** for WAV export (import into professional tools)
3. **Acceptance of preview limitations** with focus on export accuracy
4. **Measurement and documentation** of observed performance limits

## Performance Considerations

### Target Hardware

- **Minimum:** Laptop with 8GB RAM, integrated graphics, current Chrome
- **Tested:** Windows 10/11 and macOS 12+ on mid-range laptops

### Performance Budget

- **Initial load:** < 3 seconds to show bundled scene
- **Video open:** < 5 seconds for 60-second 1080p clip
- **Playback:** Smooth 24 fps video with real-time synthesis (no dropped frames)
- **Export:** < 10 seconds for 60-second scene with 3 layers
- **Project save:** < 1 second for typical project (< 100KB file)
- **Strip rendering:** 10 fps minimum during playback

### Memory Budget

- **Video:** ~200 MB for decoded 60-second 1080p clip (browser-managed)
- **Audio buffers:** ~5 MB for 60-second stereo at 48 kHz (browser-managed)
- **Undo stack:** ~5 MB for 20 snapshots
- **Total working set:** < 250 MB

### Optimization Strategies

1. **Lazy rendering:** Only draw visible strip segments
2. **Throttled updates:** Update strips at 10 fps during playback, not 24 fps
3. **Buffer reuse:** Reuse AudioBuffer objects where possible
4. **Minimal DOM:** Use Canvas for strips, not SVG or DOM elements
5. **Efficient curves:** Linear interpolation only (cubic is stretch goal)

### Measurement Plan

Before deployment, measure and document:

1. Memory usage during typical workflow (Chrome DevTools)
2. Export time for 60-second scene with 3 layers
3. Playback smoothness (video frame drops) on target laptop
4. Initial load time on slow network (3G simulation)

Report these measurements in the README under a "Performance" section.

## Deployment

### Build Process

```bash
npm run build
# Produces static files in dist/
# - index.html
# - assets/*.js (code-split bundles)
# - assets/*.css
# - bundled-assets/ (demo video files)
```

### Hosting

- **Host:** Vercel, Netlify, or GitHub Pages (any static host)
- **URL:** Custom domain or platform-provided subdomain
- **HTTPS:** Required (browser security policies for video and audio)
- **CORS:** Not needed (all assets served from same origin)

### Asset Attribution

Bundled demonstration videos must include:

- **Credits.md** in repository documenting permissions
- Clear attribution in app UI (footer or about screen)
- Original footage or properly licensed with documented permissions

### Availability Guarantee

- App must remain deployed and accessible until 29 January 2027
- Choose host with free tier or pay for hosting period
- Set calendar reminder to check availability weekly

### Pre-Deployment Checklist

- [ ] Complete demonstration scenario works end-to-end
- [ ] Export → import into video editor → verify hit lands on correct frame
- [ ] Tested on Chrome Windows and Chrome macOS
- [ ] Unsupported browser message displays correctly
- [ ] All bundled assets have documented permissions
- [ ] README includes performance measurements
- [ ] LICENSE file at repository root (MIT)
- [ ] `.kiro` directory in repository, not in `.gitignore`
- [ ] Deployment URL documented in submission materials

## Open Questions

None at this time. All technical questions from the brief (section 8) have been addressed in this design.

## Appendix: Starter Recipes

### Tension Pulse

A rhythmic low-frequency pulse for building dread.

```typescript
const tensionPulse: Identity = {
  id: 'starter-tension-pulse',
  name: 'Tension Pulse',
  waveform: {
    points: [
      { x: 0.0, y: 0.0 },
      { x: 0.25, y: 0.7 },
      { x: 0.5, y: 0.0 },
      { x: 0.75, y: -0.7 },
      { x: 1.0, y: 0.0 }
    ],
    interpolation: 'linear'
  },
  defaultPitch: {
    points: [
      { x: 0.0, y: 0.2 },
      { x: 0.5, y: 0.3 },
      { x: 1.0, y: 0.2 }
    ],
    interpolation: 'linear'
  },
  defaultIntensity: {
    points: [
      { x: 0.0, y: 0.0 },
      { x: 0.3, y: 0.6 },
      { x: 0.5, y: 1.0 },
      { x: 0.7, y: 0.6 },
      { x: 1.0, y: 0.0 }
    ],
    interpolation: 'linear'
  },
  recipeSettings: {
    baseFrequency: 110,
    frequencyRange: [60, 220],
    gainRange: [0, 0.4]
  }
};
```

### Energy Build and Release

A rising pitch and intensity that releases at the hit.

```typescript
const energyBuild: Identity = {
  id: 'starter-energy-build',
  name: 'Energy Build',
  waveform: {
    points: [
      { x: 0.0, y: 0.0 },
      { x: 0.33, y: 0.8 },
      { x: 0.66, y: -0.8 },
      { x: 1.0, y: 0.0 }
    ],
    interpolation: 'linear'
  },
  defaultPitch: {
    points: [
      { x: 0.0, y: 0.2 },
      { x: 0.5, y: 0.8 }, // hit: peak frequency
      { x: 1.0, y: 0.1 }
    ],
    interpolation: 'linear'
  },
  defaultIntensity: {
    points: [
      { x: 0.0, y: 0.1 },
      { x: 0.5, y: 1.0 }, // hit: peak intensity
      { x: 0.8, y: 0.3 },
      { x: 1.0, y: 0.0 }
    ],
    interpolation: 'linear'
  },
  recipeSettings: {
    baseFrequency: 220,
    frequencyRange: [110, 880],
    gainRange: [0, 0.5]
  }
};
```

### Stylized Presence

An eerie, sustained tone with subtle movement.

```typescript
const stylizedPresence: Identity = {
  id: 'starter-stylized-presence',
  name: 'Stylized Presence',
  waveform: {
    points: [
      { x: 0.0, y: 0.0 },
      { x: 0.2, y: 0.5 },
      { x: 0.4, y: 0.0 },
      { x: 0.6, y: -0.3 },
      { x: 0.8, y: 0.0 },
      { x: 1.0, y: 0.0 }
    ],
    interpolation: 'linear'
  },
  defaultPitch: {
    points: [
      { x: 0.0, y: 0.4 },
      { x: 0.3, y: 0.5 },
      { x: 0.5, y: 0.45 },
      { x: 0.7, y: 0.5 },
      { x: 1.0, y: 0.4 }
    ],
    interpolation: 'linear'
  },
  defaultIntensity: {
    points: [
      { x: 0.0, y: 0.0 },
      { x: 0.2, y: 0.6 },
      { x: 0.5, y: 0.7 },
      { x: 0.8, y: 0.6 },
      { x: 1.0, y: 0.0 }
    ],
    interpolation: 'linear'
  },
  recipeSettings: {
    baseFrequency: 165,
    frequencyRange: [110, 330],
    gainRange: [0, 0.35]
  }
};
```

These recipes will be refined during implementation to ensure they produce recognizable, editable sounds that demonstrate the strip-drawing concept effectively.
