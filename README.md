# ASLAM

**Spatial hearing for robots: know which speaker is talking and where they are relative to us.**

ASLAM explores a microphone-array system that associates speaker IDs with directions relative to a robot. The immediate experience is a simple view of nearby speakers, their speaking activity, and their bearings, with an output a robot can use to face a selected speaker. The longer-term vision includes an interactive 2D/3D representation of conversations, enhanced listening, other sound classes, and potentially a dedicated microphone cluster.

**Status:** concept and research planning. This repository does not yet contain a working implementation, installation procedure, or measured results. The milestones below are a proposed implementation sequence.

## Project brief

The [original ASLAM Google Doc](https://docs.google.com/document/d/1lJZnOdtqwRRqZcgUf0py1M1D1JKg-9NNcGxzuiB2Atg/edit) captures the motivation and initial references. This README adapts that brief and the subsequent clarification to prioritize speaker ID and robot-relative direction, reviewed on September 25, 2026. The link provides traceability; changes to the Doc are not automatically synchronized here.

The idea grew from exploring Reachy Mini and speaker diarization, alongside an interest in separating voices in a noisy restaurant. The central question is: **can a device maintain a useful picture of who is speaking around it, and let a person or robot direct its attention to one of those sources?**

## Capabilities and scope

These capabilities are related, but each needs its own implementation and evaluation:

| Capability | Question it answers | Important boundary |
| --- | --- | --- |
| Sound localization | Where is the sound coming from? | A direction is not a measured distance. |
| Source tracking | Is this the same source as a moment ago? | Pauses, movement, and crossings can break continuity. |
| Speaker diarization | Which speaker spoke when? | Anonymous speaker labels do not imply a person's real-world identity. |
| Track–speaker association | Which voice belongs to which spatial track? | Matching timestamps alone becomes ambiguous during overlapping speech. |
| Selective listening | Can the selected speaker be heard more clearly? | Activity labels do not provide separated audio. |
| Conversation grouping | Which speakers are participating together? | Proximity alone does not establish a conversation. |

The first four capabilities support the immediate goal. Selective listening and conversation grouping are later extensions. Conversation groups could combine spatial evidence, turn-taking, and interaction history, with uncertainty or manual grouping when needed.

## First prototype: a directional speaker map

Proposed initial setting: one stationary, calibrated microphone array in one room, with two to three speakers at distinguishable bearings. Begin with recorded multichannel audio, then run the same pipeline live.

The prototype should:

1. Capture synchronized raw microphone channels with known microphone positions and channel order.
2. Estimate source bearings and display them on a 360-degree directional view.
3. Associate spatial tracks with anonymous speaker IDs such as Speaker A and Speaker B, aiming for consistency within a session through pauses and movement.
4. Show each speaker's direction and speaking state; let the user select a speaker as the robot's attention target.
5. Expose timestamped bearings and confidence for a robot-facing integration, and show stale observations or track loss explicitly.

Use a ring or bearing rays until distance is supported. A point plotted at an arbitrary radius must not look like a measured room position. Closely aligned speakers and strong reflections are explicit test cases, not assumed solved.

The first demo should answer: **with two people speaking from different directions, can I see who is speaking, where they are relative to the robot, and keep their IDs consistent through pauses and modest movement?** A following robot demo should turn toward a selected speaker and settle facing them.

### A simple speaker interface

Use a top-down ring with the robot at its center and an arrow showing forward. Place labeled speakers around the ring by bearing, highlight whoever is speaking, and show the selected attention target. All markers use the same radius because distance is not measured. Detailed signal plots belong in an optional diagnostics view.

Illustrative display, not measured data:

| Speaker | Direction relative to robot | State |
| --- | --- | --- |
| Speaker A | 35 degrees left | Speaking; selected |
| Speaker B | 60 degrees right | Speaking |
| Speaker C | Last heard behind us | Silent; last observed bearing |

Both speakers can be active at once. A silent speaker's last observed bearing must age visibly; silence does not prove that the person stayed in place.

Speaker IDs are anonymous labels within a session. A spatial track ID is not sufficient evidence that the same voice has returned after a pause. Persistent recognition across sessions and real-world names are outside the first prototype.

### Robot-facing output

Proposed fields: `speaker_id`, `track_id`, `azimuth_deg`, `speaking`, `direction_confidence`, `identity_confidence`, `observed_at`, and `frame_id`. Uncertain speaker associations may have no speaker ID yet. Calibrating confidence values is part of evaluation.

Define the initial bearing convention explicitly: zero is forward, positive angles are left, negative angles are right, and angles wrap at the rear. Convert the microphone array's axes into the robot's frame using its calibrated mounting orientation; include head pose if the array moves with the head.

A first controller should follow an explicitly selected speaker, smooth bearing updates, and use a small angular deadband to avoid jitter. Hold when observations are stale or uncertain. After a turn, recompute the relative bearing so the target approaches zero; do not repeatedly apply an old angle as another turn command. Motor noise and changes in head pose are additional evaluation cases for this stage.

## Proposed technical approach

- **Spatial processing:** evaluate [ODAS](https://github.com/introlab/odas) as a backend for localization and tracking. Build a simpler speaker-centric interface; adopting the backend does not require using ODAS Studio as the product interface.
- **Speaker activity:** evaluate [NVIDIA Nemotron 3 Diarization](https://huggingface.co/nvidia/Nemotron-3-Diarization) as a candidate. Its model card specifies single-channel 16 kHz input, up to eight speakers, and timestamped speaker activity. It does not output source positions or isolated voice waveforms.
- **Association:** combine spatial continuity with speaker activity and, if needed, voice embeddings. Keep spatial track IDs distinct from diarization labels; permit uncertain matches rather than forcing an identity.
- **Presentation and robot integration:** expose speaker IDs, timestamped bearings, activity, and confidence to the viewer and robot controller through the same interface.
- **Later audio selection:** evaluate spatial separation and target-speaker extraction after the speaker-direction interface works reliably.

Preserve the original synchronized multichannel stream for spatial processing. Prepare a separate mono input for diarization; downmixing the only copy would discard the inter-microphone information needed for localization.

Measure bearing and labeling latency separately. Nemotron's documented 0.32-second recommended minimum input buffer excludes computation time. A proposed design is to update bearings independently of slower speaker labels, associate observations using timestamps, and avoid presenting an uncertain match as a confirmed speaker. End-to-end delay matters when the robot follows a moving person.

## From directions to a room map

A compact array can estimate arrival direction from differences across microphones. Reliable range is a separate challenge: loudness also depends on the source, while reverberation cues depend on the room. Treat range estimates as uncertain until validated.

The proposed progression is:

1. **Fixed-array directional tracking:** bearings and speaker activity in the device's frame.
2. **Mapping with known device poses:** test multiple observation positions or multiple arrays with known relative poses. Begin with stationary sources before introducing moving speakers.
3. **Acoustic SLAM:** jointly estimate the moving device's trajectory and the source map, with explicit assumptions about source motion and observability.

This distinction follows acoustic SLAM research on [joint array and source localization](https://eurasip.org/Proceedings/Eusipco/Eusipco2016/papers/1570256259.pdf). A stationary speaker display is a useful starting point, but it is not yet simultaneous localization and mapping. A map of sound emitters also does not automatically recover walls or room geometry.

## Milestones and evaluation

| Milestone | Deliverable | Evidence to collect |
| --- | --- | --- |
| 1. Capture and replay | Repeatable multichannel recordings with geometry and timing metadata | Channel order, synchronization, clipping, and dropped samples |
| 2. Directional tracking | Replayable and then live source visualization | Angular error, missed/false sources, and track ID switches |
| 3. Speaker IDs and directions | Simple speaker view with session IDs, bearings, activity, and confidence | Diarization error, association errors, ID continuity through pauses, and update latency |
| 4. Face a speaker | Robot turns toward the selected speaker using current relative bearings | Final heading error, settling time, jitter, stale-observation handling, and robustness to motor noise |
| 5. Expansion | Spatial mapping, enhanced listening, conversation grouping, other sound classes, or a dedicated device | A separate evaluation for the chosen application |

Use [LOCATA](https://www.locata.lms.tf.fau.de/datasets/) for localization/tracking comparisons, alongside recordings from the actual hardware. Public benchmarks cannot substitute for testing the selected array in the intended rooms. Include separated and overlapping speech, silence, reverberation, close bearings, and moving speakers. Add robot rotation and motor noise when evaluating the turn-to-speaker stage.

Set numerical acceptance thresholds after establishing the baseline and choosing the target hardware and use case. No accuracy, latency, range, or battery-life claims have been established yet.

## Research references from the brief

These projects solve different parts of the problem; they are candidates and references, not integrated dependencies.

| Project | Relevance to ASLAM |
| --- | --- |
| [ODAS](https://github.com/introlab/odas) | Embedded sound localization, tracking, separation, and post-filtering; first baseline to evaluate. |
| [Nemotron 3 Diarization](https://huggingface.co/nvidia/Nemotron-3-Diarization) | Speaker activity and diarization; requires separate spatial processing and track association. |
| [3D-SoundSourceMapping](https://github.com/JiangWAV/3D-SoundSourceMapping) | Camera–microphone source mapping with unknown relative sensor poses; its demo uses RGB-D and ORB-SLAM2, so it is a reference for a sensor-assisted mapping stage. |
| [Acoustix](https://github.com/GaetanLepage/acoustix) | Dynamic acoustic simulation for testing sources and arrays in reverberant environments. |
| [EchoSLAM Acoustics](https://github.com/ERC-BPGC/echoslam_acoustics) | Echo-based localization and environment-mapping simulations; adjacent to the passive speech-focused starting point. |
| [Acoular](https://github.com/acoular/acoular) | Microphone-array beamforming and acoustic source maps; useful for offline experiments and comparisons. |

## Longer-term opportunity

The broader hypothesis is a reusable **spatial hearing system for robots and interactive devices**: a changing representation of which sources exist, where they are, when they speak, and how to attend to them.

The map is one interface to that representation. A robot could use the same tracks to orient toward a speaker, maintain attention through movement, or choose an audio stream for speech recognition. The potential contribution is reliable integration and measured behavior across these tasks; the references above already cover substantial parts of the underlying signal processing.

Selective listening for people and spatial audio interfaces are possible later applications. Start with the tabletop/robotics prototype to validate the core before committing to wearable hardware or a broader product. Product demand and differentiation remain hypotheses to test.

## Decisions still open

- Which array exposes synchronized raw channels, and what geometry suits the first experiment?
- What host computer and latency budget should the first live prototype target?
- Which robot or head mechanism should consume the speaker-direction output?
- For room-scale mapping, are cameras, known device poses, or multiple arrays acceptable?
- How long should a speaker ID persist through silence, and what evidence supports re-association?

Audio isolation, custom hardware, automatic conversation grouping, general sound-event recognition, and full acoustic SLAM remain later stages of the vision. The immediate next step is a repeatable capture/replay baseline that can test speaker IDs and robot-relative directions.
