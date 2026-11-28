# Product: GhostReel

## Target User

**Independent picture editor** handling sound on horror or genre short films. The director requests and approves creative changes. An experienced sound designer can review output quality.

Horror is the first audience because designed, non-realistic sound carries the genre: a presence, a machine, or a threat needs a signature that the audience learns and then dreads.

## Job to Be Done

Given a short reference scene, **create a designed sound cue by drawing it**, **synchronize its important moment to a picture frame**, **revise one aspect without losing that sync**, and **export audio** the editor can drop into an existing project.

## Value Hypothesis

Understandable drawn controls and saved sound identities may make this task easier for the intended editor. This is a hypothesis to test. No customer adoption, time-saving percentage, industry superiority, or revenue has been established.

## The Resurrection Concept

Daphne Oram's Oramics machine generated sound from painted waveforms and controlled parameters using marks on parallel film strips. GhostReel preserves the meaningful relationship between drawn marks and generated sound.

**At least one user-editable waveform must actually shape generated audio.** Pitch and intensity curves must directly control its behavior through time. A decorative drawing over a prerecorded sound does not satisfy the intended concept.

## The Interface IS the Resurrection

A first-time viewer should recognize the revived mechanism within a few seconds, without reading an explanation:

- The main workspace shows the reference picture above a set of **parallel film strips** that scroll together under a single fixed playhead
- Each sound layer has three strips: **waveform**, **pitch**, and **intensity**. The user draws directly on the strips
- The strips are locked to the picture timeline. Scrubbing the picture moves the strips; the playhead position is the same moment in both
- Markers such as cue start, hit, and end appear on the strips and on the picture timeline together
- A short, dismissible note credits Daphne Oram and Oramics and links to the Science Museum record

## Demonstration Scenario

Use two short original shots involving the same fictional presence (e.g., a corridor where a light fails as something approaches, and a second shot where it returns).

The app opens with the first shot loaded and a starter cue already drawn, so the first click produces sound:

1. Play the first shot and hear the starter cue
2. Redraw the waveform, pitch, and intensity strips and hear the sound change
3. Place the cue's hit marker on a chosen picture frame
4. Save the sound's character as a named identity (e.g., "The Presence")
5. Save a snapshot of the cue
6. Apply this director note: "Begin the build six frames earlier. Keep the hit where it is. Soften the ending."
7. Compare the snapshot with the revised cue. Other layers keep their settings
8. Open the second shot and use the same identity with a different duration or intensity
9. Export the scene audio, then play the exported file under the picture inside GhostReel to confirm the hit lands on its frame

For a 24 fps reference, six frames equals 0.25 seconds. At 48 kHz that is 12,000 samples, or 2,000 samples per frame.

## MVP Boundaries

- Desktop browser application, served from a public URL with no login
- Tested on current Chrome on Windows and macOS
- One reference video open at a time, up to 60 seconds
- One explicitly supported constant-frame-rate format: 24 fps, confirmed by the user
- Up to three synthesized sound layers, one cue per layer
- Three carefully tuned starting recipes: tension pulse, energy build and release, stylized presence
- Local video playback and audio processing. No server-side processing and no account
- Scene-by-scene work

## Credit

Link to Science Museum: [Oramics Machine](https://collection.sciencemuseumgroup.org.uk/objects/co8188395/oramics-machine)
