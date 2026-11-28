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
- **No Hidden State:** All sound parameters visible on strips and saveable

### Technology Stack

- **React 18+** with TypeScript for UI components and state management
- **Vite** for build tooling and development server
- **Pure function rendering** for synthesis (Web Audio-agnostic, testable in Node)
- **Web Audio API** for playback only (AudioBufferSourceNode)
- **HTML5 Video Element** for picture playback
- **Canvas 2D** for drawable strips
- **Web Workers** for debounced re-rendering after edits
- **No server-side processing:** all work happens in the browser

## Architecture

### High-Level Component Structure

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

### Component Responsibilities

**Application Shell:** Global project state, playback coordination, file operations, undo/redo, transport controls, volume, triggers re-render after edits (debounced).

**Video Component:** HTML5 video element wrapper, exposes playback time and frame position, responds to play/pause/seek, emits frame updates via requestAnimationFrame.

**Pure Renderer:** `renderScene(scene, sampleRate) -> RenderResult`. Wavetable synthesis, per-sample curve evaluation, mixing, peak limiting. No Web Audio dependency, testable in Node.

**Playback Engine:** Connects rendered buffer to Web Audio via AudioBufferSourceNode, starts at offset matching video.currentTime, detects and fixes audio/video drift (>40ms), applies volume.

**Strip Editor Canvas:** Renders three strips per layer, handles drawing interactions, displays cue markers on strips and timeline, scrolls in sync with video via requestAnimationFrame (60 fps target). Waveform strip tiles single-cycle waveform; editing any tile edits the cycle.

**Identity & Snapshot Manager:** Maintains identity shelf, creates/updates/deletes identities, applies identities to cues (copies data, fits curves to cue duration), saves/loads snapshots for A/B comparison, provides three starter recipes, handles missing identities gracefully (identityId is provenance only).

### State Management Strategy

Use React Context + useReducer for global state:

- **project:** Project (identities, scenes, active scene index)
- **playback:** isPlaying, currentFrame, volume, mode ('synthesis' | 'verifyExport'), optional exportedAudioBuffer
- **ui:** selectedLayer (0-2), selectedStrip, comparisonActive, tourDismissed, creditNoteDismissed, activeSceneIndex
- **undoStack/redoStack:** ProjectSnapshot arrays (max 50 entries)

Local component state for transient interactions (mouse position, hover, drag).

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
  x: number; // waveform: phase 0-1 (0° to 360°); pitch/intensity: normalized time 0-1 (0=start, 0.5=hit, 1=end)
  y: number; // waveform: -1 to 1 (bipolar), pitch/intensity: 0 to 1
}
```

### Curve-to-Time Mapping

Waveform curves represent a single cycle using phase (x = 0 to 1 = 0° to 360°) and have NO timing meaning. Only pitch and intensity curves map to scene time.

Pitch and intensity cues have three anchor frames: start, hit, and end. These curves use normalized x coordinates where:

- **x = 0.0** corresponds to the **start frame**
- **x = 0.5** corresponds to the **hit frame**
- **x = 1.0** corresponds to the **end frame**

The mapping from curve x to sample position is **piecewise linear**:

```typescript
function curveXToSample(x: number, cue: Cue): number {
  const startSample = frameToSample(cue.startFrame);
  const hitSample = frameToSample(cue.hitFrame);
  const endSample = frameToSample(cue.endFrame);
  
  if (x <= 0.5) {
    // Pre-hit: map [0, 0.5] to [startSample, hitSample]
    const t = x / 0.5; // 0 to 1
    return startSample + t * (hitSample - startSample);
  } else {
    // Post-hit: map [0.5, 1] to [hitSample, endSample]
    const t = (x - 0.5) / 0.5; // 0 to 1
    return hitSample + t * (endSample - hitSample);
  }
}

function sampleToCurveX(sample: number, cue: Cue): number {
  const startSample = frameToSample(cue.startFrame);
  const hitSample = frameToSample(cue.hitFrame);
  const endSample = frameToSample(cue.endFrame);
  
  if (sample <= hitSample) {
    const t = (sample - startSample) / (hitSample - startSample);
    return t * 0.5;
  } else {
    const t = (sample - hitSample) / (endSample - hitSample);
    return 0.5 + t * 0.5;
  }
}
```

**Key properties:**

1. **x = 0.5 always maps to hit sample:** `curveXToSample(0.5, cue) === frameToSample(cue.hitFrame)`
2. **Monotonic:** Increasing x always maps to increasing sample position
3. **No curve point changes when markers move:** Moving start or end changes only the time mapping, not the curve points themselves
4. **Cue validity:** startFrame < hitFrame < endFrame with at least one frame on each side (enforced when markers move and when project loads; prevents divide-by-zero)

This mapping is used for both rendering (evaluating curves at sample positions) and drawing (displaying curves aligned to the timeline).

**Marker movement implications:**

- **Move start:** Changes pre-hit time span; curve points unchanged
- **Move end:** Changes post-hit time span; curve points unchanged  
- **Move hit:** Moves all three markers together; preserves (hit - start) and (end - hit) durations

### Audio Rendering

**Core Signature:**

```typescript
function renderScene(scene: Scene, sampleRate: number = 48000): RenderResult

interface RenderResult {
  samples: Float32Array; // mono output
  reductionDb: number; // 0 if no limiting applied, negative if peak was reduced
}
```

Produces mono output (single channel). Preview and export both use this same buffer. The function is Web Audio-agnostic so it can run in Node tests with fast-check.

**Wavetable Preparation:**

Each cue's waveform curve (bipolar, -1 to 1) becomes a 2048-sample lookup table:

```typescript
function prepareWavetable(curve: Curve): Float32Array {
  const table = new Float32Array(2048);
  for (let i = 0; i < 2048; i++) {
    table[i] = evaluateCurve(curve, i / 2048); // bipolar -1 to 1
  }
  
  // Remove DC offset
  const mean = table.reduce((sum, v) => sum + v, 0) / 2048;
  for (let i = 0; i < 2048; i++) table[i] -= mean;
  
  // Normalize peak to ±1
  let peak = 0;
  for (let i = 0; i < 2048; i++) {
    const abs = Math.abs(table[i]);
    if (abs > peak) peak = abs;
  }
  if (peak > 0) {
    for (let i = 0; i < 2048; i++) table[i] /= peak;
  }
  
  return table;
}
```

**Phase Accumulator Synthesis:**

```typescript
function renderCue(cue: Cue, sampleRate: number, output: Float32Array, layerLevel: number): void {
  const wavetable = prepareWavetable(cue.waveform);
  let phase = 0.0; // 0 to 2048
  
  const startSample = frameToSample(cue.startFrame);
  const endSample = frameToSample(cue.endFrame);
  const fadeInSamples = Math.floor(0.005 * sampleRate); // 5ms
  const fadeOutSamples = Math.floor(0.020 * sampleRate); // 20ms
  
  for (let sample = startSample; sample < endSample; sample++) {
    const x = sampleToCurveX(sample, cue);
    
    // Evaluate curves -> Hz and linear gain
    const pitchY = clamp(evaluateCurve(cue.pitch, x), 0, 1);
    const [minFreq, maxFreq] = cue.recipeSettings.frequencyRange;
    const freq = minFreq + pitchY * (maxFreq - minFreq);
    
    const intensityY = clamp(evaluateCurve(cue.intensity, x), 0, 1);
    const [minGain, maxGain] = cue.recipeSettings.gainRange;
    let gain = minGain + intensityY * (maxGain - minGain);
    
    // Apply cue-boundary fades
    const offset = sample - startSample;
    if (offset < fadeInSamples) gain *= offset / fadeInSamples;
    const remaining = endSample - sample;
    if (remaining < fadeOutSamples) gain *= remaining / fadeOutSamples;
    
    // Read wavetable with linear interpolation
    const index = phase % 2048;
    const i0 = Math.floor(index);
    const i1 = (i0 + 1) % 2048;
    const frac = index - i0;
    const waveSample = wavetable[i0] * (1 - frac) + wavetable[i1] * frac;
    
    // Mix into output
    output[sample] += waveSample * gain * layerLevel;
    
    // Advance phase
    phase += (freq * 2048) / sampleRate;
  }
}
```

**Mixing and Peak Limiting:**

```typescript
function renderScene(scene: Scene, sampleRate: number = 48000): RenderResult {
  const output = new Float32Array(frameToSample(scene.durationFrames));
  
  for (const layer of scene.layers) {
    if (layer.cue && !layer.muted) {
      renderCue(layer.cue, sampleRate, output, layer.level);
    }
  }
  
  // Scene-end fade (last 20ms)
  const sceneEndFade = Math.floor(0.020 * sampleRate);
  for (let i = Math.max(0, output.length - sceneEndFade); i < output.length; i++) {
    output[i] *= (output.length - i) / sceneEndFade;
  }
  
  // Peak limiting: measure with loop (not Math.max spread), normalize if needed
  let peak = 0;
  for (let i = 0; i < output.length; i++) {
    const abs = Math.abs(output[i]);
    if (abs > peak) peak = abs;
  }
  
  let reductionDb = 0;
  if (peak > 0.5) {
    const factor = 0.5 / peak;
    for (let i = 0; i < output.length; i++) output[i] *= factor;
    reductionDb = 20 * Math.log10(factor);
  }
  
  return { samples: output, reductionDb };
}
```

**Band-Limiting:**

Strategy: Compute the cycle's harmonics once from the 2048-sample waveform table. Build one band-limited table per octave, keeping only harmonics below Nyquist for that octave's top frequency. Select the table based on current frequency during phase accumulation.

- **Supported frequency range:** 20 Hz to 2 kHz (clamp outside this range)
- **Fallback:** If pure band-limiting proves infeasible, use `OfflineAudioContext(1, frames * 2000, 48000)` with `OscillatorNode` and `PeriodicWave` behind the same signature

## Data Models

### Project File Format

Projects are saved as JSON files. Key structure:

```typescript
interface Project {
  version: '1.0';
  identityShelf: Identity[];
  scenes: Scene[]; // max 2
  activeSceneIndex: number;
}
```

**Validation on Load:**

1. Check `version === '1.0'`
2. **Do not fail if an identity ID is missing** — `identityId` is provenance only. If a cue references a deleted identity, allow the project to load; the cue retains its own curves.
3. Clamp frame numbers to scene duration
4. Clamp curve point x and y values to valid ranges (x: 0-1, waveform y: -1 to 1, pitch/intensity y: 0-1)
5. Substitute safe defaults for missing or invalid fields

**Scene Switching:**

The UI provides controls to switch `activeSceneIndex` between the two scenes. Both scenes share the same identity shelf. When switching scenes, re-render the new scene's audio.

### WAV Export Format

- **Sample Rate:** 48000 Hz
- **Bit Depth:** 24-bit PCM (little-endian)
- **Channels:** 2 (stereo, both carrying same mono signal)
- **Duration:** Exactly scene duration in frames
- **Origin:** Sample 0 = frame 0

Export flow: `renderScene` → WAV encoding → download Blob.

### Export Verification (REQ-G2)

User can load a previously exported WAV to verify sync:

1. Parse WAV header directly to extract sample rate, channel count, and duration (no audio decoding yet)
2. Display these values and warn if they differ from expected (48000 Hz, 2 channels, scene duration)
3. Decode the WAV file using Web Audio `decodeAudioData`
4. Play the decoded buffer through an `AudioBufferSourceNode`, synchronized to video
5. Mute the live rendered mix while playing the exported file
6. Display hit marker positions on the timeline for visual verification

### Timeline Mapping and Rounding

**Fundamental Conversion:**

At 24 fps and 48 kHz:
- 1 frame = 1/24 second ≈ 0.041667 seconds
- 1 frame = 2000 samples

```typescript
function frameToSample(frame: number): number {
  return frame * 2000;
}

function sampleToFrame(sample: number): number {
  return Math.floor(sample / 2000);
}

function frameToTime(frame: number): number {
  return frame / 24.0; // start of frame
}

function frameSeekTime(frame: number): number {
  return (frame + 0.5) / 24.0; // middle of frame, for video seeks only
}

function timeToFrame(time: number): number {
  return Math.floor(time * 24.0 + 1e-6); // epsilon for float precision
}
```

**Rounding rule:** Video seeks target the middle of a frame using `frameSeekTime(n)`. This centers the displayed image within the frame interval and reduces seek error.

**Display:**

```typescript
function formatHitPosition(hitFrame: number): string {
  return `Frame ${hitFrame} (Sample ${frameToSample(hitFrame)})`;
}
```

### Playback and Synchronization

**Strip Scrolling:**

Drive from `requestAnimationFrame` reading `video.currentTime` every frame (60 fps target).

```typescript
function syncStripScroll(video: HTMLVideoElement) {
  requestAnimationFrame(() => {
    updateStripPosition(video.currentTime);
    syncStripScroll(video); // loop
  });
}
```

**Audio Playback:**

- Start an `AudioBufferSourceNode` at the buffer offset matching `video.currentTime`
- Check drift every 100ms: if `|video.currentTime - audioTime| > 0.040`, stop and restart audio
- Apply user volume via a `GainNode`

**Live Render Updates:**

Preview re-renders after every edit. To meet the under-250ms budget for 60-second scenes:

1. **Cache one rendered buffer per layer** (only audio data, not full scene mix)
2. **Re-render only the edited layer** when curves or markers change
3. **Mix all layers and apply peak ceiling** to produce final buffer
4. **Debounce during pointer drag:** re-render on pointer-up only
5. **If playing:** swap to new buffer at current offset with 10ms crossfade
6. **Curve evaluation optimization:** maintain forward-moving segment index instead of searching per sample

**Waveform Strip Drawing:**

The waveform strip shows the single-cycle waveform tiled across the cue's time span. Editing any tile edits the underlying cycle (all tiles update). Draw by repeating the cycle at a fixed visual frequency (e.g., every 50 pixels).

## Error Handling

### Audio Safety

Stop ramps the single playback GainNode to zero over 10 ms and stops the one AudioBufferSourceNode.

**Invalid Parameter Handling:**

- Gain values: if not finite or NaN, substitute 0; otherwise clamp to [0, 1]
- Frequency values: if not finite or NaN, substitute 440 Hz; otherwise clamp to [20, 20000]
- Pitch/intensity curve evaluation: clamp all results to [0, 1]

**Unstable Output Prevention:**

- Validate frequency range before applying
- Check for NaN/Infinity before setting Web Audio parameters
- If synthesis throws, fall back to silence rather than crash

### File Operations

**Video Load Validation:**

- Check duration does not exceed 60 seconds (1440 frames)
- Confirm file is decodable as video
- Extract duration, width, height metadata
- Show clear error for unsupported formats

**Project Load Validation:**

- Check version === '1.0'
- Allow projects with missing identity IDs (cue retains its own curves)
- Clamp frame numbers to scene duration
- Enforce cue validity: startFrame < hitFrame < endFrame with at least one frame margin
- Clamp curve point x and y to valid ranges
- Substitute safe defaults for missing fields

**Export Validation:**

- Verify rendered buffer before encoding
- Measure peak level and warn if exceeds -6 dBFS
- Confirm WAV header matches content before download

### User-Facing Error Messages

- **Cannot decode video:** "The selected video file cannot be decoded. Please use a supported format (MP4, WebM)."
- **Video too long:** "The video exceeds the 60 second limit. Current duration: X seconds."
- **Frame rate mismatch:** "Please confirm the video is 24 fps. Other frame rates are not supported in this version."
- **Export failed:** "Audio export failed. Please check that your scene has at least one unmuted cue and try again."
- **Load failed:** "Cannot open project file. The file may be corrupted or from an incompatible version."

### State Recovery

**Undo/Redo**

Maintain undo and redo stacks. On each edit action (draw on strip, move marker, change layer level, apply identity, save snapshot, delete cue):

1. Deep-clone the current project state
2. Push to undo stack (max 50 entries)
3. Clear redo stack

On undo: move current state to redo stack, pop from undo stack.
On redo: move current state to undo stack, pop from redo stack.

Actions that do NOT push undo state: play/pause, scrub timeline, volume, mute toggle, A/B comparison toggle.

## First-Run State and Unsupported Browser Detection

### Tour (REQ-A2)

On first load: Check `sessionStorage.getItem('ghostreel_tour_seen')`. If null, display dismissible 3-4 step tour (parallel strips, hit marker, export). When dismissed, set sessionStorage flag. Display dismissible credit note linking to [https://collection.sciencemuseumgroup.org.uk/objects/co8188395/oramics-machine](https://collection.sciencemuseumgroup.org.uk/objects/co8188395/oramics-machine). Store credit dismissal in localStorage.

### Unsupported Browser Detection (REQ-H1.5)

Feature check: AudioContext, requestAnimationFrame, Float32Array, video MP4 support, OfflineAudioContext. If any missing, show: "GhostReel requires a modern browser. Tested on current Chrome for Windows and macOS."

## Testing Strategy

### Unit Tests

Test specific examples, edge cases, and error conditions:

- Timeline conversions at boundaries
- Curve evaluation at x=0, x=0.5, x=1
- Waveform buffer creation from simple curves
- WAV header encoding with known parameters
- Project serialization (minimal and maximal)
- Marker movement with boundary constraints
- Peak level verification with known signals
- Video element interaction (load, play, seek)
- Web Audio context lifecycle
- File I/O (save, load, export)
- Canvas drawing and mouse interaction
- Invalid video file handling
- Video duration limit enforcement
- Corrupt project file handling
- Missing identity references
- Out-of-bounds frame numbers

### Property-Based Tests

Property-based testing is appropriate for GhostReel because:

- Core logic involves pure functions (timeline conversions, curve evaluation, data transformations)
- Serialization and deserialization should round-trip reliably
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

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Timeline Conversion Round Trip (Frame-Sample-Frame)

**Test Type:** Node, fast-check, pure function

*For any* frame number in [0, 1440], converting frame → sample → frame SHALL return the original frame number.

**Validates: Requirements REQ-C1.3, REQ-C1.4, REQ-C1.5**

### Property 2: Timeline Conversion Round Trip (Frame-Time-Frame and Frame-SeekTime-Frame)

**Test Type:** Node, fast-check, pure function

*For any* frame number in [0, 1440], both frameToTime → timeToFrame and frameSeekTime → timeToFrame SHALL return the original frame number.

**Validates: Requirements REQ-C1.3, REQ-C1.5**

### Property 3: Project Serialization Round Trip

**Test Type:** Node, fast-check, pure function

*For any* valid project state, serializing to JSON and then deserializing SHALL produce an equivalent project with the same identities, cues, markers, and settings.

**Validates: Requirements REQ-F1.1, REQ-F1.2, REQ-F1.3**

### Property 4: Move Start Preserves Hit and Curve Points

**Test Type:** Node, fast-check, pure function

*For any* cue with start < hit < end, moving the start frame to a new position within [0, hit) SHALL preserve the hit frame position and all curve point coordinates unchanged.

**Validates: Requirements REQ-C3.1, REQ-C3.2, REQ-C3.6**

### Property 5: Move End Preserves Hit and Curve Points

**Test Type:** Node, fast-check, pure function

*For any* cue with start < hit < end, moving the end frame to a new position within (hit, sceneDuration] SHALL preserve the hit frame position and all curve point coordinates unchanged.

**Validates: Requirements REQ-C3.3, REQ-C3.6**

### Property 6: Move Hit Preserves Segment Durations

**Test Type:** Node, fast-check, pure function

*For any* cue with start < hit < end, moving the hit frame SHALL move all three markers together such that (hit - start) and (end - hit) remain unchanged.

**Validates: Requirements REQ-C3.4**

### Property 7: Curve X=0.5 Maps to Hit Sample

**Test Type:** Node, fast-check, pure function

*For any* valid cue, curveXToSample(0.5, cue) SHALL equal frameToSample(cue.hitFrame) exactly.

**Validates: New property - ensures piecewise mapping anchors correctly at hit**

### Property 8: Curve Mapping is Monotonic

**Test Type:** Node, fast-check, pure function

*For any* valid cue and any two curve x values x1 < x2, curveXToSample(x1, cue) < curveXToSample(x2, cue).

**Validates: New property - ensures curve-to-time mapping is strictly increasing**

### Property 9: Identity Application Round Trip

**Test Type:** Node, fast-check, pure function

*For any* saved identity, applying it to a cue and then saving that cue's waveform as a new identity SHALL produce a waveform with equivalent curve points.

**Validates: Requirements REQ-D1.4, REQ-D2.1**

### Property 10: Layer Independence

**Test Type:** Node, fast-check, pure function

*For any* scene with multiple layers, editing one layer's cue (curves, markers, level) SHALL leave all other layers' cue data and settings unchanged.

**Validates: Requirements REQ-C2.4, REQ-C3.6**

### Property 11: Peak Level Ceiling

**Test Type:** Node, fast-check, pure function (calls renderScene)

*For any* scene with valid cues, renderScene SHALL produce a buffer with peak level ≤ 0.5 (within 1e-6 tolerance).

**Validates: Requirements REQ-G1.6 (updated to 0.5 peak, not -6 dBFS)**

### Property 12: Export Duration Accuracy

**Test Type:** Node, fast-check, pure function (calls renderScene)

*For any* scene with duration D frames, renderScene SHALL return a Float32Array of length exactly D × 2000.

**Validates: Requirements REQ-G1.4, REQ-C1.4**

### Property 13: Pitch/Intensity Curve Evaluation Range

**Test Type:** Node, fast-check, pure function

*For any* pitch or intensity curve and any x in [0, 1], evaluating the curve SHALL return y in [0, 1].

**Validates: Requirements REQ-B3.5 (safe parameter values, note: Waveform curves return [-1, 1])**

### Property 14: Mute Excludes Layer

**Test Type:** Node, fast-check, pure function (calls renderScene)

*For any* scene with at least one muted layer, the rendered audio SHALL be equivalent to rendering with that layer removed from the scene.

**Validates: Requirements REQ-C2.3, REQ-G1.2**

### Property 15: Snapshot Comparison Preserves Current State

**Test Type:** Browser example test (requires UI state)

*For any* layer with a saved snapshot, toggling A/B comparison on and off SHALL return the current cue to its exact state before comparison began.

**Validates: Requirements REQ-E1.3, REQ-E1.4**

### Property 16: Identity Editing Does Not Affect Existing Cues

**Test Type:** Node, fast-check, pure function

*For any* cue created from an identity, subsequently editing and re-saving that identity SHALL leave the existing cue's waveform, pitch, and intensity curves unchanged.

**Validates: Requirements REQ-D2.3**

### Property 17: Stop Clears All Audio

**Test Type:** Browser example test (requires Web Audio interaction)

*For any* playback state with audio running, calling stop SHALL cease all audio within 100ms, and subsequent play/stop cycles SHALL never leave audio running.

**Validates: Requirements REQ-B3.3**

### Render Test: Hit Spike Lands at Hit Sample

**Test Type:** Node, example test (specific constructed cue)

Given a cue with unequal lead-in and tail (e.g., startFrame=10, hitFrame=40, endFrame=100) and an intensity curve with a narrow spike at x=0.5, renderScene SHALL place the peak intensity sample at hitFrame × 2000, within ±5 samples tolerance.

**Validates: Curve-to-time mapping correctness under asymmetric timing**

## Performance and Deployment

### Target Hardware

Minimum: Laptop with 8GB RAM, integrated graphics, current Chrome.

### Performance Budget

- Initial load: < 3 seconds
- Re-render after edit (60s scene, 1 layer): < 250 ms
- Strip rendering: 60 fps target via requestAnimationFrame
- Memory: < 250 MB working set

### Deployment

- Host: Vercel, Netlify, or GitHub Pages (static hosting)
- HTTPS required (browser security policies)
- Available until 29 January 2027
- Tested on current Chrome (Windows and macOS)

### Pre-Deployment Checklist

- [ ] Demo scenario works end-to-end
- [ ] Export → import into video editor → verify hit lands on correct frame
- [ ] Tested on Chrome Windows and macOS
- [ ] Unsupported browser message works
- [ ] Bundled assets documented
- [ ] README includes performance measurements
- [ ] LICENSE file present
- [ ] Deployment URL documented

## Open Questions

1. **Measured render time:** How long does renderScene take for a 60-second scene with three layers on target hardware (current laptop, Windows/macOS)?

## Appendix: Starter Recipes

### Tension Pulse

A rhythmic low-frequency pulse for building dread.

- **Waveform:** Triangle-ish shape (0 → 0.7 → 0 → -0.7 → 0)
- **Pitch:** Low, subtle movement (60-220 Hz range, 0.2-0.3 normalized)
- **Intensity:** Peak at hit (0 → 0.6 → 1.0 → 0.6 → 0)

### Energy Build and Release

A rising pitch and intensity that releases at the hit.

- **Waveform:** Asymmetric sawtooth (0 → 0.8 → -0.8 → 0)
- **Pitch:** Rise to peak at hit (110-880 Hz range, 0.2 → 0.8 → 0.1)
- **Intensity:** Peak at hit, decay (0.1 → 1.0 → 0.3 → 0)

### Stylized Presence

An eerie, sustained tone with subtle movement.

- **Waveform:** Complex shape with harmonics (0 → 0.5 → 0 → -0.3 → 0 → 0)
- **Pitch:** Slow variation around mid-range (110-330 Hz, 0.4-0.5 normalized)
- **Intensity:** Sustained with gentle shape (0 → 0.6 → 0.7 → 0.6 → 0)

These recipes will be refined during implementation to ensure they produce recognizable, editable sounds that demonstrate the strip-drawing concept effectively.
