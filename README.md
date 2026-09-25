# ASLAM

**Spatial hearing: map nearby speakers, track them over time, and choose whom to listen to.**

ASLAM explores a microphone-array system that turns surrounding audio into an interactive 2D/3D representation of sound sources. The long-term experience is to see nearby conversations, distinguish their individual speakers, and select a speaker or conversation for enhanced listening. Other sounds, such as a horn or alarm, are a later extension. A dedicated microphone cluster could eventually package the system into a device.

**Status:** concept and research planning. This repository does not yet contain a working implementation, installation procedure, or measured results. The milestones below are a proposed implementation sequence.

## Project brief

The [original ASLAM Google Doc](https://docs.google.com/document/d/1lJZnOdtqwRRqZcgUf0py1M1D1JKg-9NNcGxzuiB2Atg/edit) captures the motivation and initial references. This README adapts that brief and adds a proposed technical scope, reviewed on September 25, 2026. The link provides traceability; changes to the Doc are not automatically synchronized here.

The idea grew from exploring Reachy Mini and speaker diarization, alongside an interest in separating voices in a noisy restaurant. The central question is: **can a device maintain a useful picture of who is speaking around it, and let a person or robot direct its attention to one of those sources?**

## What the system needs to do

These capabilities are related, but each needs its own implementation and evaluation:

| Capability | Question it answers | Important boundary |
| --- | --- | --- |
| Sound localization | Where is the sound coming from? | A direction is not a measured distance. |
| Source tracking | Is this the same source as a moment ago? | Pauses, movement, and crossings can break continuity. |
| Speaker diarization | Which speaker spoke when? | Anonymous speaker labels do not imply a person's real-world identity. |
| Track–speaker association | Which voice belongs to which spatial track? | Matching timestamps alone becomes ambiguous during overlapping speech. |
| Selective listening | Can the selected speaker be heard more clearly? | Activity labels do not provide separated audio. |
| Conversation grouping | Which speakers are participating together? | Proximity alone does not establish a conversation. |

Start with individual speakers. Conversation groups can later combine spatial evidence, turn-taking, and interaction history, with uncertainty or manual grouping when needed.

## First prototype: a directional speaker map

Proposed initial setting: one stationary, calibrated microphone array in one room, with two to three speakers at distinguishable bearings. Begin with recorded multichannel audio, then run the same pipeline live.

The prototype should:

1. Capture synchronized raw microphone channels with known microphone positions and channel order.
2. Estimate source bearings and display them on a 360-degree directional view.
3. Maintain temporary track IDs and associate anonymous speaker labels with those tracks.
4. Let the user select a track and compare enhanced audio with the original mixture.
5. Show confidence and track loss explicitly, including when a speaker becomes silent.

Use a ring or bearing rays until distance is supported. A point plotted at an arbitrary radius must not look like a measured room position. Closely aligned speakers and strong reflections are explicit test cases, not assumed solved.

The first demo should answer: **with two people speaking from different directions, can I select either person, hear an improvement, and keep that selection through pauses and modest movement?**

## Proposed technical approach

- **Spatial processing:** evaluate [ODAS](https://github.com/introlab/odas) first as a baseline for localization, tracking, separation, and post-filtering. Its existing visualization project also provides a useful comparison before building a new interface.
- **Speaker activity:** evaluate [NVIDIA Nemotron 3 Diarization](https://huggingface.co/nvidia/Nemotron-3-Diarization) as a candidate. Its model card specifies single-channel 16 kHz input, up to eight speakers, and timestamped speaker activity. It does not output source positions or isolated voice waveforms.
- **Association:** combine spatial continuity with speaker activity and, if needed, voice embeddings. Keep spatial track IDs distinct from diarization labels; permit uncertain matches rather than forcing an identity.
- **Audio selection:** begin with the spatial separation baseline; add target-speaker extraction only if tests show that direction-based selection is insufficient.
- **Presentation:** expose timestamped source tracks, activity, confidence, and selected audio to a small viewer. Keep this interface reusable by a future robot integration.

Preserve the original synchronized multichannel stream for spatial processing. Prepare a separate mono input for diarization; downmixing the only copy would discard the inter-microphone information needed for localization.

Measure the audio and labeling paths separately. Nemotron's documented 0.32-second recommended minimum input buffer excludes computation time. A proposed design is therefore to let audio enhancement run independently of slower label updates, then associate results using timestamps. A responsive map does not by itself establish acceptable listening latency.

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
| 3. Speaker-aware selection | Speaker labels linked to tracks and selectable enhanced audio | Diarization error, association errors, target leakage, listening comparisons, and end-to-end latency |
| 4. Spatial mapping | Source positions with uncertainty in a shared coordinate frame | Position error against ground truth; robustness to source and device motion |
| 5. Expansion | Conversation grouping, robot integration, other sound classes, or a dedicated device | A separate evaluation for the chosen application |

Use [LOCATA](https://www.locata.lms.tf.fau.de/datasets/) for localization/tracking comparisons, alongside recordings from the actual hardware. Public benchmarks cannot substitute for testing the selected array in the intended rooms. Include separated and overlapping speech, silence, reverberation, close bearings, and moving speakers. For audio separation measurements, use controlled mixtures with clean source references as well as real-room listening tests.

Set numerical acceptance thresholds after establishing the baseline and choosing the target hardware and use case. No accuracy, latency, range, or battery-life claims have been established yet.

## Research references from the brief

These projects solve different parts of the problem; they are candidates and references, not integrated dependencies.

| Project | Relevance to ASLAM |
| --- | --- |
| [ODAS](https://github.com/introlab/odas) | Embedded sound localization, tracking, separation, and post-filtering; first baseline to evaluate. |
| [Nemotron 3 Diarization](https://huggingface.co/nvidia/Nemotron-3-Diarization) | Speaker activity and diarization; requires a separate spatial and audio-separation pipeline. |
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
- Is the first useful outcome a robot hearing interface or human audio monitoring?
- For room-scale mapping, are cameras, known device poses, or multiple arrays acceptable?
- What evidence will justify advancing from speaker selection to automatic conversation grouping?

Custom hardware, automatic conversation grouping, general sound-event recognition, and full acoustic SLAM remain later stages of the vision. The immediate next step is a repeatable capture/replay baseline that can test directional tracking and selected-speaker audio.
