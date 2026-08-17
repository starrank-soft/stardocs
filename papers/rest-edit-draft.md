# REST-Edit: A Lightweight, Streaming-Friendly REST-Like Protocol for Precise LLM Video Editing

**Status:** Working paper draft, July 2026
**Primary research area:** LLM agents, multimedia authoring, collaborative systems
**Candidate venues:** ACM Multimedia, ACM IUI, CHI, or an agent-systems venue
**Recommended short name:** REST-Edit

> **Title decision.** “REST-like” is deliberately used instead of “RESTful.” The protocol borrows semantic resource addressing and familiar method-like operation headers, but it is not a network architecture satisfying the full REST constraints defined by Fielding. The title emphasizes the intended optimization: a lightweight and incrementally parsable language for precise partial edits.

## Draft Marker Legend

- **[EVIDENCE NEEDED P1-E#]** requires a measurement, dataset, or reproducible artifact before submission.
- **[EXPERIMENT TODO P1-X#]** identifies an experiment that must be run.
- **[IMPLEMENTATION GAP P1-I#]** identifies a protocol capability or invariant not yet implemented.
- **[NOVELTY RISK P1-N#]** identifies a claim that overlaps prior work and must be narrowed.
- **[DESIGN DECISION P1-D#]** identifies a decision that should remain stable or be deliberately revisited during development.
- **[RELATED-WORK CHECK P1-R#]** identifies literature that must be rechecked immediately before submission.

## Abstract

Large language model agents can plan video edits, but reliable execution remains difficult. A video project is already a live, identity-bearing runtime model: clips, tracks, effects, text, references, ordering, and collaborative state have established semantics. Rewriting its complete serialized representation can overwrite unrelated state or bypass the model’s mutation boundaries, while exposing every mutation as a separate tool call increases protocol overhead. We present **REST-Edit**, a lightweight REST-like protocol for precise partial editing of a schema-projected video document. The model addresses visible resources by short stable identifiers and emits a linear stream of semantic `PATCH`, `POST`, `PUT`, and `DELETE` blocks. Operation headers, property lines, and embedded VML subtrees can be recognized incrementally with bounded parser state. The executor maps these operations back through the existing runtime model, stages the proposed change, validates document invariants, and commits one canonical transaction. We propose an execution-based evaluation against fine-grained tools, structured operation arrays, and full-document regeneration, measuring edit accuracy, grammar validity, time to first parsed operation, parser buffering, tokens, bytes, model turns, unintended mutations, and end-to-end latency.

> **[EVIDENCE NEEDED P1-E1]** Replace the final evaluation sentence with measured results. The current implementation supports the grammar, incremental parser components, precise executor, and atomic commit as separate pieces, but the streamed field-delta path is not fully connected and no publication-grade cross-model benchmark has been run.

## 1. Introduction

Video editing agents sit between two very different abstractions. Users express high-level goals such as “make the opening faster, keep the product close-up, and align the final cut to the beat.” Non-linear editors, however, store a graph of tracks, clips, effects, generated assets, timing constraints, and ownership relations. The agent must cross this gap without corrupting the project or forcing the language model to reproduce the editor’s internal state.

Recent systems demonstrate that LLMs can interpret natural-language edit intent and invoke editing functions. LAVE combines language descriptions with agent planning and execution; ExpressEdit grounds temporal, spatial, and operational references; EditDuet separates editing and critique; VideoAgent orchestrates a large library of specialized editing agents; Crayotter makes workflow artifacts traceable; and RefineCut emits typed timeline patches checked by a deterministic verifier. These works establish the broader problem and provide useful reference points. REST-Edit does not need to claim that structured editing is unprecedented; it studies a different and incremental systems question: whether a semantic, short-address, streaming-friendly protocol is a better interface between an LLM and an already modeled editing runtime.

The central problem is **precise editing of structured data without replacing the structure that already gives the data meaning**. A useful agent-computer interface should let the model identify exactly which semantic resource and property it intends to change, without reproducing the whole document and without navigating long storage paths. It should reuse the editor’s established mutation layer, preserve unrelated runtime state, support concise multi-resource edits, and remain easy for a generative model to emit and for a deterministic parser to consume.

REST-Edit addresses this question through four separations:

1. The canonical document is a flat CRDT graph and remains the only persistent source of truth.
2. The model reads a schema-projected video markup representation containing stable visible identities, not raw CRDT fields.
3. The model writes a compact, line-oriented operation stream addressed by semantic identity rather than long traversal paths.
4. A deterministic executor maps the stream back through the existing model APIs and owns identity allocation, structural mutation, validation, and transactional commit.

The paper makes the following intended contributions:

1. A lightweight REST-like editing language that uses short semantic resource identifiers, locally recognizable operation boundaries, property-oriented patches, and native VML subtree creation.
2. A streaming-oriented parser design that can consume arbitrary model-output chunks without requiring a complete document or a globally nested operation structure.
3. A schema-derived executor in which one definition drives model-visible projection, operation routing, structural constraints, creation, validation, and serialization.
4. A precise-edit execution path that preserves the existing runtime model, stages proposed mutations, and commits one validated canonical transaction.
5. An evaluation methodology for comparing generation and parsing behavior across REST-Edit, fine-grained tools, structured operation arrays, and full-document regeneration.

> **[NOVELTY RISK P1-N1]** Do not define the contribution as the invention of partial updates or four operation names. The contribution is the design and measured optimization of an LLM-facing protocol: semantic short addressing, low syntactic overhead, incremental parsing, direct reuse of the projected domain language, and precise execution through an existing runtime model.

> **[NOVELTY RISK P1-N2]** RefineCut is close related work because it also produces executable structured timeline changes and verifies them. It should be cited as an independently developed neighboring approach, not described as the origin of REST-Edit. The repository records the original VML REST-like protocol on March 9, 2026, before RefineCut’s public May 26, 2026 submission; preserve this provenance in an archival research log.

## 2. Research Questions

- **RQ1 — Precise editability:** Does REST-Edit increase the probability that an LLM changes exactly the intended resources while preserving unrelated modeled state?
- **RQ2 — Generation efficiency:** How does it change grammar validity, generated tokens, transmitted bytes, tool calls, model turns, and wall-clock latency?
- **RQ3 — Streaming and parsing:** How early can a receiver recognize complete operations, how much buffering and parser state are required, and how locally can errors be diagnosed?
- **RQ4 — Scale:** How do correctness and communication cost change with timeline size, edit span, nesting depth, and number of affected resources?
- **RQ5 — Runtime preservation:** How reliably does each interface route edits through established model boundaries rather than replacing or reconstructing the full document?
- **RQ6 — Schema convergence:** How much implementation duplication and schema drift are eliminated when one schema drives projection and execution?

## 3. Problem Formulation

Let the canonical collaborative document be a graph:

\[
D = (N, O, A, T)
\]

where \(N\) is a set of identity-bearing nodes, \(O\) maps each contained node to one atomic owner pair, \(A\) contains schema-declared attributes, and \(T\) contains shared text. Ordered members carry a fractional order value. Every contained node must reach exactly one document root through its owner chain.

Let \(S\) be a document schema. The AI-visible representation is a projection:

\[
V = \pi_S(D)
\]

that includes only visible tags, attributes, text, references, and stable identifiers. Internal ownership, ordering, storage paths, CRDT metadata, and other reserved fields are not writable through \(V\).

The agent emits an edit program:

\[
P = [o_1, o_2, \ldots, o_k]
\]

where each \(o_i\) is one of four resource-oriented operation forms. The executor computes:

\[
D' = \operatorname{Commit}\left(
  \operatorname{Validate}_S\left(
    \operatorname{Apply}_S(\operatorname{Clone}(D), P)
  \right)
\right)
\]

and commits nothing if parsing, resolution, application, or validation fails.

The desired outcome is not textual similarity between two edit programs. It is semantic equivalence of their canonical final states under task-specific tolerances:

\[
\operatorname{Equivalent}(D', D^\*) = \text{true}
\]

where \(D^\*\) is a verifier-defined target or a set of allowed target states.

## 4. Protocol Design

### 4.1 Read Projection

The model reads Video Markup Language (VML), a schema-projected view of the canonical document. Each visible resource carries a stable identifier. Tags and attributes correspond to declared domain concepts rather than CRDT storage layout.

The projection has five intended properties:

- **Addressability:** every writable visible entity can be referenced without positional array paths.
- **Semantic locality:** a short visible identifier selects the modeled resource directly; the model does not reproduce a storage traversal path.
- **Safety:** hidden structural fields cannot be copied back or mutated by the model.
- **Compactness:** the model sees domain-relevant state, not runtime or replication metadata.
- **Round-trip semantics:** the same schema definition determines what can be serialized, parsed, created, and edited.

> **[EVIDENCE NEEDED P1-E2]** Quantify projection compression against the raw canonical representation: bytes, model tokens, and number of exposed fields across small, medium, and large projects.

### 4.2 Write Language

REST-Edit uses four operation forms.

#### `PATCH` — Update declared properties or shared text

```text
PATCH #clip-17
opacity="0.5"
start="1200000"
```

Scalar assignments may update only schema-declared properties. Reserved structural fields such as `owner`, `order`, and `path` are rejected. Shared text uses an exact search/replace form, which acts as a local precondition.

#### `POST` — Create a visible child

```text
POST #video-track-2
before: #clip-23
<VideoClip ref="artifacts/product-demo.mp4"
           start="0"
           duration="5000000"
           sourceStart="0"
           sourceDuration="5000000" />
```

The operation addresses the visible parent rather than an internal collection. The child tag and parent schema must identify exactly one property or member field. The executor, not the model, generates the new identifier.

#### `PUT` — Move or reorder an ordered member

```text
PUT #clip-17
to: #video-track-3
before: #clip-31
```

Omitting `to` reorders the resource within its current parent. The executor validates the destination, prevents ownership cycles, replaces the atomic owner during reparenting, and computes a new fractional order.

#### `DELETE` — Delete a subtree

```text
DELETE #clip-17
```

Deletion removes the target and its complete contained subtree. Document-level deletion is outside this operation because reference checking belongs to the project file layer.

### 4.3 Why “REST-Like,” Not “RESTful”

REST is an architectural style for networked hypermedia systems, not a synonym for “uses HTTP verbs.” REST-Edit does not claim hypermedia-driven application state, HTTP caching semantics, uniform network intermediaries, or stateless client-server deployment. It borrows two ideas:

1. a semantic resource is addressed directly instead of through an imperative UI sequence or a long structural path; and
2. a small, familiar operation vocabulary makes the intended kind of change recognizable before its complete body has arrived.

This naming precision is important scientifically. The method should be evaluated as an agent-computer interface and edit transaction language, not as a new web architecture.

### 4.4 Precise Partial Editing Versus Full Replacement

The primary contrast in this paper is not REST-Edit versus JSON Patch. It is **precise semantic mutation versus full representation replacement**.

A full-document output asks the model to reproduce both the intended change and every unchanged fact. This creates a risk surface proportional to the document rather than the edit. More importantly, a serialized document is only a projection of the running editor model. Replacing that projection can accidentally reconstruct identity-bearing objects, disturb references or ownership, reset state that was not visible to the model, or bypass the mutation methods that maintain invariants.

REST-Edit instead expresses a delta against the stable abstraction already owned by the runtime:

\[
\text{edit cost and risk} \propto \text{intended change set}, \quad
\text{not total document size}
\]

For example, changing one clip’s opacity names the clip and the declared property. It does not reproduce the clip’s owner, order, source metadata, neighboring clips, or document root. Creating a child names its semantic parent and supplies only the new VML subtree. Moving a clip preserves its identity and descendants.

This precision produces downstream benefits when the executor is connected to a transactional collaborative model: unrelated state is left untouched, the change can be committed as one unit, and the existing collaboration and undo mechanisms observe normal model mutations. Those benefits come from the complete architecture; they are not claimed as syntax features that generic patch formats cannot support.

> **[DESIGN DECISION P1-D1]** Keep the runtime model authoritative. VML is a readable projection and REST-Edit is a mutation interface; neither becomes a second persistent document tree.

> **[EXPERIMENT TODO P1-X1]** Inject hidden and unrelated runtime state, execute semantically identical tasks through precise edits and full replacement, and measure unintended field changes, identity churn, reference breakage, and validation failures.

### 4.5 Relationship to JSON Patch and Other Generic Patch Formats

JSON Patch addresses the same broad problem—precisely modifying structured data without replacing the complete JSON value—but it is not the conceptual definition of REST-Edit. It is cited because it is established adjacent work and because reviewers need to understand the boundary between a generic data patch and a domain protocol.

The comparison should remain at the representation layer:

| Representation property | JSON Patch | REST-Edit |
|---|---|---|
| Target abstraction | Generic JSON value tree | Schema-projected video resource graph |
| Target expression | JSON Pointer path | Short stable visible resource ID |
| Mutation expression | Generic value operations | Domain property update, VML subtree creation, semantic reparent or reorder, subtree deletion |
| Surface syntax | JSON array of operation objects | Line-oriented method blocks with VML bodies |
| Schema relationship | External to the patch format | Same schema drives projection and execution |
| Creation payload | JSON value at a path | Native projected VML subtree |
| Structural ownership | Expressed through JSON shape and paths | Retained by runtime and mutated through model APIs |

JSON Patch can be parsed incrementally, can be embedded in collaborative systems, and can participate in undoable transactions. REST-Edit should not claim otherwise. The research hypothesis is narrower: for an LLM editing this domain, short semantic identifiers and a line-oriented protocol with native VML fragments may be easier and lighter to generate and parse than a generic nested operation representation. That hypothesis must be measured.

> **[EXPERIMENT TODO P1-X2]** Compare REST-Edit and a well-designed JSON Patch representation only on shared dimensions: output tokens, syntax validity, time to first complete operation, peak parser buffer, error locality, target correctness, and unintended mutations. Use the same runtime executor after parsing.

### 4.6 Streaming-Friendly Grammar

REST-Edit is designed as a linear text stream:

- an operation type and target become known after one header line;
- each scalar `PATCH` assignment becomes complete after one property line;
- `DELETE` is complete after its header;
- `PUT` uses a small bounded header block;
- `POST` switches to the existing incremental XML parser as soon as the first `<` arrives;
- a completed XML root closes the operation without waiting for the full tool response.

The intended parser needs only the current line buffer, the current operation state, and the XML nesting state for the active `POST`. It does not need to materialize the full document or backtrack across earlier operations.

Streaming output and atomic canonical commit are separate properties. The desired execution pipeline is:

```text
LLM output deltas
  -> incremental top-level field decoder
  -> REST-Edit state machine
  -> isolated draft or staged operation log
  -> end-of-stream validation
  -> one canonical transaction
```

This permits early parsing, early diagnostics, and optional draft preview without exposing a partially generated edit as authoritative project state.

> **[IMPLEMENTATION GAP P1-I1]** `OpsLineParser` and the XML parser already accept arbitrary chunks, but the production `edit` tool does not currently wire `onFieldDelta`, and `StructuredEditing` buffers received chunks before parsing the complete input. The paper may claim a streaming-friendly grammar today; it must not claim end-to-end incremental parsing until this path is connected and measured.

> **[EVIDENCE NEEDED P1-E3]** Test every protocol form under adversarial chunk boundaries, including one-character chunks, split Unicode sequences at the transport boundary, escaped outer JSON strings, incomplete operations, nested XML, CDATA, and multiple operations in one delta.

### 4.7 Transactional Execution

The executor clones the current Y.Doc state into an isolated draft, applies the complete parsed program through schema-aware mutations, validates diagnostics and graph invariants, encodes the state difference, and applies that update to the source document in one transaction.

This arrangement provides:

- no observable partially applied agent program;
- one validation point over the complete proposed state;
- one CRDT transaction and one undo boundary;
- reuse of the canonical state’s synchronization and persistence path;
- deterministic rejection before collaborative publication.

It does **not** guarantee that an agent’s intent remains valid after arbitrary concurrent changes. That is a separate stale-intent concern and should be reported as a robustness boundary rather than treated as a syntax claim.

> **[EVIDENCE NEEDED P1-E4]** Add fault-injection tests proving all-or-nothing behavior for parse failure, missing targets, invalid attributes, illegal moves, cycle creation, invalid subtree creation, and validation failure at every operation index.

### 4.8 One Schema as the Semantic Compression Boundary

The architecture is strongest when a single schema fact drives all dependent behavior:

```text
schema declaration
  -> AI-visible projection
  -> tag and attribute parsing
  -> edit routing
  -> parent-child admissibility
  -> creation defaults
  -> validation
  -> serialization
```

If these facts are separately maintained, the protocol can remain syntactically elegant while becoming semantically inconsistent. The implementation therefore treats conceptual compression as a correctness property.

> **[EXPERIMENT TODO P1-X3]** Measure schema convergence with a mutation study: add or modify representative node properties and count required source edits, inconsistent behaviors, and generated tests under REST-Edit versus a manually duplicated baseline.

## 5. Communication and Complexity Analysis

Assume a task affects \(k\) resources and requires \(r\) model-tool round trips under a fine-grained tool interface. Let \(C_s\) be static tool-schema context, \(C_i\) the returned observation at round \(i\), and \(O_i\) the generated operation.

A simplified uncached communication cost is:

\[
C_{\text{tools}} = \sum_{i=1}^{r} (C_s + C_i + O_i)
\]

For a single REST-Edit batch:

\[
C_{\text{REST-Edit}} = C_p + C_V + O_P
\]

where \(C_p\) is the protocol specification, \(C_V\) is the relevant VML projection, and \(O_P\) is the batched edit program.

This formulation does not prove savings. Modern model providers may cache repeated prefixes; tools may execute in parallel; a full VML view may be larger than targeted tool observations; and one failed batch may require a costly retry. The paper must separately report:

- total input tokens;
- non-cached input tokens;
- output tokens;
- serialized request and response bytes;
- tool calls and model turns;
- end-to-end latency;
- retry and repair cost;
- dollar cost under declared provider pricing.

Protocol-generation efficiency also requires grammar-level measurements:

- valid operations before retry;
- tokens per semantic mutation;
- maximum and average target-expression length;
- time from first output token to first complete operation;
- receiver bytes buffered before the first operation can be parsed;
- parser state and peak memory;
- error span and earliest diagnostic position.

The current composition protocol documentation contributes approximately 25.7 kB of static text before the broader system prompt: 15,980 bytes for the project specification and 9,703 bytes for the composition specification.

> **[EVIDENCE NEEDED P1-E5]** Tokenize the actual production prompts for every evaluated model. Bytes are not tokens, and cached prompt cost must not be reported as if it were uncached marginal cost.

> **[NOVELTY RISK P1-N3]** “Fewer tool calls” is not automatically “less data” or “lower cost.” A batched protocol can lose if it repeatedly includes a large document projection. Claims must be based on traces from equal tasks and equal model settings.

## 6. Related Work

### 6.1 LLM-Assisted Video Editing

LAVE establishes agent planning and function execution over language-augmented footage. ExpressEdit studies natural-language and sketch expressions for temporal, spatial, and operational edit references. StoryFlow uses language representations for shot-level editing decisions. EditDuet uses editor and critic agents. Crayotter emphasizes inspectable artifacts and tool-grounded long-form execution. VideoAgent uses more than thirty specialized agents and graph orchestration. These systems motivate structured execution but primarily study interaction, planning, orchestration, or workflow quality.

RefineCut is the closest known work. It trains an open-weight planner to edit a typed timeline through structured patches and uses a deterministic verifier with an explicit constraint ledger. The approaches were developed independently: the repository records the initial VML REST-like protocol in commit `6b09012` on March 9, 2026, while the public RefineCut submission is dated May 26, 2026. Once RefineCut is known, however, normal scholarship requires citing and comparing it. This citation identifies neighboring work; it does not imply that REST-Edit was derived from it.

### 6.2 Tool-Using Language Models and Agent Interfaces

ReAct interleaves reasoning and actions. Toolformer studies self-supervised tool use. API-Bank and ToolBench evaluate tool selection, planning, and API invocation across large tool libraries. SWE-agent shows that a deliberately designed agent-computer interface can materially affect agent behavior and task success. REST-Edit applies this interface-design perspective to a structured multimedia document.

### 6.3 Patch and Resource Protocols

Fielding defines REST as a constrained architectural style for network applications. RFC 6901 defines JSON Pointer, RFC 6902 defines JSON Patch, and RFC 7396 defines JSON Merge Patch. These works belong in related work because they address resource-oriented interaction and precise partial modification of structured data. REST-Edit is not presented as an extension or re-encoding of JSON Patch; it is a domain protocol optimized around a projected video model, semantic short addressing, native VML fragments, and incremental LLM output.

### 6.4 Collaborative Structured Data

Conflict-free replicated JSON datatypes define convergence for concurrent map and list modifications. Replicated-tree research shows that move operations are especially subtle under concurrency and can introduce cycles or divergent structure if modeled incorrectly. REST-Edit does not introduce a new CRDT algorithm; it constrains an LLM-facing operation language over an existing CRDT-backed editor.

> **[RELATED-WORK CHECK P1-R1]** Repeat the literature search immediately before submission. Video-editing agents are developing quickly; VideoAgent, Crayotter, and RefineCut all appeared or changed in 2026.

## 7. Evaluation Plan

### 7.1 Systems and Baselines

1. **REST-Edit:** the complete proposed method.
2. **Fine-grained tools:** one typed tool per editor action, with targeted reads.
3. **Parallel multi-tool calls:** the strongest available tool-calling baseline, not a forced sequential version.
4. **Structured operation array:** one tool call containing typed operation objects over the same semantic resources.
5. **Full-document regeneration:** model rewrites the complete projected document, followed by validation.
6. **Typed timeline patches:** a RefineCut-compatible or faithfully reconstructed patch representation.
7. **JSON Patch, secondary protocol reference:** a domain mapping through stable JSON paths, evaluated only where the comparison is semantically fair.
8. **GUI or computer-use agent, optional:** useful for ecological comparison but not a clean protocol ablation.

> **[EXPERIMENT TODO P1-X4]** Implement all non-GUI baselines through the same canonical editor mutation layer. Otherwise executor quality, not interface design, will confound the comparison.

### 7.2 Task Dataset

The benchmark should contain executable tasks across:

- property changes;
- exact text changes;
- clip insertion and generated-asset insertion;
- intra-track reorder;
- cross-track movement;
- subtree deletion;
- multi-operation montage construction;
- timing and duration constraints;
- multi-layer compositing;
- shared-source reuse;
- tasks with several valid final states;
- concurrent human edits between read and commit.

Each task record should include:

- initial canonical state;
- natural-language instruction;
- allowed assets and metadata;
- hard constraints;
- one target state or an executable semantic verifier;
- difficulty attributes;
- expected affected-resource span.

> **[EVIDENCE NEEDED P1-E6]** Build and release, if licensing permits, at least a few hundred tasks with stratified train/development/test splits. A handful of product demos will not support a paper-level generalization claim.

### 7.3 Models and Sampling

Evaluate at least:

- one frontier proprietary model;
- one smaller cost-efficient proprietary model;
- one open-weight model that supports tool or structured output;
- multiple repeated runs per task and model.

Report exact model versions, dates, temperatures, context limits, schema formats, retry policies, and prompt caching behavior.

> **[EXPERIMENT TODO P1-X5]** Freeze model snapshots for the final experiment. Silent provider model updates make agent-interface comparisons irreproducible.

### 7.4 Metrics

#### Correctness

- executable success rate;
- semantic final-state success;
- hard-constraint satisfaction;
- number and type of rejected operations;
- repair attempts;
- unintended resource mutations.

#### Efficiency

- input, cached-input, and output tokens;
- serialized bytes;
- model turns;
- tool calls;
- average target-expression length;
- syntax-valid operation rate;
- time to first complete parsed operation;
- receiver bytes buffered before first parsed operation;
- parsing throughput and peak parser memory;
- diagnostic offset and error-span length;
- time to first valid edit;
- time to final verified state;
- provider cost.

#### Precision and runtime preservation

- intended resources changed;
- unrelated resources or hidden fields changed;
- identity churn after editing;
- reference or ownership breakage;
- invalid intermediate states exposed;
- transaction rollback success;
- undo steps per user-level edit;
- stale-target failures under concurrent modification.

#### Maintainability

- independent schema definitions;
- source locations changed for a new node type;
- schema drift defects found by mutation tests.

### 7.5 Ablations

- remove isolated-draft application;
- remove final graph validation;
- expose structural fields;
- use model-generated IDs;
- replace stable IDs with array-index paths;
- split one batch into per-operation calls;
- remove schema-derived child routing;
- remove exact text preconditions;
- omit the relevant-view projection and send the full document.
- replace short IDs with full structural traversal paths;
- replace the line-oriented grammar with typed operation objects;
- buffer the complete edit before parsing versus incrementally parse into an isolated draft.

### 7.6 Statistical Analysis

Use paired task instances across protocols. For binary success, report confidence intervals and a paired test such as McNemar’s test or a hierarchical logistic model. For skewed cost and latency, report medians, percentile distributions, bootstrap confidence intervals, and paired effect sizes. Model, task family, project size, and run should be treated as explicit factors.

> **[EVIDENCE NEEDED P1-E7]** Define the power analysis after a pilot estimates the expected success-rate difference and within-task variance.

## 8. Expected Results and Falsifiable Hypotheses

- **H1:** REST-Edit will improve executable final-state success for multi-resource edits relative to fine-grained tools.
- **H2:** REST-Edit will reduce target-expression length, model turns, and output tokens as edit span increases.
- **H3:** REST-Edit will expose a complete parseable operation earlier and with less buffering than a full structured operation array.
- **H4:** Full-document regeneration will have lower protocol-learning burden but more unintended mutations, identity churn, and output cost.
- **H5:** Routing precise edits through the established runtime model will preserve unrelated modeled state better than reconstruction.
- **H6:** Short stable identifiers will outperform long traversal paths after structural change.

These hypotheses are intentionally falsifiable. If a parallel tool interface achieves equal correctness at lower total cost, the paper should report that boundary rather than claim universal superiority.

## 9. Limitations and Threats to Validity

- The protocol is specialized for schema-addressable editing and may not suit free-form pixel manipulation.
- A compact operation language shifts complexity from the model to the executor and schema.
- Stable identities do not by themselves preserve stale user intent under concurrency.
- “Streaming-friendly” is a grammar property until the end-to-end delta path is wired and benchmarked.
- Generic JSON operation arrays may benefit more from provider-enforced structured output than a text DSL; the comparison is model- and API-dependent.
- Provider-specific structured-output and prompt-caching behavior can dominate cost.
- Benchmark verifiers may reward structural compliance without measuring aesthetic quality.
- A product codebase can bias task selection toward the proposed method.
- The current operation vocabulary does not expose every professional NLE concept.
- The implementation uses an existing CRDT and does not claim a new convergence algorithm.

## 10. Engineering and Evidence Watchlist

| ID | What must be collected or resolved during development | Why it matters |
|---|---|---|
| P1-E1 | Cross-model benchmark results | Required for every performance claim |
| P1-E2 | Raw state versus VML projection size | Establishes representation benefit |
| P1-E3 | Adversarial streaming-chunk parser suite | Supports incremental parsing claim |
| P1-E4 | Atomicity and fault-injection suite | Supports execution claim |
| P1-E5 | Cached and uncached token traces | Prevents misleading cost claims |
| P1-E6 | Diverse executable task dataset | Supports external validity |
| P1-E7 | Power analysis and repeated-run count | Supports statistical validity |
| P1-I1 | Wire model field deltas into isolated incremental parsing | Converts grammar property into system behavior |
| P1-I2 | Return or bind host-generated IDs within a batch | Needed when later operations address newly created resources |
| P1-I3 | Protocol versioning and capability negotiation | Needed when schemas and clients evolve |
| P1-I4 | Machine-readable error taxonomy with failing operation index | Needed for principled repair loops |
| P1-I5 | Explicit limits for batch size, nesting, and generated content | Needed for resource and abuse safety |
| P1-X1 | Precise edit versus full replacement state-preservation study | Tests the central motivation |
| P1-X2 | Fair REST-Edit versus JSON Patch representation study | Bounds adjacent-format claims |
| P1-X3 | Schema-convergence mutation study | Tests conceptual compression |
| P1-X4 | Same-executor protocol baselines | Isolates interface effects |
| P1-X5 | Frozen model snapshots and prompt manifests | Reproducibility |
| P1-X6 | Short IDs versus long paths under structural change | Tests semantic addressing |
| P1-X7 | Streaming parser versus full-buffer parser | Tests latency and memory benefit |

> **[IMPLEMENTATION GAP P1-I2]** Newly created IDs are host-owned, but later operations in the same or next plan need a principled way to address them. Consider batch-local symbolic references or an explicit creation-result map. Do not solve this by accepting arbitrary model-generated persistent IDs.

> **[IMPLEMENTATION GAP P1-I3]** Define protocol and schema version behavior before public deployment. A published protocol needs a compatibility story, not only a parser.

> **[IMPLEMENTATION GAP P1-I4]** Errors should identify phase, operation index, resource, invariant, and repairability. This supports evaluation and model repair without leaking hidden storage structure.

## 11. Planned Figures and Tables

1. **Figure 1:** user intent → agent-visible VML → REST-Edit program → draft execution → validation → CRDT commit.
2. **Figure 2:** one schema driving projection, parsing, routing, creation, validation, and serialization.
3. **Figure 3:** full replacement risk surface versus a short semantic precise edit.
4. **Figure 4:** model-output deltas → incremental parser → isolated draft → atomic commit.
5. **Table 1:** representation-level comparison of REST-Edit, structured operation arrays, JSON Patch, and full regeneration.
6. **Table 2:** task taxonomy and dataset scale.
7. **Table 3:** correctness, generation, parsing, and latency by model and protocol.
8. **Figure 5:** tokens and turns versus affected-resource span.
9. **Figure 6:** time-to-first-operation and parser buffer under streamed generation.
10. **Table 4:** ablations for short addressing, grammar, validation, batching, and schema derivation.

## 12. Submission Readiness Criteria

The paper should not be submitted until:

- RefineCut is cited and either evaluated directly or its non-comparable training contribution is clearly separated;
- structured operation, full-regeneration, strongest parallel tool-calling, and any JSON Patch baselines share the same executor where applicable;
- the incremental parser is connected to actual model field deltas or the paper consistently limits the claim to grammar streamability;
- a precise-edit versus full-replacement preservation experiment is complete;
- cached and uncached token accounting is separated;
- the task dataset and verifier are independently reviewed for proposal bias;
- failure cases are categorized rather than hidden by aggregate success;
- all “first,” “novel,” “streaming,” and “less data” statements are tied to evidence.

## References

1. Roy T. Fielding. 2000. *Architectural Styles and the Design of Network-based Software Architectures*. https://ics.uci.edu/~fielding/pubs/dissertation/top.htm
2. Paul C. Bryan and Mark Nottingham. 2013. *RFC 6902: JavaScript Object Notation (JSON) Patch*. https://www.rfc-editor.org/info/rfc6902/
3. Paul C. Bryan, Kris Zyp, and Mark Nottingham. 2013. *RFC 6901: JavaScript Object Notation (JSON) Pointer*. https://www.rfc-editor.org/info/rfc6901/
4. Paul Hoffman and James Snell. 2014. *RFC 7396: JSON Merge Patch*. https://www.rfc-editor.org/info/rfc7396/
5. Martin Kleppmann and Alastair R. Beresford. 2017. *A Conflict-Free Replicated JSON Datatype*. https://arxiv.org/abs/1608.03960
6. Martin Kleppmann, Dominic P. Mulligan, Victor B. F. Gomes, and Alastair R. Beresford. 2021. *A Highly-Available Move Operation for Replicated Trees*. https://doi.org/10.1109/TPDS.2021.3118603
7. Shunyu Yao et al. 2022. *ReAct: Synergizing Reasoning and Acting in Language Models*. https://arxiv.org/abs/2210.03629
8. Timo Schick et al. 2023. *Toolformer: Language Models Can Teach Themselves to Use Tools*. https://arxiv.org/abs/2302.04761
9. Minghao Li et al. 2023. *API-Bank: A Comprehensive Benchmark for Tool-Augmented LLMs*. https://arxiv.org/abs/2304.08244
10. Yujia Qin et al. 2023. *ToolLLM: Facilitating Large Language Models to Master 16000+ Real-world APIs*. https://arxiv.org/abs/2307.16789
11. John Yang et al. 2024. *SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering*. https://arxiv.org/abs/2405.15793
12. Bryan Wang et al. 2024. *LAVE: LLM-Powered Agent Assistance and Language Augmentation for Video Editing*. https://arxiv.org/abs/2402.10294
13. Bekzat Tilekbay et al. 2024. *ExpressEdit: Video Editing with Natural Language and Sketching*. https://arxiv.org/abs/2403.17693
14. Yuzhi Li, Haojun Xu, and Fang Tian. 2025. *From Shots to Stories: LLM-Assisted Video Editing with Unified Language Representations*. https://arxiv.org/abs/2505.12237
15. *EditDuet: A Multi-Agent System for Video Editing*. 2025. https://arxiv.org/abs/2509.10761
16. Lecheng Yan et al. 2026. *Crayotter: Traceable Multi-Agent Workflows for Long-Form Video Editing*. https://arxiv.org/abs/2606.07636
17. *VideoAgent: All-in-One Framework for Video Understanding and Editing*. 2026. https://arxiv.org/abs/2606.23327
18. *Plans You Can Check: Verifier-Grounded Learning of an Open-Weight Planner for Executable Video-Editing*. 2026. https://openreview.net/forum?id=uBD1X4u4AD
