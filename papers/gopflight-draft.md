# GOPFlight: Demand-Reconciled Scheduling for Play, Seek, Scrub, and Thumbnails in Browser Video Editors

**Status:** Working paper draft, July 2026
**Primary research area:** Multimedia systems, browser runtimes, interactive video editing
**Candidate venues:** ACM Multimedia Systems, ACM Multimedia, IEEE ICME, or a web-systems venue
**Recommended short name:** GOPFlight

> **Title decision.** The title names the mechanism rather than claiming a new decoder. “GOPFlight” is the reusable forward decode route; “demand-reconciled scheduling” is the principle that replaces separate imperative play, seek, scrub, and thumbnail queues.

## Draft Marker Legend

- **[EVIDENCE NEEDED P2-E#]** requires a measurement, dataset, or reproducible artifact before submission.
- **[EXPERIMENT TODO P2-X#]** identifies an experiment that must be run.
- **[IMPLEMENTATION GAP P2-I#]** identifies missing runtime behavior or instrumentation.
- **[NOVELTY RISK P2-N#]** identifies a contribution that can be confused with prior work.
- **[DESIGN DECISION P2-D#]** identifies an architectural principle that should remain stable or be deliberately revisited.
- **[RELATED-WORK CHECK P2-R#]** identifies literature that must be rechecked before submission.

## Abstract

Browser video editors must support interaction patterns with incompatible latency and accuracy requirements. Playback needs continuous exact composites; committed seek needs one exact frame; moving scrub benefits from immediate approximate feedback but must converge after dwell; and timeline thumbnails require sparse uniform coverage. Implementing these modes as independent decode queues duplicates work, replays long groups of pictures, competes unpredictably for decoder and memory capacity, and allows stale tasks to survive mode changes. We present **GOPFlight**, a demand-reconciled WebCodecs runtime that derives all decode work from one immutable temporal-demand snapshot. Interaction profiles produce source-space `point`, `stream`, `range`, and `grid` requirements. A pure planner unions overlapping requirements by source sample and random-access GOP, while a stateful reconciler advances reusable forward decoder flights, accounts for pending output as coverage, and preserves useful checkpoints across demand revisions. Exact decode obligations remain separate from approximate presentation policy, enabling responsive scrubbing without falsely declaring the target satisfied. Thumbnail requests subscribe to the same sparse grid outputs rather than launching a second decoding system. We define a comparative evaluation across playback, seek, scrub, direction reversal, multi-layer composition, and thumbnail workloads, measuring visible feedback, exactness, startup and seek latency, decode amplification, GOP-prefix replay, memory, decoder churn, and fairness.

> **[EVIDENCE NEEDED P2-E1]** Replace the final sentence with publication-grade results. Existing traces are useful pilot evidence but cover one evolving implementation, a small source set, and limited hardware.

## 1. Introduction

Interactive browser video editing is not ordinary video playback. A single editing session alternates among continuous play, discontinuous seek, rapid and reversing scrub, paused exact inspection, and sparse thumbnail acquisition. A composition may contain several simultaneously visible sources, repeated references to the same source, speed changes, overlapping clips, and a viewport spanning hundreds of future segments.

Compressed video makes these interactions stateful. An arbitrary target frame usually cannot be decoded independently: the decoder must start from an appropriate random-access sample and feed a dependency-ordered prefix. Repeatedly treating every cursor position or thumbnail as a new seek can replay the same GOP prefix, destroy pending output through decoder reset, and multiply decoded-frame memory. Conversely, a playback-oriented forward queue can waste work after a direction change and starve the exact frame under the pointer.

The browser adds another constraint. WebCodecs provides low-level `VideoDecoder` control, but the application owns demuxing, timestamps, key-sample selection, queueing, frame lifetime, scheduling, and presentation. After decoder configuration, reset, or flush, the next submitted chunk must satisfy the codec’s key-chunk requirement. Resetting a decoder can discard queued work, and decoded `VideoFrame` objects retain substantial resources until closed. These are mechanism-level facts, not an interaction policy.

Prior research already establishes that scrub latency matters, that low-resolution previews and thumbnail grids improve navigation, and that coarse-to-fine frame delivery can preserve temporal coverage. GOP-aligned random access, decoder caching, prefetching, and priority scheduling also have long histories. The contribution of GOPFlight is therefore not “using GOPs” or “showing thumbnails.” It is a unified reconciliation model for an editor runtime:

1. all interaction modes become profiles over one temporal state rather than separate physical decoder implementations;
2. those profiles project into a small set of source-space demand geometries;
3. overlapping exact samples and sparse grid stations share reusable GOP flights;
4. decoded, pending, and warm-route coverage are distinguished explicitly;
5. approximate presentation during movement never weakens exact decode completion;
6. thumbnails consume the same materialized grid outputs as foreground work.

The intended contributions are:

1. A single temporal-demand abstraction for play, committed seek, moving scrub, scrub dwell, and viewport thumbnail coverage.
2. A pure GOP-flight planner and stateful reconciler that preserve useful physical decode progress across changing demand.
3. A separation of exact decode requirements from bounded approximate presentation.
4. A shared thumbnail delivery path that reuses foreground and background decoded outputs.
5. An evaluation methodology for interactive editor workloads, including multi-layer fairness, decode amplification, and final exactness.

> **[NOVELTY RISK P2-N1]** Keyframe seeking, GOP-aware decoding, preview thumbnails, caching, and decoder pooling are established techniques. The paper must claim an integration and scheduling abstraction, then prove its value through strong baselines and ablations.

> **[NOVELTY RISK P2-N2]** Vega-Video detects continuous scrubbing and applies encoding-aware optimizations, reporting up to 4× responsiveness improvement. Swift, Swifter, and Spread Loading already study preview delivery and thumbnail-based scrub interaction. GOPFlight must explain and evaluate what unified physical work reconciliation adds beyond those interaction techniques.

## 2. Research Questions

- **RQ1 — Interaction quality:** Can one demand-driven runtime satisfy the distinct latency and exactness contracts of play, seek, scrub, dwell, and thumbnails?
- **RQ2 — Work reuse:** How much compressed input, decoded output, GOP-prefix replay, and decoder churn are avoided by unioning work into reusable flights?
- **RQ3 — Responsiveness:** Does separating approximate presentation from exact decode demand improve moving-scrub feedback without delaying final exact convergence?
- **RQ4 — Contention:** How does the scheduler behave with several visible layers, future clips, viewport grids, rapid direction changes, and constrained decoder residency?
- **RQ5 — Resource bounds:** Can the runtime bound decoded-frame memory, pending outputs, compressed bytes, and decoder count without mode-specific mechanisms?
- **RQ6 — Portability:** How stable are the results across codecs, GOP structures, resolutions, frame rates, browser versions, operating systems, and hardware?

## 3. Workload and Temporal Model

### 3.1 One Temporal State

At time \(t\), the runtime resolves one immutable snapshot:

\[
R_t = \operatorname{Resolve}(t, m_t, v_t, c_t)
\]

where \(m_t\) contains cursor velocity, acceleration, direction, dwell state, and interaction profile; \(v_t\) is viewport geometry; and \(c_t\) is the current composition.

The snapshot contains:

- **point:** visible video and image layers intersecting the global instant \(t\);
- **transport:** continuous temporal windows needed for interaction progress;
- **viewport:** a zoom-derived global ruler grid plus overscan;
- **frame selection:** presentation policy, separate from decode completion.

Segment boundaries are projection domains, not scheduler events. A global point, interval, or grid either intersects a segment or does not. Source speed and offset are applied once during projection.

> **[DESIGN DECISION P2-D1]** Maintain one temporal source of truth. No downstream module may derive an independent play window, scrub neighborhood, or thumbnail range.

### 3.2 Interaction Contracts

| Profile | Presentation contract | Decode geometry | Completion contract |
|---|---|---|---|
| Play | Exact floor-PTS composite; on miss, retain last complete composite | Current point plus bounded forward stream | Prepare waits for bounded materialization and one exact atomic composite |
| Committed seek | Exact floor-PTS frame | Current point plus finite neighborhood | Latest request resolves only on exact readiness or terminal cancellation/failure |
| Moving scrub | Nearest monotonic cached sample within a motion-derived bound | Exact point plus bounded directional sparse targets and reciprocal hot guard | Approximate feedback is useful but never marks the exact point satisfied |
| Scrub dwell | Exact, identical to committed seek | Seek neighborhood after a dwell threshold | Cursor converges to exact without a synthetic new interaction |
| Viewport thumbnails | Exact selected grid station when delivered | Sparse global ruler grid projected into sources | Each subscriber resolves independently as its station materializes |

The current implementation uses a forward play window of 1.5 seconds; a committed seek or dwell window of 0.5 seconds behind and 1.5 seconds ahead; and an 80 ms dwell threshold.

> **[EXPERIMENT TODO P2-X1]** Treat these constants as evaluated policy parameters, not universal truths. Sweep window sizes, dwell thresholds, feedback intervals, and scrub prediction counts.

### 3.3 Decode Demand Shapes

The temporal snapshot is projected into four source-space geometries:

| Demand | Meaning |
|---|---|
| `point` | One exact floor-quantized source sample |
| `stream` | A rolling continuous source range |
| `range` | Dense finite coverage or explicit exact sparse targets |
| `grid` | Sparse samples projected from a global viewport ruler |

These are geometry classes, not aliases for interaction modes. A future interaction can reuse an existing geometry without adding a new decoder task type.

Let:

\[
Q_t = \operatorname{Project}(R_t, I, B)
\]

where \(I\) is the shared source index and \(B\) is the resource-admission policy.

## 4. GOP Flight Planning and Reconciliation

### 4.1 Pure Flight Planning

The planner quantizes all requirements through the source index, unions identical source samples, and groups missing requirements by random-access GOP:

\[
F_t = \operatorname{Plan}(Q_t, C_t, P_t)
\]

where \(C_t\) is materialized cache coverage and \(P_t\) is pending-output coverage.

Each immutable flight target contains:

```text
{
  source,
  keySample,
  forwardEnd,
  requiredTargets,
  priority,
  routeReplay
}
```

Three concepts remain separate:

| Concept | Responsibility |
|---|---|
| Flight target | Desired stations and endpoint for one source/GOP route |
| Flight driver | Stateful reconciler that repeatedly reads the latest target |
| Decoder session | Physical forward decoder path, pending outputs, and reusable checkpoint |

This separation allows a changing target to reuse a compatible physical route. A cursor update replaces desired state; it does not enqueue an imperative seek command that must later be canceled.

### 4.2 Reconciliation, Not Task Cancellation

The only stateful boundary computes:

\[
\text{Live}_{t+1} =
\operatorname{Reconcile}(\text{Live}_t, F_{t+1})
\]

A removed target stops new submission at the next guard, while already issued reads and decoder outputs are allowed to settle when they still satisfy the latest plan. If a new demand needs the same source/GOP before the route cools, it merges back into the existing driver.

Hard abort is reserved for source invalidation and runtime destruction. Interaction-mode changes alone do not destroy potentially useful physical work.

### 4.3 Pending Output as Coverage

Decoder input history is not sufficient evidence that a sample will become available: reset, decode failure, or resource pressure may discard it. GOPFlight therefore tracks output tickets. A pending ticket matching a latest-plan target is in-flight coverage and prevents duplicate route creation. Before reset or reconfiguration, pending output is reconciled against current demand.

> **[EVIDENCE NEEDED P2-E2]** Instrument every submitted sample through output, cache insertion, discard, close, reset, and failure. The pending-coverage claim requires an auditable lifecycle, not inferred timing.

### 4.4 Warm Checkpoints and Fleet Limits

After finite work completes, a decoder session may remain as a warm source checkpoint. A compatible later target selects the nearest checkpoint whose resume cursor has not passed the required sample; otherwise a new route begins at the target’s real random-access sample.

The current policy caps leased plus warm decoder residency at six and uses separate budgets for:

- physical decoder residency;
- active flight drivers;
- decoded-frame bytes;
- compressed GOP bytes;
- I/O reads;
- speculative samples.

One limit must not silently stand in for another.

> **[EXPERIMENT TODO P2-X2]** Sweep decoder caps and memory budgets. Report the Pareto frontier among interaction latency, replay work, memory, and decoder churn.

### 4.5 Priority, Fairness, and Work Admission

Current composite samples outrank transport prediction, which outranks uniform grid work. Equal-priority routes rotate after a bounded sample or byte quantum. When a visible composite is incomplete, independent current sources may use the resident lanes concurrently; a duplicate route for one source may not occupy a lane needed by another visible layer.

The transport window describes useful value, not an instruction to decode every frame immediately. Each visible layer receives a mandatory sample; speculative depth and future-boundary breadth share explicit global byte and sample budgets.

> **[DESIGN DECISION P2-D2]** Treat demand geometry and resource admission as separate functions. A large useful window must not imply unbounded eager work.

## 5. Exact Decode and Approximate Presentation

Moving scrub creates a perceptual opportunity: a nearby cached frame is often more useful immediately than waiting for the exact target. But presentation tolerance must not mutate scheduler completion.

For moving scrub, the presenter may choose:

\[
\hat{s}_t =
\arg\min_{s \in C_t}
|s - s_t|
\]

subject to:

- \(|s-s_t| \leq \delta(v_t, a_t, \text{viewport})\);
- selection remains monotonic while direction is stable;
- slow movement selects at most one source sample on either side;
- the exact floor-PTS sample \(s_t\) remains outstanding decode demand.

Play, committed seek, and dwell use \(\delta=0\).

This prevents a common semantic error: displaying an approximate frame and then treating the request as complete. The user receives responsive feedback during movement and exact pixels after stopping.

> **[EXPERIMENT TODO P2-X3]** Measure both perceived and physical outcomes: feedback rate and task completion in a user study, plus exact final dwell, temporal error distribution, monotonicity violations, and time-to-exact after motion stops.

## 6. Thumbnail Delivery as a Demand Subscriber

Timeline thumbnails are sparse temporal samples, not a separate kind of decoder. The viewport generates a stable global grid from zoom and overscan. Grid stations are projected through clip timing into source samples, then planned with all other demand.

A station can be materialized by:

- its own low-priority grid flight;
- a foreground point or stream flight that decodes the exact sample;
- a compatible flight byproduct delivered to the station’s subscriber.

Materialization and provisional presentation remain distinct: a nearby byproduct may temporarily fill a visual cell, but it does not retarget or permanently satisfy the exact grid station.

The worker crops and scales delivered frames and transfers thumbnail `ImageBitmap` objects to the main thread. The same source frame can satisfy several projected clips or subscribers.

This does not claim that thumbnail grids are new. Swifter, HiStory, and many storyboard systems already demonstrate their interaction value. The claimed systems contribution is eliminating a second decode pipeline and reusing already paid physical work.

> **[EXPERIMENT TODO P2-X4]** Compare shared-grid decoding with an independent thumbnail decoder pool, pre-generated sprite sheets, and no background thumbnails. Separate local-media, remote-media, cold-cache, and warm-cache conditions.

## 7. Browser Runtime Considerations

### 7.1 WebCodecs Semantics

The runtime must respect:

- decoder configuration and key-chunk requirements;
- decode queue visibility;
- reset and flush effects;
- asynchronous output after input submission;
- transferable but explicitly closeable frame resources;
- browser and hardware limits on concurrent decoders.

WebCodecs exposes mechanisms but does not prescribe editor interaction scheduling. GOPFlight is an application policy above those mechanisms.

### 7.2 Timestamp and Dependency Order

Presentation timestamp order and decoder feed order may differ, especially with B-frames. The source index maps requested PTS samples to dependency-correct DTS feed order and identifies actual random-access samples.

> **[IMPLEMENTATION GAP P2-I1]** Validate indexing and feed order against variable-frame-rate files, B-frame reordering, edit lists, non-zero starts, open GOPs, and malformed media. Correct scheduling over an incorrect index is still incorrect.

### 7.3 Frame Lifetime and Memory

Decoded frames, cached images, pending output, compressed chunks, thumbnails, and compositor intermediates consume different resources. The evaluation must report them separately. A single “cache size” hides the actual pressure source.

> **[IMPLEMENTATION GAP P2-I2]** Add consistent byte attribution for live `VideoFrame` objects, pending decoder output, `ImageBitmap` thumbnails, compressed GOP cache, read buffers, and compositor surfaces.

## 8. Related Work

### 8.1 Interactive Video Scrubbing

Swift demonstrates that scrub latency materially harms navigation and overlays a low-resolution video during movement before returning to high resolution. Swifter presents a pre-cached thumbnail grid and reports large improvements in scene-location tasks. Spread Loading orders frame delivery to provide coarse whole-video coverage early and refine it progressively. HiStory uses hierarchical storyboards for varying temporal granularity. The broader video-interaction literature surveys timeline control, storyboards, direct manipulation, and summarization.

GOPFlight complements these user-interface techniques. It studies how one browser decode runtime can produce exact and approximate feedback, sparse grids, and continuous playback under a shared physical work budget.

### 8.2 Declarative and Encoding-Aware Video Interaction

Vega-Video integrates video into a declarative visualization grammar, separates fast visualization state from delayed player state, and detects continuous scrub interactions to apply encoding-aware optimizations. It is the closest recent example of declarative interaction semantics meeting video-system constraints. GOPFlight differs in focus: multi-layer editing, shared WebCodecs decode work, GOP-route reconciliation, pending-output ownership, and thumbnail reuse.

### 8.3 Video Coding and Random Access

Modern inter-frame codecs organize dependencies around random-access points and GOP structures. Random seek often requires decoding from an earlier key sample. Decoder parallelism, prefetch, cache management, and deadline-aware scheduling are established systems concerns. GOPFlight does not modify a codec or bitstream; it schedules application-visible demands over indexed compressed sources.

### 8.4 Browser Decoding APIs

WebCodecs provides direct access to browser encoders and decoders and defines asynchronous queue, reset, flush, and resource-lifetime behavior. HTML media elements and Media Source Extensions provide higher-level playback mechanisms with less direct control over multi-demand decode planning.

> **[RELATED-WORK CHECK P2-R1]** Conduct a targeted patent and systems-literature search for “decoder flight,” reusable GOP route, multi-cursor decoder scheduling, and shared playback-thumbnail decode before choosing the final terminology.

> **[RELATED-WORK CHECK P2-R2]** Repeat the WebCodecs and browser-editor literature search before submission. Vega-Video appeared in April 2026, indicating rapid movement in this area.

## 9. Evaluation Plan

### 9.1 Baselines

1. **HTML media element:** one element per visible video layer, using native seek and playback.
2. **Stateless WebCodecs:** each point or interaction request starts from a key sample without checkpoint reuse.
3. **Mode-specific queues:** separate play, seek, scrub, and thumbnail work queues.
4. **Unified demand without GOP union:** common temporal model, but each demand decodes independently.
5. **GOP union without pending coverage:** merged targets, but pending output does not suppress duplicate work.
6. **GOPFlight without warm checkpoints:** full planner with sessions closed after finite demand.
7. **Full GOPFlight.**
8. **Pre-generated thumbnail sprite sheet:** thumbnail-specific reference under declared storage and network assumptions.

> **[EXPERIMENT TODO P2-X5]** Make every WebCodecs baseline share the same demuxer, source index, cache, compositor, and presentation rules unless that component is the ablation target.

### 9.2 Workload Matrix

#### Media dimensions

- H.264/AVC, VP9, AV1, and supported HEVC environments;
- short and long GOPs;
- open and closed GOPs;
- B-frame and no-B-frame streams;
- constant and variable frame rate;
- 720p, 1080p, 1440p, and 4K;
- landscape and portrait;
- local and remote sources;
- low and high source counts.

#### Composition dimensions

- one, three, and six simultaneous video layers;
- repeated clips from one source;
- many short clips versus one long clip;
- speed changes;
- source offsets;
- overlapping transitions and effects;
- cold and warm caches.

#### Interaction traces

- uninterrupted playback;
- random committed seeks;
- slow scrub;
- fast scrub;
- repeated direction reversals;
- scrub then dwell;
- scrub then immediate play;
- play while zooming or scrolling the viewport;
- cold and warm thumbnail acquisition.

> **[EVIDENCE NEEDED P2-E3]** Build a versioned, replayable trace corpus. Each trace must record temporal input, viewport state, composition revision, source identity, and expected exact samples.

### 9.3 Metrics

#### User-visible quality

- playback startup latency;
- exact complete-composite rate;
- missed presentation ticks;
- maximum and percentile starvation interval;
- committed-seek latency;
- moving-scrub feedback rate;
- maximum feedback gap;
- temporal presentation error;
- monotonicity violations;
- dwell time-to-exact;
- thumbnail time-to-first-coverage and time-to-complete-grid.

#### Work efficiency

- samples submitted, output, cached, presented, and discarded;
- compressed bytes read;
- decoded-output bytes;
- GOP prefixes replayed;
- duplicate `(source, sample)` submissions;
- decoder configure, reset, flush, close, and construct counts;
- route reuse distance;
- foreground work reused by grid subscribers.

#### Resource behavior

- peak live `VideoFrame` bytes;
- peak pending-output bytes;
- compressed-cache bytes;
- thumbnail bytes;
- decoder and driver concurrency;
- CPU, GPU, and memory utilization;
- energy use where measurable.

#### Correctness and fairness

- exact floor-PTS agreement;
- final exact dwell for every visible layer;
- source starvation by layer;
- equal-priority service distribution;
- stale output shown after demand revision;
- resource leaks after destruction.

### 9.4 Statistical Method

Use identical prerecorded traces for paired comparisons. Report medians, p95 and p99 latency, maximum starvation, empirical distributions, bootstrap confidence intervals, and paired effect sizes. Separate cold and warm runs. Treat device, codec, source, composition, interaction trace, and repeated run as explicit factors.

For user studies, pre-register scene-location tasks, target visibility, completion criteria, and subjective workload measures. Do not use only scheduler counters as a proxy for human-perceived scrub quality.

> **[EVIDENCE NEEDED P2-E4]** Perform a power analysis after pilot variance is known. System traces require repeated runs; user-facing claims require participants.

## 10. Pilot Evidence from the Current Implementation

The repository contains an evolving performance baseline that is useful for selecting experiments but is not yet paper evidence.

### 10.1 Three-Layer Playback

In one current source set, an earlier allocation policy produced 123 exact composites out of 135 requests (91.1%). Two current-only lead runs produced 150/153 (98.0%) and 158/162 (97.5%). Staged play admission was 314 ms and 309 ms in two runs, with no first-running-second starvation in either run.

### 10.2 Thumbnail and Grid Warm-Up

Across three cold thumbnail/grid runs, visible feedback was 27.4–30.5% with maximum starvation of 192–218 ms. After all 128 requested thumbnails were ready, feedback was 46.7–49.1% with maximum starvation of 59–71 ms.

These numbers demonstrate measurable behavior and expose useful metrics. They do not establish general superiority because:

- they come from one development machine and browser setup;
- revisions changed more than one policy at a time;
- source and composition diversity is limited;
- there is no independently implemented baseline;
- some metrics are internal proxies rather than user outcomes.

> **[EVIDENCE NEEDED P2-E5]** Convert pilot scripts into clean, versioned baseline-versus-treatment experiments and retain raw traces, media manifests, browser builds, hardware details, and analysis code.

## 11. Expected Results and Falsifiable Hypotheses

- **H1:** GOP union will reduce duplicate sample submission and GOP-prefix replay under seek, scrub, and repeated-source compositions.
- **H2:** Pending-output coverage will reduce duplicate routes and destructive resets without increasing stale presentation.
- **H3:** Warm checkpoints will improve direction reversal and nearby seek latency at the cost of bounded decoder residency.
- **H4:** Exact-demand and approximate-presentation separation will improve moving feedback while preserving exact dwell completion.
- **H5:** Shared grid delivery will reduce thumbnail-specific decoding work when foreground routes already cross requested stations.
- **H6:** Mode-specific queues may match GOPFlight in isolated single-mode traces but perform worse across rapid interaction transitions.
- **H7:** Under some codecs, devices, or very short GOPs, checkpoint reuse may offer little benefit; the paper should report this boundary.

## 12. Limitations and Threats to Validity

- GOPFlight depends on accurate source indexing and random-access metadata.
- WebCodecs behavior and hardware decoder availability vary across browsers and devices.
- Some codecs or encrypted sources may be unavailable to the low-level runtime.
- A custom compositor has different overhead from a native media element.
- Internal feedback counters may not predict perceived quality.
- Thumbnail sprite sheets can dominate on remote static media when preprocessing is acceptable.
- Warm decoders consume scarce platform resources even when application-level memory appears bounded.
- Product-specific workload choices can favor the proposed scheduler.
- Current pilot history includes overlapping code and policy changes.
- The scheduler does not improve codec compression efficiency or decoder kernels.

## 13. Engineering and Evidence Watchlist

| ID | What must be collected or resolved during development | Why it matters |
|---|---|---|
| P2-E1 | Publication-grade baseline results | Required for all performance claims |
| P2-E2 | End-to-end pending-output lifecycle trace | Supports coverage and reset semantics |
| P2-E3 | Versioned replayable interaction corpus | Enables fair paired evaluation |
| P2-E4 | Power analysis and repeated-run plan | Statistical validity |
| P2-E5 | Raw pilot-to-experiment artifact pipeline | Reproducibility |
| P2-I1 | Robust index validation across timestamp and GOP edge cases | Scheduling correctness |
| P2-I2 | Per-resource byte attribution | Memory claims |
| P2-I3 | Browser capability and fallback matrix | Portability |
| P2-I4 | Formal terminal failure and cancellation taxonomy | Correct completion semantics |
| P2-I5 | Deterministic media and browser test harness | Regression control |
| P2-X1 | Temporal-policy parameter sweep | Avoids magic constants |
| P2-X2 | Decoder and memory budget sweep | Establishes trade-offs |
| P2-X3 | Physical plus user scrub study | Connects counters to usability |
| P2-X4 | Shared thumbnails versus independent and pre-generated paths | Isolates thumbnail contribution |
| P2-X5 | Same-mechanism scheduler baselines | Isolates scheduling effects |
| P2-X6 | Codec, GOP, device, browser, and layer-count matrix | External validity |
| P2-X7 | Full component ablation | Identifies which mechanisms matter |

> **[IMPLEMENTATION GAP P2-I3]** Define behavior when WebCodecs, a codec, hardware concurrency, or transferable frame support is unavailable. Silent fallback to different semantics would invalidate comparisons.

> **[IMPLEMENTATION GAP P2-I4]** Seek and dwell completion need explicit terminal outcomes for source failure, index failure, decode error, supersession, and destruction. A timeout is not a semantic substitute.

> **[IMPLEMENTATION GAP P2-I5]** Pin browser builds or record exact versions and flags. Browser decoder changes can alter results independently of scheduler code.

## 14. Planned Figures and Tables

1. **Figure 1:** one temporal snapshot projected into point, stream, range, and grid demand.
2. **Figure 2:** demand → pure GOP target planning → stateful flight reconciliation → shared cache and presenter.
3. **Figure 3:** three scrub targets in one long GOP using independent seeks versus one reusable flight.
4. **Figure 4:** exact decode obligation separated from approximate moving presentation.
5. **Figure 5:** foreground output satisfying thumbnail grid subscribers.
6. **Table 1:** related systems and contribution boundaries.
7. **Table 2:** workload matrix across codecs, GOPs, layers, and interactions.
8. **Table 3:** latency, feedback, exactness, work amplification, and resource use by baseline.
9. **Figure 6:** GOP replay and decoder resets across a direction-reversal trace.
10. **Figure 7:** latency-memory Pareto frontier under decoder and cache budgets.
11. **Table 4:** component ablations.

## 15. Submission Readiness Criteria

The paper should not be submitted until:

- the claim is consistently “unified demand and reusable-flight scheduling,” not “new GOP decoding”;
- Swift, Swifter, Spread Loading, Vega-Video, WebCodecs, and random-access literature are addressed;
- mode-specific, stateless, no-union, no-pending-coverage, and no-checkpoint baselines are implemented fairly;
- raw traces include submission, output, cache, presentation, discard, reset, and frame-close lifecycles;
- at least three hardware classes and multiple browser or OS configurations are tested;
- codecs, GOP structures, resolutions, frame rates, and layer counts are varied;
- exact final dwell and stale-presentation correctness are verified;
- thumbnail claims include a pre-generated sprite-sheet baseline;
- user-facing scrub claims are supported by a user study or explicitly limited to system metrics;
- all pilot numbers are rerun on frozen commits with controlled changes.

## References

1. W3C. 2026. *WebCodecs*. https://www.w3.org/TR/webcodecs/
2. Thomas Wiegand et al. 2003. *Overview of the H.264/AVC Video Coding Standard*. https://doi.org/10.1109/TCSVT.2003.815165
3. Justin Matejka, Tovi Grossman, and George Fitzmaurice. 2012. *Swift: Reducing the Effects of Latency in Online Video Scrubbing*. https://doi.org/10.1145/2207676.2207766
4. Justin Matejka, Tovi Grossman, and George Fitzmaurice. 2013. *Swifter: Improved Online Video Scrubbing*. https://doi.org/10.1145/2470654.2466149
5. Wolfgang Hürst and Dimitrios Darzentas. 2012. *HiStory: A Hierarchical Storyboard Interface Design for Video Browsing on Mobile Devices*. https://doi.org/10.1145/2406367.2406389
6. Klaus Schoeffmann, Marco A. Hudelist, and Jochen Huber. 2015. *Video Interaction Tools: A Survey of Recent Work*. https://doi.org/10.1145/2808796
7. Carl Gutwin et al. 2019. *Improving Early Navigation in Time-Lapse Video with Spread Loading*. https://doi.org/10.1145/3290605.3300785
8. Dominik Winecki and Arnab Nandi. 2026. *Vega-Video: Integrating Video into the Grammar of Graphics*. https://arxiv.org/abs/2604.24958
