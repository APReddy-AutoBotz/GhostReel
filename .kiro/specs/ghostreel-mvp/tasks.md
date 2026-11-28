# Implementation Plan: GhostReel MVP

## Overview

This plan breaks down the GhostReel MVP into discrete coding tasks, organized into stages from scaffold through deployment. The implementation language is TypeScript with React. Each task builds on previous work, with checkpoints at reasonable intervals to ensure incremental validation.

Core strategy: Build the audio engine first with pure functions testable in Node, then add the browser UI layer by layer. Measure render performance early to validate the caching strategy.

## Tasks

### Stage A: Scaffold and Initial Setup

- [ ] 1. Initialize Vite + React + TypeScript project
  - Create Vite project with React and TypeScript templates
  - Configure TypeScript with strict mode
  - Set up directory structure: `src/`, `src/components/`, `src/engine/`, `src/types/`, `src/utils/`
  - _Requirements: REQ-H1.1, REQ-H1.2_

- [ ] 2. Configure testing infrastructure
  - Install and configure Vitest for unit tests
  - Install fast-check for property-based tests
  - Create test setup files
  - Verify test runner works with a simple example
  - _Requirements: REQ-H1.4_

- [ ] 3. Add project files and GitHub Pages deployment
  - Create MIT LICENSE file at repository root
  - Create README.md with project description and setup instructions
  - Create .gitignore that does NOT exclude `.kiro/` directory
  - Ensure `.kiro/specs/ghostreel-mvp/` directory is present
  - Configure Vite's `base` option to '/GhostReel/' (exact casing) in vite.config.ts
  - Ensure all asset URLs (including bundled videos) respect the base path
  - Create GitHub Actions workflow at `.github/workflows/deploy.yml` that builds and deploys to GitHub Pages
  - Stop and inform user to enable Pages in repository settings (Settings > Pages > Source: GitHub Actions)
  - After user confirms Pages is enabled, verify deployment at the GitHub Pages URL
  - _Requirements: REQ-H1.1, REQ-H1.2, REQ-H1.3_

### Stage B: Audio Engine Spike

- [ ] 4. Implement core data types
  - Create TypeScript interfaces for Project, Scene, Layer, Cue, Identity, Curve, Point, VideoSource, RecipeSettings, RenderResult
  - Place in `src/types/index.ts`
  - _Requirements: REQ-A1, REQ-B1, REQ-C1, REQ-D1_

- [ ] 5. Implement timeline conversion functions
  - Implement `frameToSample`, `sampleToFrame`, `frameToTime`, `frameSeekTime`, `timeToFrame` in `src/engine/timeline.ts`
  - Use exact formulas from audio-timing.md steering file
  - _Requirements: REQ-C1.3, REQ-C1.4, REQ-C1.5_

- [ ] 6. Write property test for timeline round trip (frame-sample-frame)
  - **Property 1: Timeline Conversion Round Trip (Frame-Sample-Frame)**
  - **Validates: Requirements REQ-C1.3, REQ-C1.4, REQ-C1.5**
  - For any frame in [0, 1440], frame → sample → frame returns original frame
  - Use fast-check with fc.integer({min: 0, max: 1440})
  - Run 100 iterations minimum
  - Tag: `// Feature: ghostreel-mvp, Property 1: Timeline Conversion Round Trip (Frame-Sample-Frame)`
  - _Requirements: REQ-C1.3, REQ-C1.4, REQ-C1.5_

- [ ] 7. Write property test for timeline round trip (frame-time-frame)
  - **Property 2: Timeline Conversion Round Trip (Frame-Time-Frame and Frame-SeekTime-Frame)**
  - **Validates: Requirements REQ-C1.3, REQ-C1.5**
  - For any frame in [0, 1440], both frameToTime → timeToFrame and frameSeekTime → timeToFrame return original frame
  - Use fast-check with fc.integer({min: 0, max: 1440})
  - Run 100 iterations minimum
  - Tag: `// Feature: ghostreel-mvp, Property 2: Timeline Conversion Round Trip (Frame-Time-Frame and Frame-SeekTime-Frame)`
  - _Requirements: REQ-C1.3, REQ-C1.5_

- [ ] 8. Implement curve-to-time mapping functions
  - Implement `curveXToSample` and `sampleToCurveX` in `src/engine/curves.ts`
  - Use piecewise linear mapping from audio-timing.md
  - Enforce x=0.5 maps to hit sample exactly
  - _Requirements: REQ-B1.5, REQ-C3.1, REQ-C3.2, REQ-C3.3_

- [ ] 9. Write property test for curve x=0.5 maps to hit sample
  - **Property 7: Curve X=0.5 Maps to Hit Sample**
  - **Validates: New property - ensures piecewise mapping anchors correctly at hit**
  - For any valid cue, curveXToSample(0.5, cue) === frameToSample(cue.hitFrame) exactly
  - Generate random valid cues with fc.record
  - Run 100 iterations minimum
  - Tag: `// Feature: ghostreel-mvp, Property 7: Curve X=0.5 Maps to Hit Sample`
  - _Requirements: REQ-B1.5_

- [ ] 10. Write property test for curve mapping monotonicity
  - **Property 8: Curve Mapping is Monotonic**
  - **Validates: New property - ensures curve-to-time mapping is strictly increasing**
  - For any valid cue and x1 < x2, curveXToSample(x1, cue) < curveXToSample(x2, cue)
  - Use fc.record for cue and fc.double for x values
  - Run 100 iterations minimum
  - Tag: `// Feature: ghostreel-mvp, Property 8: Curve Mapping is Monotonic`
  - _Requirements: REQ-B1.5_

- [ ] 11. Implement curve evaluation with linear interpolation
  - Implement `evaluateCurve(curve: Curve, x: number): number` in `src/engine/curves.ts`
  - Support linear interpolation between points
  - Clamp pitch/intensity results to [0, 1]
  - Clamp waveform results to [-1, 1]
  - _Requirements: REQ-B1.3, REQ-B1.4_

- [ ] 12. Write property test for pitch and intensity curve evaluation range
  - **Property 13: Pitch and Intensity Curve Evaluation Range**
  - **Validates: Requirements REQ-B1.3, REQ-B1.4**
  - For any curve and any x in [0, 1], evaluateCurve returns value in [0, 1] for pitch/intensity curves
  - Generate random curves with fc.array and fc.double
  - Run 100 iterations minimum
  - Tag: `// Feature: ghostreel-mvp, Property 13: Pitch and Intensity Curve Evaluation Range`
  - _Requirements: REQ-B1.3, REQ-B1.4_

- [ ] 13. Implement wavetable preparation with band-limiting
  - Implement `buildWavetables(curve: Curve)` in `src/engine/synthesis.ts`
  - Returns the full set of per-octave band-limited tables (not one array)
  - Sample curve to 2048-point table
  - Remove DC offset
  - Normalize peak to ±1
  - Compute harmonics via FFT
  - Build per-octave band-limited tables (20 Hz to 2 kHz, clamp outside range)
  - _Requirements: REQ-B1.2_

- [ ] 14. Write property test for band-limited table harmonic cutoff
  - **Property: Band-Limited Table Harmonic Cutoff**
  - **Validates: Band-limiting removes harmonics that would alias**
  - For any octave table, verify that any harmonic whose frequency at the top of that octave exceeds 24 kHz has zero amplitude (within 1e-6 tolerance)
  - Generate random waveforms and check each octave table
  - Run 100 iterations minimum
  - Tag: `// Feature: ghostreel-mvp, Property: Band-Limited Table Harmonic Cutoff`
  - _Requirements: REQ-B1.2_

- [ ] 15. Implement cue-edit functions
  - Implement `moveStart`, `moveEnd`, and `moveHit` functions in `src/engine/cue.ts`
  - Each function returns a new cue with updated marker positions
  - `moveStart`: moves start frame, preserves hit and end frames
  - `moveEnd`: moves end frame, preserves start and hit frames
  - `moveHit`: moves all three markers to preserve lead-in and tail durations
  - Enforce cue validity: startFrame < hitFrame < endFrame with at least 1 frame margin
  - Enforce scene-boundary limits: start ≥ 0, end ≤ durationFrames
  - _Requirements: REQ-C1.1, REQ-C1.2, REQ-C3.1, REQ-C3.2, REQ-C3.3, REQ-C3.4, REQ-C3.5_

- [ ] 16. Write property test for move start preserves hit
  - **Property 4: Move Start Preserves Hit and Curve Points**
  - **Validates: Requirements REQ-C3.1, REQ-C3.2, REQ-C3.6**
  - For any cue with start < hit < end, moving start to new position within [0, hit) preserves hit frame and all curve points unchanged
  - Generate random cues and new start positions
  - Run 100 iterations minimum
  - Tag: `// Feature: ghostreel-mvp, Property 4: Move Start Preserves Hit and Curve Points`
  - _Requirements: REQ-C3.1, REQ-C3.2, REQ-C3.6_

- [ ] 17. Write property test for move end preserves hit
  - **Property 5: Move End Preserves Hit and Curve Points**
  - **Validates: Requirements REQ-C3.3, REQ-C3.6**
  - For any cue with start < hit < end, moving end to new position within (hit, sceneDuration] preserves hit frame and all curve points unchanged
  - Generate random cues and new end positions
  - Run 100 iterations minimum
  - Tag: `// Feature: ghostreel-mvp, Property 5: Move End Preserves Hit and Curve Points`
  - _Requirements: REQ-C3.3, REQ-C3.6_

- [ ] 18. Write property test for move hit preserves segment durations
  - **Property 6: Move Hit Preserves Segment Durations**
  - **Validates: Requirements REQ-C3.4**
  - For any cue with start < hit < end, moving hit moves all three markers such that (hit - start) and (end - hit) remain unchanged
  - Generate random cues and new hit positions
  - Verify segment durations preserved
  - Run 100 iterations minimum
  - Tag: `// Feature: ghostreel-mvp, Property 6: Move Hit Preserves Segment Durations`
  - _Requirements: REQ-C3.4_

- [ ] 19. Implement phase accumulator synthesis for single cue
  - Implement `renderCue(cue: Cue, sampleRate: number, output: Float32Array): void` in `src/engine/synthesis.ts`
  - Layer level is NOT applied here
  - Extract per-sample gain (intensity curve plus cue fades) as pure function `cueGainAtSample(cue, sample)` and use it inside renderCue
  - Use phase accumulator with wavetable lookup
  - Apply frequency from pitch curve via recipe frequencyRange
  - Apply gain from intensity curve via recipe gainRange
  - Apply 5ms fade-in and 20ms fade-out at cue boundaries
  - Handle invalid parameters (substitute 440 Hz, 0 gain)
  - Select band-limited table based on current frequency
  - _Requirements: REQ-B1.2, REQ-B1.3, REQ-B1.4, REQ-B3.4, REQ-B3.5_

- [ ] 20. Implement layer rendering function
  - Implement `renderLayer(layer: Layer, durationFrames: number, sampleRate: number): Float32Array` in `src/engine/synthesis.ts`
  - Allocate fresh buffer for layer duration
  - Call renderCue for the layer's cue (if present and not muted)
  - Apply layer level here and nowhere else
  - Return layer buffer (do NOT write into shared buffer)
  - _Requirements: REQ-B1.6, REQ-C2.5_

- [ ] 21. Implement mix and peak ceiling function
  - Implement `mixAndApplyCeiling(buffers: Float32Array[]): RenderResult` in `src/engine/synthesis.ts`
  - Allocate output buffer matching input buffer length
  - Sum all input buffers sample-by-sample
  - Apply scene-end fade (last 20ms)
  - Measure peak with loop (not Math.max spread)
  - If peak > 0.5, normalize to 0.5 and record reduction in dB
  - Return {samples: Float32Array, reductionDb: number}
  - _Requirements: REQ-G1.1, REQ-G1.2, REQ-G1.6_

- [ ] 22. Implement scene rendering function
  - Implement `renderScene(scene: Scene, sampleRate: number): RenderResult` in `src/engine/synthesis.ts`
  - Call renderLayer for each layer
  - Pass all layer buffers to mixAndApplyCeiling
  - Return the RenderResult
  - _Requirements: REQ-B1.6, REQ-C2.5, REQ-G1.1, REQ-G1.2, REQ-G1.6_

- [ ] 23. Write property test for peak level ceiling
  - **Property 11: Peak Level Ceiling**
  - **Validates: Requirements REQ-G1.6**
  - For any scene with valid cues, renderScene produces buffer with peak ≤ 0.5 (within 1e-6 tolerance)
  - Generate random scenes with fc.record
  - Run 100 iterations minimum
  - Tag: `// Feature: ghostreel-mvp, Property 11: Peak Level Ceiling`
  - _Requirements: REQ-G1.6_

- [ ] 24. Write property test for export duration accuracy
  - **Property 12: Export Duration Accuracy**
  - **Validates: Requirements REQ-G1.4, REQ-C1.4**
  - For any scene with duration D frames, renderScene returns Float32Array of length exactly D × 2000
  - Generate scenes with random durations up to 1440 frames
  - Run 100 iterations minimum
  - Tag: `// Feature: ghostreel-mvp, Property 12: Export Duration Accuracy`
  - _Requirements: REQ-G1.4, REQ-C1.4_

- [ ] 25. Write property test for mute excludes layer
  - **Property 14: Mute Excludes Layer**
  - **Validates: Requirements REQ-C2.3, REQ-G1.2**
  - For any scene with at least one muted layer, rendered audio is equivalent to rendering with that layer removed
  - Generate scenes with some layers muted
  - Compare buffer outputs
  - Run 100 iterations minimum
  - Tag: `// Feature: ghostreel-mvp, Property 14: Mute Excludes Layer`
  - _Requirements: REQ-C2.3, REQ-G1.2_

- [ ] 26. Write render tests for hit spike accuracy
  - **Render Test 1: Intensity Spike at Hit Sample**
  - Construct cue with startFrame=10, hitFrame=40, endFrame=100
  - Use intensity curve with narrow spike at x=0.5
  - Verify maximum of cueGainAtSample over the cue is at exactly sample 80000
  - Tag: `// Feature: ghostreel-mvp, Render Test 1: Intensity Spike at Hit Sample`
  - **Render Test 2: Audio Peak Near Hit Sample**
  - Same cue, pitch held at 1000 Hz
  - Verify largest absolute sample in rendered audio lies within 48 samples of sample 80000
  - Tag: `// Feature: ghostreel-mvp, Render Test 2: Audio Peak Near Hit Sample`
  - _Requirements: REQ-C1.4, REQ-B1.5_

- [ ] 27. Implement WAV encoder
  - Implement `encodeWAV(samples: Float32Array, sampleRate: number): Blob` in `src/engine/wav.ts`
  - Convert mono Float32Array to stereo 24-bit PCM (both channels carry same signal)
  - Write WAV header with correct format, sample rate, bit depth, channel count
  - Return Blob with MIME type 'audio/wav'
  - _Requirements: REQ-G1.7, REQ-G1.8_

- [ ] 28. Implement WAV header parser and add property test
  - Implement `parseWavHeader(data: ArrayBuffer)` in `src/engine/wav.ts`
  - Extract sample rate, channel count, bit depth, and duration from WAV header
  - Return object with parsed values
  - Write property test: for any sample buffer, encodeWAV then parseWavHeader reports 48000 Hz, 2 channels, 24 bits, and exact duration
  - Verify decoding the data returns the samples within 24-bit quantization tolerance
  - Run 100 iterations minimum
  - Tag: `// Feature: ghostreel-mvp, Property: WAV Encode-Parse Round Trip`
  - _Requirements: REQ-G1.7, REQ-G1.8_

- [ ] 29. Create three starter recipe templates
  - Create `src/data/recipes.ts`
  - Define three recipes with Identity format:
    - **Tension Pulse:** Triangle-ish waveform, low pitch (60-220 Hz), peak at hit intensity
    - **Energy Build and Release:** Asymmetric sawtooth, rising pitch (110-880 Hz), peak at hit then decay
    - **Stylized Presence:** Complex harmonics, mid-range pitch (110-330 Hz), sustained intensity
  - Each recipe has waveform curve, default pitch/intensity curves, and recipe settings
  - _Requirements: REQ-B2.1_

- [ ] 30. Create enhanced test page for engine
  - Create `src/TestPage.tsx` with:
    - Canvas for drawing waveform curve
    - Controls to select and audition each of the three starter recipes as full cues with waveform, pitch, and intensity
    - Ability to redraw the waveform curve
    - Button to render and play via Web Audio
    - Button to export WAV and download
  - Wire up renderScene and encodeWAV
  - Verify sound plays and WAV downloads for all three recipes
  - _Requirements: REQ-B1.6, REQ-G1.1, REQ-B2.1_

- [ ] 31. Measure and record render performance
  - Create test scene with 60 seconds (1440 frames), 3 layers with cues
  - Measure time for renderLayer call for one layer
  - Measure time for renderScene call for three layers
  - Record BOTH measurements in README.md under "Performance" section
  - The 250ms budget applies to the one-layer figure
  - If one-layer figure is over 250ms budget, implement per-layer caching now
  - _Requirements: REQ-H1.4_

### CHECKPOINT: Stage B Review

- [ ] 32. Checkpoint - Ensure all tests pass and engine works
  - Ensure all property tests pass
  - Ensure render tests pass
  - Ensure test page can audition all three recipes, redraw waveform, play, and export
  - Ensure performance measurement is documented
  - Stop and wait for user

### Stage C: Picture and Strip Interface

- [ ] 33. Implement application shell
  - Create global state management with React Context and reducer as designed
  - Create main layout component with:
    - Picture display on top
    - Strip canvas area under fixed playhead
    - Transport controls: play, stop, and labelled volume control
    - Layer controls
    - Identity shelf
  - Replace test page as default view
  - Keep test page accessible via route or menu option
  - _Requirements: REQ-A1.7_

- [ ] 34. Implement video component
  - Create `src/components/VideoPlayer.tsx`
  - Wrap HTML5 video element
  - Provide controls for play, pause, seek
  - Emit currentTime updates via requestAnimationFrame
  - Video control offers "Shot 1", "Shot 2" and "Open your own video"
  - Until two-scene task is done, choosing one replaces active scene's video and duration and clears its cues, but keeps identity shelf
  - Validate duration ≤ 60 seconds
  - Require user confirmation of 24 fps frame rate
  - Show descriptive error for unsupported or too-long videos
  - _Requirements: REQ-A1.3, REQ-A1.4, REQ-A1.5, REQ-A1.6_

- [ ] 35. Create bundled starter scene with placeholder clips
  - Generate TWO clips with ffmpeg, 24 fps with large burned-in frame counter:
    - 8 seconds at `public/videos/shot1.mp4`
    - 12 seconds at `public/videos/shot2.mp4`
  - If ffmpeg is NOT installed, stop and ask user to provide video files
  - Do NOT download any video from the internet
  - Create starter cue with recognizable waveform, pitch, and intensity curves
  - Place in initial project state
  - _Requirements: REQ-A1.1, REQ-B2.1, REQ-B2.2_

- [ ] 36. Implement playback engine
  - Create or resume AudioContext only inside user gesture
  - Move playback.ts and renderCoordinator.ts to `src/playback/` directory
  - Add to structure.md: src/engine is pure TypeScript with no React, DOM or Web Audio imports
  - Create `src/playback/playback.ts`
  - Connect rendered buffer to Web Audio via AudioBufferSourceNode
  - Start playback at offset matching video.currentTime
  - Check drift every 100ms: if |video.currentTime - audioTime| > 0.040s, stop and restart audio
  - Apply volume via GainNode
  - Provide stop function that ramps GainNode to zero over 10ms
  - _Requirements: REQ-A1.2, REQ-B3.1, REQ-B3.2, REQ-B3.3_

- [ ] 37. Implement render coordinator with debouncing
  - Create `src/playback/renderCoordinator.ts`
  - Trigger re-render after strip edits (debounced during drag)
  - Cache one rendered buffer per layer
  - Re-render only edited layer when curves or markers change
  - Mix all layers and apply peak ceiling to produce final buffer
  - If playing, swap to new buffer at current offset with 10ms crossfade
  - _Requirements: REQ-B1.6, REQ-C2.4_

- [ ] 38. Implement strip canvas rendering
  - Create `src/components/StripCanvas.tsx`
  - Render three strips per layer (waveform, pitch, intensity) using Canvas 2D
  - Display cue markers (start, hit, end) on strips and timeline
  - Scroll strips in sync with video.currentTime via requestAnimationFrame (60 fps target)
  - For waveform strip, tile single-cycle waveform across cue duration
  - _Requirements: REQ-A1.7, REQ-B1.1_

- [ ] 39. Implement strip drawing interaction
  - Add mouse/pointer event handlers to StripCanvas
  - Support drawing new points and editing existing points
  - Update curve data in project state
  - Debounce re-render during drag (re-render on pointer-up only)
  - _Requirements: REQ-B1.1, REQ-B1.7_

- [ ] 40. Implement marker placement interaction
  - Add controls to place and drag start, hit, and end markers
  - Call cue-edit functions (moveStart, moveEnd, moveHit) from src/engine/cue.ts
  - Update project state with returned cue
  - _Requirements: REQ-C1.1, REQ-C1.2, REQ-C3.1, REQ-C3.2, REQ-C3.3, REQ-C3.4, REQ-C3.5_

- [ ] 41. Implement recipe and identity application to empty layers
  - When applying a recipe or identity to an empty layer, create a new cue at the playhead position
  - Use default lead-in duration: 12 frames (0.5 seconds)
  - Use default tail duration: 12 frames (0.5 seconds)
  - If default lead-in or tail would fall outside scene, shift the cue so it fits and stays valid, including when playhead is at frame 0
  - Place hit marker at playhead frame
  - Apply recipe/identity curves and settings
  - _Requirements: REQ-D1.4_

- [ ] 42. Display hit position in frames and samples
  - Add UI component to show hit marker position
  - Format as "Frame {n} (Sample {n × 2000})"
  - Update when hit marker moves
  - _Requirements: REQ-C1.3_

### CHECKPOINT: Stage C Review

- [ ] 43. Checkpoint - Ensure UI and playback work
  - Ensure application shell displays correctly
  - Ensure video loads and plays
  - Ensure strips render and scroll in sync
  - Ensure drawing on strips updates curves
  - Ensure markers can be moved
  - Ensure playback engine synchronizes audio and video
  - After ten rapid play/stop presses no audio is left running
  - Audio stops within 100ms of pressing stop
  - Stop and wait for user

### Stage D: Layer Controls, Identity Management, Export, Visual Design, and First Full Deployment

- [ ] 44. Implement layer controls
  - Create `src/components/LayerControls.tsx`
  - Provide layer selection (up to 3 layers)
  - Provide mute toggle and level slider per layer
  - Provide "Clear cue" control per layer
  - Update project state on interaction
  - _Requirements: REQ-C2.1, REQ-C2.2, REQ-C2.3_

- [ ] 45. Implement identity shelf UI
  - Create `src/components/IdentityShelf.tsx`
  - Display list of saved identities by name (initially the three starter recipes)
  - Provide controls to save current cue as new identity
  - Provide controls to apply saved identity to selected layer's cue
  - Provide controls to copy identity under new name
  - Provide controls to delete identity
  - _Requirements: REQ-D1.1, REQ-D1.3, REQ-D2.1_

- [ ] 46. Implement identity application logic
  - Implement `applyIdentityToCue` function in `src/utils/identity.ts`
  - Copy identity's waveform, recipe settings, and default curves to cue
  - Fit pitch and intensity curves to cue's start, hit, and end frames (preserve x=0.5 at hit)
  - Store identity ID as provenance only
  - _Requirements: REQ-D1.4, REQ-D1.5_

- [ ] 47. Implement identity editing without affecting existing cues
  - When identity is edited and saved, update only that identity in shelf
  - Do NOT alter existing cues or snapshots until user explicitly reapplies identity
  - _Requirements: REQ-D2.2, REQ-D2.3_

- [ ] 48. Implement export UI with error handling
  - Add "Export Audio" button
  - Call renderScene for active scene
  - Pass mono buffer straight to encodeWAV (no separate stereo conversion step)
  - Call encodeWAV with 48000 Hz sample rate
  - Download WAV file with descriptive name (e.g., "ghostreel_scene1.wav")
  - On failure, show error message and leave project unchanged
  - When reductionDb is not zero, display the reduction value to the user
  - _Requirements: REQ-G1.1, REQ-G1.3, REQ-G1.4, REQ-G1.7, REQ-G1.8, REQ-B3.6_

- [ ] 49. Implement credit note
  - Display dismissible credit note naming Daphne Oram and Oramics
  - Link to [https://collection.sciencemuseumgroup.org.uk/objects/co8188395/oramics-machine](https://collection.sciencemuseumgroup.org.uk/objects/co8188395/oramics-machine)
  - Store dismissal in localStorage
  - _Requirements: REQ-A2.4_

- [ ] 50. Apply visual design direction
  - Apply visual direction from product.md across the application:
    - Film-strip treatment of strips (perforations, frame-like borders)
    - Restrained haunting palette (deep blacks, muted grays, subtle accent colors)
    - Readable labels and timeline with clear contrast
    - Consistent spacing and typography
  - Ensure design reinforces the Oramics resurrection concept
  - _Requirements: REQ-A1.7_

- [ ] 51. First full deployment
  - If user has replaced shot1.mp4 and shot2.mp4 with real footage, update scene durations and starter cue to fit
  - Deploy to GitHub Pages
  - Run the demonstration scenario on the live URL:
    1. Play the first shot and hear the starter cue
    2. Redraw the waveform, pitch, and intensity strips and hear the sound change
    3. Place the cue's hit marker on a chosen picture frame
    4. Save the sound's character as a named identity
    5. Export the scene audio
  - Verify all steps work on deployed build
  - _Requirements: REQ-H1.1, REQ-H1.2, REQ-H1.3_

### Stage E: Export Verification, A/B Comparison, Two Scenes, Save/Reopen, Undo

- [ ] 52. Implement export verification workflow
  - Add "Load Exported WAV" button
  - Parse WAV header to extract sample rate, channel count, duration
  - Display these values and warn if they differ from expected
  - Decode WAV using Web Audio decodeAudioData
  - Play decoded buffer synchronized with video
  - Mute live synthesis while playing exported file
  - Display hit marker positions on timeline for visual verification
  - Allow toggling between exported playback and live synthesis
  - _Requirements: REQ-G2.1, REQ-G2.2, REQ-G2.3, REQ-G2.4, REQ-G2.5_

- [ ] 53. Implement snapshot save and A/B comparison
  - Add "Save Snapshot" control to layer
  - Store snapshot in layer.snapshot
  - Add "Compare" toggle control
  - When comparing, switch playback and strip display between current cue and snapshot
  - User can play either version
  - Toggling compare on and off leaves current cue unchanged
  - _Requirements: REQ-E1.2, REQ-E1.3, REQ-E1.4_

- [ ] 54. Implement two-scene project support
  - Extend project to hold up to 2 scenes with activeSceneIndex
  - Add scene switcher UI control
  - When switching scenes, re-render the new scene's audio
  - Preserve identity shelf across scene switches
  - _Requirements: REQ-D3.1, REQ-D3.2, REQ-D3.3_

- [ ] 55. Implement project serialization
  - Implement `serializeProject(project: Project): string` in `src/utils/project.ts`
  - Convert project to JSON with version '1.0'
  - Return JSON string
  - _Requirements: REQ-F1.1_

- [ ] 56. Implement project deserialization with validation
  - Implement `deserializeProject(json: string): Project` in `src/utils/project.ts`
  - Parse JSON and check version === '1.0'
  - Allow projects with missing identity IDs (identityId is provenance only)
  - Clamp frame numbers to scene duration
  - Enforce cue validity: startFrame < hitFrame < endFrame with at least 1 frame margin
  - Clamp curve point x and y to valid ranges
  - Substitute safe defaults for missing fields
  - _Requirements: REQ-F1.2, REQ-F1.3_

- [ ] 57. Write property test for project serialization round trip
  - **Property 3: Project Serialization Round Trip**
  - **Validates: Requirements REQ-F1.1, REQ-F1.2, REQ-F1.3**
  - For any valid project state, serializing to JSON and then deserializing produces equivalent project
  - Generate random projects with fc.record
  - Compare identities, cues, markers, and settings
  - Run 100 iterations minimum
  - Tag: `// Feature: ghostreel-mvp, Property 3: Project Serialization Round Trip`
  - _Requirements: REQ-F1.1, REQ-F1.2, REQ-F1.3_

- [ ] 58. Implement project save and open UI
  - Add "Save Project" button that calls serializeProject and downloads JSON file
  - Add "Open Project" button that prompts for JSON file, calls deserializeProject, and loads result
  - For bundled videos, reload automatically
  - For user videos, prompt to re-select file and display expected fileName
  - _Requirements: REQ-F1.1, REQ-F1.2, REQ-F1.4, REQ-F1.5, REQ-F1.6_

- [ ] 59. Implement undo/redo
  - Create `src/utils/undo.ts`
  - On each edit action (draw, move marker, change level, apply identity, save snapshot, delete cue), deep-clone project state and push to undo stack
  - Maintain max 50 entries in undo stack
  - On undo: move current state to redo stack, pop from undo stack
  - On redo: move current state to undo stack, pop from redo stack
  - Do NOT push undo state for: play/pause, scrub, volume, mute toggle, comparison toggle
  - _Requirements: REQ-E1.1_

### Stage F: Property Tests and Polish

- [ ] 60. Write property test for layer independence
  - **Property 10: Layer Independence**
  - **Validates: Requirements REQ-C2.4, REQ-C3.6**
  - For any scene with multiple layers, editing one layer's cue leaves all other layers' cue data and settings unchanged
  - Generate scenes with multiple layers
  - Apply edits to one layer
  - Verify other layers unchanged
  - Run 100 iterations minimum
  - Tag: `// Feature: ghostreel-mvp, Property 10: Layer Independence`
  - _Requirements: REQ-C2.4, REQ-C3.6_

- [ ] 61. Write property test for identity application round trip
  - **Property 9: Identity Application Round Trip**
  - **Validates: Requirements REQ-D1.4, REQ-D2.1**
  - For any saved identity, applying it to a cue and then saving that cue's waveform as a new identity produces equivalent curve points
  - Generate random identities and cues
  - Compare waveform curves
  - Run 100 iterations minimum
  - Tag: `// Feature: ghostreel-mvp, Property 9: Identity Application Round Trip`
  - _Requirements: REQ-D1.4, REQ-D2.1_

- [ ] 62. Write property test for identity editing isolation
  - **Property 16: Identity Editing Does Not Affect Existing Cues**
  - **Validates: Requirements REQ-D2.3**
  - For any cue created from an identity, subsequently editing and re-saving that identity leaves the existing cue's curves unchanged
  - Generate identity, create cue from it, edit identity, verify cue unchanged
  - Run 100 iterations minimum
  - Tag: `// Feature: ghostreel-mvp, Property 16: Identity Editing Does Not Affect Existing Cues`
  - _Requirements: REQ-D2.3_

- [ ] 63. Implement first-run tour
  - Check sessionStorage.getItem('ghostreel_tour_seen')
  - If null, display dismissible 3-4 step tour
  - Tour points out: parallel strips, hit marker, export control
  - When dismissed, set sessionStorage flag
  - _Requirements: REQ-A2.1, REQ-A2.2, REQ-A2.3_

- [ ] 64. Implement unsupported browser detection
  - Feature check: AudioContext, requestAnimationFrame, Float32Array, video MP4 support, OfflineAudioContext
  - If any missing, show: "GhostReel requires a modern browser. Tested on current Chrome for Windows and macOS."
  - _Requirements: REQ-H1.5_

- [ ] 65. Add plain-language labels and accessibility
  - Label all controls with descriptive text
  - Ensure no state is shown by color alone
  - Add ARIA labels where appropriate
  - _Requirements: REQ-B2.3_

### Stage G: Final Deployment

- [ ] 66. Final deployment and verification
  - Add asset credits file documenting all external assets used
  - README lists any requirement not implemented in final build
  - Add README sections for:
    - Performance measurements
    - System limits (60 seconds, 24 fps, 3 layers)
    - Browser compatibility
  - Deploy final build to GitHub Pages
  - Confirm deployed build loads without credentials
  - Test complete demonstration scenario on deployed build
  - Test on Chrome Windows and Chrome macOS
  - Verify all requirements from demo scenario work end-to-end
  - Document deployment URL in README
  - _Requirements: REQ-H1.1, REQ-H1.2, REQ-H1.3, REQ-H1.4_

- [ ] 67. Final checkpoint - Ensure all tests pass
  - Ensure all unit tests pass
  - Ensure all property tests pass
  - Ensure render tests pass
  - Ensure deployed build completes demonstration scenario
  - Ask the user if questions arise.

## Notes

- All tasks are required (no optional tasks in this plan)
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation at key milestones
- Property tests validate universal correctness properties across randomized inputs
- The pure renderScene function is testable in Node without browser dependencies
- Stage C includes explicit stop-and-wait checkpoint for user review

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1", "2", "3"] },
    { "id": 1, "tasks": ["4"] },
    { "id": 2, "tasks": ["5"] },
    { "id": 3, "tasks": ["6", "7"] },
    { "id": 4, "tasks": ["8"] },
    { "id": 5, "tasks": ["9", "10"] },
    { "id": 6, "tasks": ["11"] },
    { "id": 7, "tasks": ["12"] },
    { "id": 8, "tasks": ["13"] },
    { "id": 9, "tasks": ["14"] },
    { "id": 10, "tasks": ["15"] },
    { "id": 11, "tasks": ["16", "17", "18"] },
    { "id": 12, "tasks": ["19"] },
    { "id": 13, "tasks": ["20"] },
    { "id": 14, "tasks": ["21"] },
    { "id": 15, "tasks": ["22"] },
    { "id": 16, "tasks": ["23", "24", "25"] },
    { "id": 17, "tasks": ["26"] },
    { "id": 18, "tasks": ["27"] },
    { "id": 19, "tasks": ["28"] },
    { "id": 20, "tasks": ["29"] },
    { "id": 21, "tasks": ["30"] },
    { "id": 22, "tasks": ["31"] },
    { "id": 23, "tasks": ["32"] },
    { "id": 24, "tasks": ["33"] },
    { "id": 25, "tasks": ["34", "35"] },
    { "id": 26, "tasks": ["36", "37"] },
    { "id": 27, "tasks": ["38"] },
    { "id": 28, "tasks": ["39"] },
    { "id": 29, "tasks": ["40", "41"] },
    { "id": 30, "tasks": ["42"] },
    { "id": 31, "tasks": ["43"] },
    { "id": 32, "tasks": ["44"] },
    { "id": 33, "tasks": ["45"] },
    { "id": 34, "tasks": ["46"] },
    { "id": 35, "tasks": ["47"] },
    { "id": 36, "tasks": ["48"] },
    { "id": 37, "tasks": ["49", "50"] },
    { "id": 38, "tasks": ["51"] },
    { "id": 39, "tasks": ["52"] },
    { "id": 40, "tasks": ["53"] },
    { "id": 41, "tasks": ["54"] },
    { "id": 42, "tasks": ["55"] },
    { "id": 43, "tasks": ["56"] },
    { "id": 44, "tasks": ["57"] },
    { "id": 45, "tasks": ["58"] },
    { "id": 46, "tasks": ["59"] },
    { "id": 47, "tasks": ["60", "61", "62"] },
    { "id": 48, "tasks": ["63", "64", "65"] },
    { "id": 49, "tasks": ["66"] },
    { "id": 50, "tasks": ["67"] }
  ]
}
```
