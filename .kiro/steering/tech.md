# Technology: GhostReel

## Stack

- **React 18+** with TypeScript for UI components and state management
- **Vite** for build tooling and development server
- **Vitest** for unit tests
- **fast-check** for property-based tests
- **Pure function rendering** for synthesis (Web Audio-agnostic, testable in Node)
- **Web Audio API** for playback only (AudioBufferSourceNode)
- **HTML5 Video Element** for picture playback
- **Canvas 2D** for drawable strips
- **Web Workers** for debounced re-rendering after edits (if needed)
- **No server-side processing:** all work happens in the browser

## Audio Architecture

### Pure Renderer (Web Audio-Agnostic)

```typescript
function renderScene(scene: Scene, sampleRate: number = 48000): RenderResult
```

- Returns `{samples: Float32Array, reductionDb: number}` (mono output)
- Testable in Node with fast-check
- Used for both preview and export
- No Web Audio dependency in the render path

### Playback Engine (Browser Only)

- Connects rendered buffer to Web Audio via `AudioBufferSourceNode`
- Starts at offset matching `video.currentTime`
- Detects and fixes audio/video drift (>40ms)
- Applies volume via `GainNode`

### Synthesis Method

**Per-octave band-limited wavetable synthesis:**

- Compute harmonics from the 2048-sample waveform table via FFT
- Build one band-limited table per octave, keeping only harmonics below Nyquist for that octave's top frequency
- Select the table based on current frequency during phase accumulation
- **Supported frequency range:** 20 Hz to 2 kHz (clamp outside this range)
- **Fallback:** If pure band-limiting proves infeasible, use `OfflineAudioContext` with `OscillatorNode` and `PeriodicWave` behind the same signature

### Render Caching Strategy

To meet the under-250ms re-render budget for 60-second scenes:

1. **Cache one rendered buffer per layer** (only audio data, not full scene mix)
2. **Re-render only the edited layer** when curves or markers change
3. **Mix all layers and apply peak ceiling** to produce final buffer
4. **Debounce during pointer drag:** re-render on pointer-up only
5. **If playing:** swap to new buffer at current offset with 10ms crossfade
6. **Curve evaluation optimization:** maintain forward-moving segment index instead of searching per sample

## Timeline Fundamentals

At 24 fps and 48 kHz:

- 1 frame = 1/24 second ≈ 0.041667 seconds
- 1 frame = 2000 samples

Origin: Sample 0 = frame 0 = time 0.0s

## Export Format

- **Sample Rate:** 48000 Hz
- **Bit Depth:** 24-bit PCM (little-endian)
- **Channels:** 2 (stereo, both carrying same mono signal)
- **Format:** WAV

## Performance Budget

- Initial load: < 3 seconds
- Re-render after edit (60s scene, 1 layer): < 250 ms
- Strip rendering: 60 fps target via requestAnimationFrame
- Memory: < 250 MB working set

## Deployment

- **Host:** Vercel, Netlify, or GitHub Pages (static hosting)
- **HTTPS required** (browser security policies)
- **Available until:** 29 January 2027
- **Tested on:** current Chrome (Windows and macOS)

## Browser Requirements

- AudioContext
- requestAnimationFrame
- Float32Array
- Video MP4 support
- OfflineAudioContext

If any missing, show: "GhostReel requires a modern browser. Tested on current Chrome for Windows and macOS."
