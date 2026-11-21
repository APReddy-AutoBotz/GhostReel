# Requirements Document

## Introduction

GhostReel is a browser-based sound design application that revives Daphne Oram's Oramics concept for independent filmmakers. Users draw sound on parallel film strips locked to picture, place synthesized cues at precise frames, save sound identities for reuse across shots, and export synchronized audio.

The target user is an independent picture editor handling sound on horror or genre short films, working scene by scene with clips up to 60 seconds at 24 fps.

## Glossary

- **GhostReel**: The browser application that generates and synchronizes drawn sound to video
- **Scene**: A single video clip up to 60 seconds at 24 fps loaded as the reference picture
- **Strip**: One of three drawable parallel tracks (waveform, pitch, intensity) that scroll with the picture timeline
- **Layer**: A group of three strips (waveform, pitch, intensity) that together define one synthesized sound
- **Cue**: A placed sound event on a layer with defined start, hit, and end positions
- **Hit**: The named important moment within a cue that must land on a chosen picture frame
- **Identity**: A saved sound recipe (waveform shape, pitch and intensity curves) that can be reused across scenes with different timing
- **Snapshot**: A saved version of a complete cue (identity plus placement and performance) for comparison
- **Project**: The saveable work containing the scene reference, identities, cues, and snapshots
- **Playhead**: The fixed vertical line under which strips scroll during playback, showing the current timeline position

## Requirements

### REQ-A1: Load and Play Scene

**User Story:** As an editor, I want to open a reference clip and hear starter sound immediately, so that I can begin work without setup delay.

#### Acceptance Criteria

1. WHEN GhostReel loads for the first time, THE GhostReel SHALL display a bundled demonstration shot with a starter cue already drawn on the strips
2. WHEN the user clicks play, THE GhostReel SHALL play the scene video and synthesized audio together
3. WHEN the user selects "Open Video", THE GhostReel SHALL prompt for a local video file and load it as the reference scene
4. WHEN a video file is loaded, THE GhostReel SHALL require the user to confirm the frame rate as 24 fps before proceeding
5. IF the selected file cannot be decoded or exceeds 60 seconds, THEN THE GhostReel SHALL display a descriptive error message and retain the previous scene
6. WHEN the user navigates the timeline, THE GhostReel SHALL move the playhead position and scroll the strips to the corresponding time

### REQ-A2: Guided Introduction

**User Story:** As a first-time user, I want to understand the key interface elements quickly, so that I can start editing without reading documentation.

#### Acceptance Criteria

1. WHEN GhostReel loads for the first time in a browser session, THE GhostReel SHALL display a skippable tour of three to four steps
2. THE tour SHALL point out the parallel strips, the hit marker, and the export control
3. WHEN the user dismisses the tour, THE GhostReel SHALL not show it again in that browser session
4. THE GhostReel SHALL display a dismissible credit note naming Daphne Oram and Oramics with a link to the Science Museum record

### REQ-B1: Draw Sound on Strips

**User Story:** As an editor, I want to draw waveform, pitch, and intensity by hand, so that I can shape the character of a sound.

#### Acceptance Criteria

1. THE GhostReel SHALL provide three editable strips per layer: waveform, pitch, and intensity
2. WHEN the user draws on a waveform strip, THE GhostReel SHALL use that shape as the audio source for that layer
3. WHEN the user draws on a pitch strip, THE GhostReel SHALL control the frequency of the waveform over time according to that curve
4. WHEN the user draws on an intensity strip, THE GhostReel SHALL control the amplitude of the output over time according to that curve
5. WHEN the user plays the scene, THE GhostReel SHALL synthesize audio from the drawn strips in real time
6. THE GhostReel SHALL support editing existing points on the strips and undoing drawing actions

### REQ-B2: Usable Starting Recipes

**User Story:** As an editor unfamiliar with synthesis, I want understandable starting sounds, so that I can make useful edits without learning oscillator theory.

#### Acceptance Criteria

1. THE GhostReel SHALL provide three named starter recipes: tension pulse, energy build and release, and stylized presence
2. WHEN a recipe is selected, THE GhostReel SHALL populate the strips with waveform, pitch, and intensity curves that produce recognizable sound
3. THE recipe curves SHALL use descriptive parameter ranges (for example, frequency in Hz) rather than raw synthesis values

### REQ-B3: Safe Audio Output

**User Story:** As an editor, I want controlled playback volume, so that I can audition sounds without hearing damage or distortion.

#### Acceptance Criteria

1. THE GhostReel SHALL start with conservative initial gain that prevents clipping for the starter recipes
2. THE GhostReel SHALL provide a visible stop control and a labeled volume control
3. WHEN transitioning between cues or stopping playback, THE GhostReel SHALL apply fade envelopes that prevent audible clicks
4. IF a drawn strip produces invalid parameter values, THEN THE GhostReel SHALL clamp or substitute safe values without producing unstable audio output

### REQ-C1: Place Cues with Frame Precision

**User Story:** As an editor, I want to mark a cue's hit moment on a specific picture frame, so that the sound event lands where the director intended.

#### Acceptance Criteria

1. THE GhostReel SHALL allow the user to place a cue start marker, a named hit marker, and an end marker on each layer's timeline
2. WHEN a hit marker is placed, THE GhostReel SHALL record its position in frames relative to the scene start
3. THE GhostReel SHALL display the hit position in both frames and samples (at 48 kHz, 2000 samples per frame at 24 fps)
4. WHEN the cue plays, THE audible sound event SHALL align with the hit marker position within the scene timeline
5. THE GhostReel SHALL use a consistent time origin at the scene start (frame 0, time 0.0s, sample 0)

### REQ-C2: Independent Layer Control

**User Story:** As an editor, I want to adjust each sound layer separately, so that I can balance and revise one element without affecting others.

#### Acceptance Criteria

1. THE GhostReel SHALL support up to three layers, each with one cue
2. THE GhostReel SHALL allow independent selection of each layer
3. THE GhostReel SHALL provide mute and level controls for each layer
4. WHEN one layer is edited, THE GhostReel SHALL leave all other layers' cue data, markers, and settings unchanged
5. WHEN layers are played together, THE GhostReel SHALL mix their outputs

### REQ-C3: Revise Lead-In While Preserving Hit

**User Story:** As an editor, I want to change how early a cue begins without moving its hit, so that I can follow director notes like "begin the build six frames earlier, keep the hit where it is."

#### Acceptance Criteria

1. WHEN the user adjusts a cue's start marker position, THE GhostReel SHALL move the start marker while keeping the hit marker at its original frame
2. IF moving the start marker earlier would place it before the scene start, THEN THE GhostReel SHALL clamp the start marker at frame 0 and report the constraint
3. WHEN the start marker is adjusted, THE GhostReel SHALL preserve the cue's waveform, pitch, intensity, end marker, and all settings on other layers

### REQ-D1: Save and Reuse Sound Identities

**User Story:** As an editor, I want to save a designed sound's character and use it in a different shot, so that the same presence or threat has a consistent signature across scenes.

#### Acceptance Criteria

1. THE GhostReel SHALL allow the user to save the current strips (waveform, pitch, intensity) as a named identity
2. WHEN an identity is saved, THE GhostReel SHALL store the parameters needed to reproduce that sound design
3. THE GhostReel SHALL provide an identity shelf that lists saved identities by name
4. WHEN the user applies a saved identity to a layer, THE GhostReel SHALL load that identity's waveform, pitch, and intensity curves
5. WHEN an identity is reused, THE GhostReel SHALL allow the user to place its cue with different start, hit, and end positions than the original

### REQ-D2: Vary Performance Without Changing Identity

**User Story:** As an editor, I want to make a variation on a saved identity, so that I can create related sounds without affecting the original.

#### Acceptance Criteria

1. THE GhostReel SHALL allow the user to copy an existing identity under a new name
2. WHEN an identity is edited and saved again, THE GhostReel SHALL update only that identity
3. WHEN an identity is edited, THE GhostReel SHALL not alter previously saved cue snapshots or other identities

### REQ-D3: Project Scope for Identities and Scenes

**User Story:** As an editor, I want to organize my sound identities and scenes together, so that I can work on related shots without losing my designs.

#### Acceptance Criteria

1. THE GhostReel SHALL maintain an identity shelf and scene references within a single project
2. THE GhostReel SHALL allow the user to switch between up to two scenes within the same project while retaining the identity shelf
3. WHEN a different scene is opened, THE GhostReel SHALL preserve all saved identities and make them available for use in the new scene

### REQ-E1: Undo and Compare

**User Story:** As an editor, I want to undo mistakes and compare my current work to a previous version, so that I can experiment without losing good takes.

#### Acceptance Criteria

1. THE GhostReel SHALL provide undo and redo controls for strip drawing and marker placement
2. THE GhostReel SHALL allow the user to save a snapshot of the current cue state on a layer
3. THE GhostReel SHALL provide a comparison view that shows the current cue alongside one saved snapshot
4. WHEN comparing, THE GhostReel SHALL allow the user to play either version

### REQ-F1: Save and Reopen Projects

**User Story:** As an editor, I want to save my work and return to it later, so that I can work across multiple sessions.

#### Acceptance Criteria

1. THE GhostReel SHALL allow the user to download an editable project file
2. THE GhostReel SHALL allow the user to open a previously saved project file
3. WHEN a project is reopened, THE GhostReel SHALL restore all cue positions, drawing data, identities, the saved snapshot, and layer levels
4. WHEN a project references a bundled demonstration shot, THE GhostReel SHALL reload that shot automatically
5. IF a project references a user-provided video file, THEN THE GhostReel SHALL prompt the user to re-select the file and display its expected name
6. THE project file format SHALL not depend solely on browser local storage

### REQ-G1: Export Scene Audio

**User Story:** As an editor, I want to export a synchronized audio file, so that I can import it into my editing application at the scene start.

#### Acceptance Criteria

1. THE GhostReel SHALL export a stereo audio file containing the mixed output of all unmuted layers
2. THE exported file SHALL start at the scene's time origin (frame 0, sample 0)
3. THE exported file SHALL have a duration equal to the scene duration plus any cue decay that fits within a defined range
4. THE exported file SHALL use 48 kHz sample rate
5. THE exported file format SHALL be PCM WAV
6. WHEN a cue's end or decay extends beyond the scene duration, THE GhostReel SHALL render the decay if it fits within the maximum export range, or define and document the truncation behavior

### REQ-G2: Verify Sync Inside GhostReel

**User Story:** As an editor, I want to confirm that my exported audio is synchronized before importing it elsewhere, so that I can catch timing errors early.

#### Acceptance Criteria

1. THE GhostReel SHALL allow the user to load a previously exported WAV file as an audio track under the reference picture
2. WHEN an exported file is loaded and played, THE GhostReel SHALL play it synchronized with the video
3. THE GhostReel SHALL display the hit marker positions on the timeline
4. THE GhostReel SHALL report each hit's position in frames and samples
5. WHEN the user plays the imported audio, THE audible hit and the visual hit marker SHALL coincide

### REQ-H1: Browser Deployment

**User Story:** As a judge or user, I want to access GhostReel from a public URL without installing software, so that I can try it immediately.

#### Acceptance Criteria

1. THE GhostReel SHALL be deployed as static files to a public URL
2. THE GhostReel SHALL load and run without requiring user credentials or login
3. THE GhostReel SHALL remain available and free to use until 29 January 2027
4. THE GhostReel SHALL be tested and confirmed working on current Chrome on Windows and macOS
5. IF GhostReel is accessed from an unsupported browser, THEN THE GhostReel SHALL display a clear message identifying tested browsers

## Out of Scope

The following capabilities are explicitly excluded from this MVP and must not be implemented:

- Imported WAV sample layers and sample editing
- Media relinking workflows beyond re-selecting the reference video
- Written sound-effects cue lists
- Individual cue WAV export (scene mix only)
- Separate aligned layer exports
- Automatic frame rate detection or support for frame rates other than 24 fps
- Support for clips longer than 60 seconds
- More than three sound layers
- Native editor plugins, AAF/OMF exchange, or direct integration with editing applications
- Automatic soundtrack generation from entire videos
- Dialogue restoration, ADR, dubbing, music composition, or final surround mixing
- General-purpose realistic Foley synthesis
- Team accounts, payments, cloud asset management, or collaboration features
- Chat assistants, autonomous production agents, or runtime LLM dependencies
- Native mobile applications
- Server-side audio processing
- Full version history (only one snapshot comparison)

## Assumptions

- The browser's video element provides sufficient seek accuracy for scene-by-scene work; we are not claiming frame-accurate professional playback.
- Generated synthesis output is repeatable within the same browser session and rendering context, but not promised bit-identical across different browsers or versions.
- Users will confirm the exported sync by importing the WAV into their editor at the scene start and checking the hit frame.
- The three starter recipes will be tuned during implementation to produce recognizable, editable sounds.
- Conservative gain and click prevention are sufficient for safe auditioning; no additional limiting or metering is provided.
- Local video decode and synthesis for clips up to 60 seconds at 24 fps will perform acceptably on target hardware (current laptop, current Chrome).

## Open Questions

1. **Project structure for multiple scenes:** Should the project support two scenes with a shared identity shelf (preferred per brief), or should users duplicate a project file and replace the video reference? **Recommendation:** Implement two scenes per project if it does not complicate the data model; otherwise use project duplication and document the workflow.

2. **Bit depth and channel behavior for WAV export:** What bit depth should be used (16-bit, 24-bit, 32-bit float), and how should stereo channels be populated (duplicate mono, true stereo, or configurable)? **Recommendation:** Use 16-bit PCM with duplicate mono (same signal on L and R) as the simplest starting point; document the choice.

3. **Decay truncation at scene end:** When a cue's decay extends past the scene duration, should the export (a) truncate exactly at scene end, (b) extend to include decay up to a fixed maximum (e.g., +2 seconds), or (c) include the full decay regardless of length? **Recommendation:** Option (b) with a 2-second post-scene grace period, documented clearly.

4. **Waveform drawing interpretation:** Should the drawn waveform be interpreted as (a) a single-cycle repeating waveform, (b) a wavetable with interpolation, or (c) direct sample data? **Recommendation:** Single-cycle repeating waveform (option a) as the simplest meaningful revival of the Oramics concept.

5. **Strip visual resolution and edit density:** What is the minimum and maximum number of editable points per strip, and how does zoom or scene length affect editing precision? **Recommendation:** Start with a fixed reasonable resolution (e.g., 512 points per strip), test during implementation, and document any length-dependent limits discovered.

## Schedule Fit

This scope fits the schedule in section 12 of the brief if implementation focuses on the audio engine and strip-to-picture lock first (21-22 Nov), then the interface and hit anchor (23-26 Nov), then save/reopen and deployment (27-29 Nov), leaving 30 Nov-1 Dec for polish and user sessions.

**If the schedule is too tight, cut in this order:**

1. **Identity shelf with two scenes (REQ-D3):** Fall back to project duplication workflow. Users copy the project file and load a different video. This removes multi-scene management complexity.

2. **Snapshot comparison (REQ-E1, comparison part):** Keep undo/redo, but remove the side-by-side comparison view. Users rely on undo to return to previous states.

3. **Third layer (REQ-C2):** Reduce maximum layers from three to two. The core workflow (draw, place, reuse, export) still works with fewer simultaneous sounds.

The core cannot be reduced further without abandoning the intended user outcome. REQ-A1, REQ-B1, REQ-C1, REQ-C3, REQ-D1, REQ-G1, REQ-G2, and REQ-H1 are essential.
