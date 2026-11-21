# GhostReel: product brief for Kiro

**Prepared:** 21 November 2026 (revision 2; replaces the earlier brief of the same date)  
**Status:** Proposed MVP input for requirements review; implementation has not started.  
**Working name:** GhostReel  
**Target category:** Kiroween 2026, Resurrection  
**Initial spec name:** ghostreel-mvp

## 1. Purpose of this document

Use this brief to create a standard Kiro Feature Spec with the Requirements-First workflow. Generate the formal requirements first and pause for product review. After that review, develop the technical design and implementation tasks through Kiro's normal spec phases.

This brief records the intended user outcome and proposed scope. Kiro should identify conflicts, expose uncertain assumptions, and recommend simpler approaches when necessary. Technical preferences below are starting proposals, not claims that an implementation has been verified.

Maintain one formal requirements document in the spec folder. Update dependent design and tasks when approved requirements change.

### What changed in this revision

The scope is smaller and the product is more focused. The drawn film strips are now the main interface, the first audience is horror and genre shorts, sync is verified inside the app, and hosting, repository, video, and Kiro-evidence requirements from the official rules are written in. Section 6 lists what was removed.

## 2. Product proposition and users

**GhostReel lets independent filmmakers draw the sound of the thing that haunts their film, lock it to picture, and reuse it from shot to shot.**

Target industry: film, television, animation, and digital storytelling.

First user: an independent picture editor who also handles sound on horror or genre short films. The director requests and approves creative changes. An experienced sound designer can review output quality.

Horror is the first audience because designed, non-realistic sound carries the genre: a presence, a machine, or a threat needs a signature that the audience learns and then dreads. The same workflow applies to science fiction, fantasy, and animation.

### Job to be done

Given a short reference scene, create a designed sound cue by drawing it, synchronize its important moment to a picture frame, revise one aspect without losing that sync, and export audio the editor can drop into an existing project.

### Value hypothesis

Understandable drawn controls and saved sound identities may make this task easier for the intended editor. This is a hypothesis to test. No customer adoption, time-saving percentage, industry superiority, or revenue has been established.

## 3. The technology being resurrected

Daphne Oram's Oramics machine generated sound from painted waveforms and controlled parameters using marks on parallel film strips. GhostReel should preserve the meaningful relationship between drawn marks and generated sound. See the Science Museum source below.

At least one user-editable waveform must actually shape generated audio. Pitch and intensity curves must directly control its behavior through time. A decorative drawing over a prerecorded sound or an unrelated generated clip does not satisfy the intended concept.

Existing tools already support visual synthesis and modern Oramics interpretations. GhostReel should credit the historical inspiration and describe its own filmmaking workflow without claiming to be the first drawing synthesizer or the first Oramics revival. The new problem it applies the mechanism to is frame-synchronized sound design for picture.

## 4. The interface is the resurrection

A first-time viewer should recognize the revived mechanism within a few seconds, without reading an explanation.

- The main workspace shows the reference picture above a set of **parallel film strips** that scroll together under a single fixed playhead.
- Each sound layer has three strips: **waveform**, **pitch**, and **intensity**. The user draws directly on the strips.
- The strips are locked to the picture timeline. Scrubbing the picture moves the strips; the playhead position is the same moment in both.
- Markers such as cue start, hit, and end appear on the strips and on the picture timeline together.
- A short, dismissible note credits Daphne Oram and Oramics and links to the Science Museum record.

Visual direction: a polished creative instrument with a restrained haunting atmosphere. Film-strip texture and sprocket detail are welcome where they aid recognition. Labels, contrast, timeline readability, and audio controls take priority over decoration. Color alone must not communicate state.

## 5. The demonstration scenario

Use two short original shots involving the same fictional presence, for example a corridor where a light fails as something approaches, and a second shot where it returns. All footage must be original or properly licensed, with permissions recorded.

The app opens with the first shot loaded and a starter cue already drawn, so the first click produces sound.

1. Play the first shot and hear the starter cue.
2. Redraw the waveform, pitch, and intensity strips and hear the sound change.
3. Place the cue's hit marker on a chosen picture frame.
4. Save the sound's character as a named identity, for example "The Presence".
5. Save a snapshot of the cue.
6. Apply this director note: "Begin the build six frames earlier. Keep the hit where it is. Soften the ending."
7. Compare the snapshot with the revised cue. Other layers keep their settings.
8. Open the second shot and use the same identity with a different duration or intensity.
9. Export the scene audio, then play the exported file under the picture inside GhostReel to confirm the hit lands on its frame.

For a 24 fps reference, six frames equals 0.25 seconds. At 48 kHz that is 12,000 samples, or 2,000 samples per frame. This is an illustrative revision test, not a claim about an already working product.

## 6. Proposed MVP boundaries

These limits keep the first version small enough to finish, polish, and evaluate in the time available. A narrow finished product is preferred over a broad proof of concept. Flag a limit if it prevents the core user outcome.

- Desktop browser application, served from a public URL with no login.
- Tested on current Chrome on Windows and macOS. Other browsers either work or show a clear message; do not claim support that has not been tested.
- One reference video open at a time, up to 60 seconds.
- One explicitly supported constant-frame-rate format, initially 24 fps, confirmed by the user. Do not claim automatic detection or universal video support unless implemented.
- Up to three synthesized sound layers, one cue per layer.
- Three carefully tuned starting recipes: a tension pulse, an energy build and release, and a stylized presence.
- Local video playback and audio processing. No server-side processing and no account.
- Scene-by-scene work.

### Removed from the earlier brief

Keep these out of the first implementation plan:

- Imported WAV sample layers and all sample editing.
- A media relinking workflow beyond re-selecting the reference video when a project is reopened.
- The written sound-effects cue list.
- Individual cue WAV export.
- Separate aligned layer exports. Treat these as the first stretch item, to add only after the core flow is finished and polished.
- A DaVinci Resolve import step inside the demonstration. See capability G for how this check is handled.

### Required user capabilities

#### A. Open a scene and hear something immediately

On first load, show the bundled first shot with a starter cue already drawn on the strips. Audio starts only after a user action, as browsers require. A short, skippable guided tour of three or four steps points out the strips, the hit marker, and export.

Let the user open their own local reference clip, play and pause it, navigate its timeline, and confirm the scene's frame rate and time origin. Show a clear failure message for an unsupported or unreadable file.

Record asset permissions and attribution for the bundled shots in the repository.

#### B. Draw and hear sound

Provide editable waveform, pitch, and intensity strips as described in section 4. Start from understandable recipes so a new user can make a useful edit without learning oscillator terminology.

Audition changes against the scene. Provide conservative initial gain, a visible stop control, and a usable volume control. Avoid clicks at cue boundaries and handle invalid control values without producing unstable output.

Drawing supports editing points and recovering from mistakes. Essential playback and editing controls have clear labels.

#### C. Place and layer cues

Support a cue start, a named important moment such as a hit, and an end or decay. Use a consistent timeline origin and explicit timing units.

Allow independent selection, auditioning, mute, and level adjustment for each layer.

A revision must be able to change a cue's lead-in while preserving its chosen hit position. Define behavior when a requested edit exceeds the scene boundary.

#### D. Preserve an identity while varying a performance

Separate the base sound recipe from its placement and time-varying performance.

Save a named identity, reuse it in the second shot, and make a variation deliberately by copying it under a new name. Reusing or editing an identity must not silently change previously saved cue snapshots.

Preferred approach: a project holds a small identity shelf and up to two scenes, with one scene open at a time. If that complicates the data model, the fallback is duplicating a project and replacing its reference video while retaining identities. Kiro should recommend one.

Preserve the sound parameters needed to reproduce a saved design. If randomized synthesis is used, choose a repeatable approach and document its limits.

#### E. Revise, compare, and recover

Provide undo and redo for core editing, and a simple comparison between the current cue and one saved snapshot. A full version-history interface is outside this MVP.

Changing one cue must leave unrelated cue data and settings unchanged. Do not promise bit-identical output across different browsers or renderer versions without evidence.

#### F. Save and reopen

Download an editable project file and reopen it in a fresh session.

Preserve cue positions, drawing data, identities, the saved snapshot, and levels. Bundled shots are referenced by name. A user's own reference video is re-selected on reopen, with a clear prompt naming the missing file. Saving must not depend solely on browser storage.

#### G. Deliver usable audio and prove the sync

Export:

- A stereo scene mix that starts at the scene's time origin and lasts the scene's duration.
- The editable project file.

A proposed output is 48 kHz PCM WAV. The design should specify and verify the actual bit depth and channel behavior. These settings are a project choice, not a claim of compliance with every film-delivery standard.

Preserve a cue's decay when the scene duration permits it; define what happens at the end of the range.

**In-app sync check.** After export, the user can load the exported WAV back under the reference picture and play it. The hit marker and the audible hit should coincide, and the app should report the hit's position in frames and samples. This replaces the earlier in-demo Resolve step and lets a judge verify sync without other software.

**Off-video editor check.** Before submission, import the exported WAV into an editing application at the scene start and confirm the hit lands on the intended frame. Record the result, the application, and its version in the README. Report only what was observed.

#### H. Run where the judges are

Deploy the app as static files to a public host. It must load without credentials and remain available and free to use until the judging period ends on 29 January 2027. Confirm the deployed build, not just the local one, completes the demonstration scenario.

## 7. Later opportunities

Keep these outside the initial implementation plan unless a separately reviewed change makes them necessary:

- Everything listed under "Removed from the earlier brief" in section 6.
- Full-length film timelines and automatic reconforming after picture edits.
- Automatic soundtrack generation from an entire video.
- Dialogue restoration, ADR, dubbing, music composition, or final surround mixing.
- General-purpose realistic Foley synthesis.
- Team accounts, payments, cloud asset management, marketplaces, or collaboration suites.
- Native editor plugins, AAF/OMF exchange, or integrations with editing applications.
- Chat assistants, autonomous production agents, and runtime LLM dependencies.
- Native mobile applications.

## 8. Technical design questions Kiro should resolve

Preferred starting stack: React, TypeScript, Vite, and browser audio capabilities. Choose compatible supported dependencies at implementation time and explain any material addition. No server is expected; justify one only if a required capability needs it.

The design should answer:

1. How will editable waveform shapes become suitable audio sources, with sensible frequency limits and artifact control?
2. How will drawn pitch and intensity paths control live playback and offline export consistently?
3. How will scene frames, timeline time, and output audio samples map to one another?
4. How will the strips stay visually locked to the picture during playback and scrubbing, and what browser video-navigation limits remain?
5. How can a lead-in change while its hit anchor stays fixed?
6. How are identities, performance curves, cue snapshots, and scenes represented and reopened?
7. How will repeated play and stop operations, cancelled loads, and failed exports recover?
8. What short-clip memory and export-duration limits will be measured on the target laptop and browser?

Precise stored timing and a responsive browser preview are different concerns. Do not equate an ordinary video seek with a verified professional frame-accurate viewer.

Prototype the highest-risk audio and timing behavior first. The first working increment should let us draw a simple sound, hear it, and export a valid WAV. Add the reference picture and the strip-to-picture lock immediately afterward, before broadening the interface.

## 9. Acceptance evidence to plan

Translate the user capabilities into stable requirement IDs and testable acceptance criteria. Prefer tests of behavior and real failures over tests that repeat implementation details.

### Engineering checks

- Drawing changes produce the intended audible parameter change.
- Timeline-to-audio conversion places an exported cue at the expected sample offset.
- Changing a lead-in preserves the selected hit marker and unrelated cue settings.
- Save and reopen preserves the design, or names exactly which video needs re-selecting.
- Exported files have the promised format, duration, and channel count.
- Play, stop, replay, and error recovery do not leave unintended audio running.

### Property-based tests

Propose property-based tests where they catch meaningful failure classes. Likely candidates:

- Frame, time, and sample conversions round-trip without drift across the supported range.
- For any lead-in change within the scene boundary, the hit position is unchanged.
- Any valid project serializes and deserializes to an equivalent project.
- Any rendered buffer encodes to a WAV whose header matches its contents.

Record what these tests actually caught.

### Human evaluation

By 30 November, try the workflow with two or three independent filmmakers or editors, and one sound practitioner if available. Observe task completion, revision time, sound quality, and whether they would keep the output in a scene.

Report only results actually observed. Quote a participant only with their permission.

## 10. Submission requirements from the official rules

Verify the current official rules before submission. As checked on 21 November 2026:

- **Deadline:** 4 December 2026 at 14:00 PST, which is 5 December 2026 at 03:30 IST. No extensions. Target submitting on 3 December.
- **New work only:** all project code is created on or after 30 October 2026. Disclose any pre-existing work other than standard frameworks and libraries.
- **Repository:** public on GitHub, with all source, assets, and setup instructions; an open source LICENSE file at the root that GitHub detects and shows in the About section (MIT proposed); and the `/.kiro` directory at the root, never listed in `.gitignore`.
- **Working app URL:** see capability H.
- **Video:** under three minutes, public on YouTube, showing the app running in a desktop browser. No third-party trademarks, product interfaces, or copyrighted music or footage. All picture and sound in the video are original.
- **Text description** of features, and a **Kiro usage write-up**.
- All materials in English.

### Video outline

Plan for about two minutes forty seconds.

1. Ten seconds: Daphne Oram drew sound on film in the 1960s. GhostReel brings that back for filmmakers.
2. Thirty seconds: draw on the strips and hear the sound change.
3. Forty seconds: place the hit on a frame, then apply the director note and compare.
4. Thirty seconds: reuse the identity in the second shot.
5. Twenty seconds: export and run the in-app sync check.
6. Thirty seconds: how Kiro was used, with one concrete result.

## 11. Kiro evidence

Judges score how thoroughly and skillfully Kiro was used and whether its tools were explored. Choose features because they improve the work, and document observed results rather than retroactively inventing usage evidence.

Planned usage, to confirm or drop based on real value:

- **Specs:** this Requirements-First spec, with Analyze Requirements run before design. Use a Bugfix Spec for the first substantial defect.
- **Steering:** product, technology, and structure files, plus an audio-and-timing file that fixes units, sample-rate conventions, gain limits, and the no-click rule.
- **Hooks:** run the audio and timing tests when engine files are saved; validate exported WAV headers; keep the README's limits section in step with the spec.
- **Property-based testing:** as listed in section 9.
- **MCP:** a browser-automation server to exercise the deployed app, if one proves useful.
- **A second Kiro surface:** build one contained slice in the CLI and note honestly how it compared with the IDE.

Keep a dated build log in the repository from the first day: what was asked of Kiro, what it produced, what was corrected, and what each feature caught or saved. The write-up is drawn from this log.

## 12. Schedule

- **21–22 November:** requirements review, steering, and the engine spike (draw, hear, export WAV).
- **23–26 November:** strip interface, picture lock, hit anchor.
- **27–29 November:** revise and compare, identity reuse, save and reopen, first deployment.
- **30 November–1 December:** user sessions, polish, second-browser testing. Request hackathon Kiro credits before 1 December, 12:00 Pacific.
- **2 December:** record the video; write the description, Kiro write-up, blog post, and social post.
- **3 December:** submit.
- **4 December:** fixes only.

Optional bonus entries: a public blog post that states it was written for Kiroween 2026, and a public social post that begins with #KiroweenHackathon26.

## 13. Immediate instruction: requirements phase only

Create the standard Feature Spec named ghostreel-mvp using Requirements-First.

For this first pass:

1. Read this brief and summarize the intended user and core workflow.
2. Produce requirements.md with stable IDs, user stories, and testable acceptance criteria.
3. Keep the required MVP distinct from later opportunities and removed items.
4. Identify material ambiguities, risks, and assumptions. Make ordinary implementation assumptions explicit; ask only questions whose answers change the product outcome.
5. Check the draft for conflicting timing, saving, reuse, and export requirements.
6. State whether the scope fits the schedule in section 12, and name what to cut first if it does not.
7. Pause for requirements review before generating design, tasks, or application code.

After review, the next phases are design.md, then tasks.md, then implementation. Each should remain consistent with the approved requirements.

## 14. Sources and context

Official workflow and submission references, checked 21 November 2026:

- [Kiro Specs](https://kiro.dev/docs/specs/)
- [Kiro Requirements-First](https://kiro.dev/docs/specs/feature-specs/requirements-first/)
- [Importing and refining existing requirements](https://kiro.dev/docs/specs/best-practices/)
- [Analyze Requirements](https://kiro.dev/docs/specs/analyze-requirements/)
- [Kiro Steering](https://kiro.dev/docs/steering/)
- [Kiroween 2026 overview](https://kiroween.devpost.com/)
- [Kiroween official rules](https://kiroween.devpost.com/rules)

Historical and competitive context previously reviewed for this project:

- [Science Museum: Oramics Machine](https://collection.sciencemuseumgroup.org.uk/objects/co8188395/oramics-machine)
- [Daphne: an existing Oramics interpretation](https://signalfunctionset.com/projects/daphne/)
- [Tsugi DSP Motion: existing drawn sound and video workflow](https://tsugi-studio.com/web/en/products-dspmotion.html)

The product scope, architecture preferences, example scene, and validation plan in this brief are proposals for GhostReel, not findings claimed by these sources.
