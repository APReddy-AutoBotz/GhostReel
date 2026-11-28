# Audio and Timing Rules: GhostReel

Every audio or timeline change MUST follow these rules.

## Timeline Conversions

At 24 fps and 48 kHz: **1 frame = 2000 samples**

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

## Curve-to-Time Mapping

**Waveform curves:**
- Represent a single cycle using phase (x = 0 to 1 = 0° to 360°)
- Have NO timing meaning
- Are bipolar: y ∈ [-1, 1]

**Pitch and intensity curves:**
- Use normalized x coordinates where:
  - **x = 0.0** corresponds to the **start frame**
  - **x = 0.5** corresponds to the **hit frame**
  - **x = 1.0** corresponds to the **end frame**
- Are unipolar: y ∈ [0, 1]

**Piecewise linear mapping:**

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

## Render Path

**One function for preview and export:**

```typescript
function renderScene(scene: Scene, sampleRate: number = 48000): RenderResult

interface RenderResult {
  samples: Float32Array; // mono output
  reductionDb: number; // 0 if no limiting applied, negative if peak was reduced
}
```

- Produces mono output (single channel)
- Web Audio-agnostic (testable in Node)
- Same path for preview and export

## Peak Ceiling

**Target:** 0.5 peak (linear), not -6 dBFS

**Method:**

1. Mix all layers into output buffer
2. Measure peak with loop (not `Math.max(...spread)`)
3. If peak > 0.5, normalize to 0.5 and record reduction in dB
4. Return `{samples, reductionDb}`

**Do NOT:**
- Use compressor nodes
- Show export-time warnings (return reductionDb instead)

## Fade Envelopes (No Clicks)

**Cue boundary fades:**
- Fade in: 5ms from cue start
- Fade out: 20ms before cue end

**Scene-end fade:**
- Last 20ms of scene

**Playback transitions:**
- 10ms crossfade when swapping buffers during playback

## Invalid Parameter Handling

**Gain values:**
- If not finite or NaN, substitute 0
- Otherwise clamp to [0, 1]

**Frequency values:**
- If not finite or NaN, substitute 440 Hz
- Otherwise clamp to [20, 20000]

**Pitch/intensity curve evaluation:**
- Clamp all results to [0, 1]

## Cue Validity Enforcement

**Always enforce:** startFrame < hitFrame < endFrame

**Minimum margin:** At least 1 frame on each side

**Where to enforce:**
- When markers are moved
- When projects are loaded
- Before rendering

This prevents divide-by-zero in curve-to-time mapping.

## Display Format

```typescript
function formatHitPosition(hitFrame: number): string {
  return `Frame ${hitFrame} (Sample ${frameToSample(hitFrame)})`;
}
```

Show both frames and samples when displaying hit position.
