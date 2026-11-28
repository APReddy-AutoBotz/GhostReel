# Structure: GhostReel

## Component Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Application Shell                     │
│  - Project state management                              │
│  - File I/O (open video, save/load project, export WAV) │
│  - Global controls (play/stop, volume, undo/redo)       │
│  - Render coordinator (debounced, Web Worker)           │
└────────────┬────────────────────────────────────────────┘
             │
    ┌────────┴────────┐
    │                 │
┌───▼──────────┐  ┌──▼─────────────────────────────────┐
│ Video        │  │ Pure Renderer (Web Audio-agnostic) │
│ Component    │  │ renderScene(scene) -> Float32Array │
│              │  │ - Wavetable synthesis              │
│ - Playback   │  │ - Per-sample evaluation            │
│ - Scrubbing  │  │ - Mixing                           │
│ - Timeline   │  │ - No DOM, testable in Node         │
└───┬──────────┘  └──┬─────────────────────────────────┘
    │                │
    │                │ (rendered buffer)
    │                │
┌───▼────────────────▼─────────────────────────────────┐
│              Playback Engine                         │
│  - AudioBufferSourceNode at video offset             │
│  - Drift detection and resync                        │
│  - Volume control                                    │
└──────────────────────────────────────────────────────┘
    │
┌───▼──────────────────────────────────────────────────┐
│              Strip Editor Canvas                     │
│  - Three strips per layer (waveform, pitch, intensity)│
│  - Marker placement (start, hit, end)                │
│  - Drawing interaction                               │
│  - rAF-driven scroll sync                            │
└──────────────────────────────────────────────────────┘
    │
┌───▼──────────────────────────────────────────────────┐
│           Identity & Snapshot Manager                │
│  - Identity shelf                                    │
│  - Snapshot storage and comparison                   │
│  - Recipe templates                                  │
└──────────────────────────────────────────────────────┘
```

## Component Responsibilities

- **Application Shell:** Global project state, playback coordination, file operations, undo/redo, transport controls, volume, triggers re-render after edits (debounced)

- **Video Component:** HTML5 video element wrapper, exposes playback time and frame position, responds to play/pause/seek, emits frame updates via requestAnimationFrame

- **Pure Renderer:** `renderScene(scene, sampleRate) -> RenderResult`. Wavetable synthesis, per-sample curve evaluation, mixing, peak limiting. No Web Audio dependency, testable in Node. Located in `src/engine/` which is pure TypeScript with no React, DOM or Web Audio imports

- **Playback Engine:** Connects rendered buffer to Web Audio via AudioBufferSourceNode, starts at offset matching video.currentTime, detects and fixes audio/video drift (>40ms), applies volume. Located in `src/playback/`

- **Strip Editor Canvas:** Renders three strips per layer, handles drawing interactions, displays cue markers on strips and timeline, scrolls in sync with video via requestAnimationFrame (60 fps target). Waveform strip tiles single-cycle waveform; editing any tile edits the cycle

- **Identity & Snapshot Manager:** Maintains identity shelf, creates/updates/deletes identities, applies identities to cues (copies data, fits curves to cue duration), saves/loads snapshots for A/B comparison, provides three starter recipes, handles missing identities gracefully (identityId is provenance only)

## Data Model

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
  identityId: string; // provenance only
  startFrame: number;
  hitFrame: number;
  endFrame: number;
  waveform: Curve; // single-cycle waveform shape
  pitch: Curve; // frequency over time
  intensity: Curve; // amplitude envelope over time
  recipeSettings: RecipeSettings; // read by renderer
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
  frequencyRange: [number, number]; // Hz, min/max for pitch curve
  gainRange: [number, number]; // linear multiplier for intensity curve
}

// Curve is a drawable, editable path
interface Curve {
  points: Point[];
  interpolation: 'linear'; // only linear in MVP
}

interface Point {
  x: number; // waveform: phase 0-1; pitch/intensity: normalized time 0-1
  y: number; // waveform: -1 to 1; pitch/intensity: 0 to 1
}
```

## State Management

Use React Context + useReducer for global state:

- **project:** Project (identities, scenes, active scene index)
- **playback:** isPlaying, currentFrame, volume, mode ('synthesis' | 'verifyExport'), optional exportedAudioBuffer
- **ui:** selectedLayer (0-2), selectedStrip, comparisonActive, tourDismissed, creditNoteDismissed, activeSceneIndex
- **undoStack/redoStack:** ProjectSnapshot arrays (max 50 entries)

Local component state for transient interactions (mouse position, hover, drag).

## File Format

**Project:** JSON file, schema version '1.0'

**Export:** 48 kHz, 24-bit PCM, stereo WAV

**Validation on Load:**

1. Check `version === '1.0'`
2. **Do not fail if an identity ID is missing** — `identityId` is provenance only
3. Clamp frame numbers to scene duration
4. Clamp curve point x and y values to valid ranges
5. Substitute safe defaults for missing or invalid fields
6. Enforce cue validity: startFrame < hitFrame < endFrame with at least one frame margin
