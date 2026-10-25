# MOLT Kernel and Runtime Architecture
## Technical Specification and Whitepaper Outline

**Status:** Architecture specification for MOLT Kernel v0.1-alpha2, MOLT-WORLD v0.1, and MOLT-OBSERVATORY v0.1  
**Repository:** `jasethepeople/molt`  
**Normative language:** The terms **MUST**, **MUST NOT**, **REQUIRED**, **SHOULD**, **SHOULD NOT**, and **MAY** are to be interpreted as normative requirements for conforming implementations.

---

## Abstract

MOLT is a protocol substrate for persistent computational agents. It separates four concerns that are frequently conflated in multi-agent systems:

1. **Kernel:** determines what is valid.
2. **World:** determines what agents can do.
3. **Observatory:** determines what can be observed and reconstructed.
4. **Experiment:** defines controlled initial conditions and measurements.

The canonical source of truth is an append-only, cryptographically authenticated event history. Materialized world state, memory indexes, snapshots, and Observatory reports are derived artifacts. This design makes restart, replay, audit, and cross-run comparison first-class properties rather than after-the-fact instrumentation.

The present implementation is deliberately local. It provides cryptographic identities, immutable signed events, causal history validation, durable JSONL storage, deterministic replay, conserved resources, delegated leases, persistent memory, replaceable agent shells, capability authorization, communication, provenance queries, and experiment manifests. Networking, distributed consensus, autonomous replication, and prescribed social organization are intentionally outside the current protocol boundary.

---

## 1. Scope and design goals

### 1.1 In scope

This specification covers:

- Identity and signature model.
- Canonical event representation and content addressing.
- Append-only event storage and history validation.
- Causal ordering and deterministic replay.
- World-state snapshots and state hashes.
- Resource pools, leases, delegation, and consumption.
- Experiment-boundary metadata and policy integrity.
- Persistent agent memory.
- Replaceable agent shells and capability execution.
- Message and action semantics.
- Read-only historical observation.
- Reproducible experiment provenance and analysis.

### 1.2 Out of scope

The following are not defined by the current implementation:

- Peer-to-peer transport.
- Network membership or discovery.
- Distributed consensus or conflict resolution between replicas.
- Global clocks or trusted wall-clock time.
- Autonomous replication or evolutionary selection.
- A mandatory organizational model for agents.
- An objective theory of intelligence, consciousness, agency, or emergence.
- A normative interpretation of communication patterns.

A future protocol version MAY define these capabilities, but it MUST preserve explicit versioning and MUST NOT silently change Alpha2 event or replay semantics.

### 1.3 Design principles

**Event history is authoritative.** Derived state MUST be rebuildable from accepted events.

**Mutation passes through validation.** No world operation may mutate canonical state except through a valid kernel event.

**Observation is not intervention.** The Observatory MUST NOT influence the population unless a separately authorized intervention policy explicitly permits it.

**Mechanisms are not social policy.** MOLT exposes identities, resources, memory, communication, and capabilities; it does not require teams, leaders, roles, markets, voting, cooperation, or competition.

**Reproducibility precedes interpretation.** Raw observations, measurements, statistical summaries, hypotheses, and interventions MUST remain distinguishable artifacts.

---

## 2. Layered architecture

```text
┌──────────────────────────────────────────────────────────────┐
│ Experiment package                                            │
│ objective • population • capabilities • resources • policies  │
└───────────────────────────────┬──────────────────────────────┘
                                │ immutable initial conditions
┌───────────────────────────────▼──────────────────────────────┐
│ MOLT-WORLD v0.1                                               │
│ agents • shells • memory • capabilities • actions • messages  │
└───────────────────────────────┬──────────────────────────────┘
                                │ signed events
┌───────────────────────────────▼──────────────────────────────┐
│ MOLT KERNEL v0.1-alpha2                                      │
│ identity • event validity • causality • resources • replay     │
└───────────────────────────────┬──────────────────────────────┘
                                │ canonical history and state
┌───────────────────────────────▼──────────────────────────────┐
│ MOLT-OBSERVATORY v0.1                                         │
│ queries • lineage • provenance • replay-at-event • analysis    │
└──────────────────────────────────────────────────────────────┘
```

### 2.1 Kernel boundary

The kernel owns validity. It validates event integrity, signatures, history structure, sequence order, causal references, resource transitions, and replay behavior. The kernel does not know what an experiment named EMERGENCE-001 means.

### 2.2 World boundary

The world owns inhabitants and action translation. A shell observes a `WorldContext`, decides on an `Action`, and submits that action to a runtime adapter. The adapter creates a signed event, validates and commits it, and only then applies derived effects such as message delivery or memory indexing.

### 2.3 Observatory boundary

The Observatory owns no canonical mutation authority. It reads event stores and memory indexes, reconstructs state, and emits JSON measurements. Its outputs are evidence for a researcher or a future explicitly authorized intervention; they are not implicit control signals.

### 2.4 Experiment boundary

An experiment is an independent package of initial conditions, objective, population, capability assignment, resource policy, audit policy, intervention policy, privacy policy, termination conditions, and analysis definition. Experiment metadata is content-addressable and immutable once frozen for a baseline series.

---

## 3. Identity and cryptography

### 3.1 Identity model

Each agent has a cryptographic identity represented by an agent identifier and an Ed25519 keypair. The private key signs event content. The public key is used to verify signatures. Identity material is independent of the agent shell: replacing a shell MUST NOT change the identity or canonical event history.

The implementation uses hex-encoded key and signature representations at the API boundary. A conforming implementation MUST treat private keys as secret and MUST NOT serialize them into event payloads, logs, or Observatory output.

### 3.2 Content addressing

MOLT uses BLAKE3 over canonical unsigned event content. The resulting lowercase hexadecimal digest is both:

- `content_hash`, and
- `event_id`.

An event with an `event_id` different from its `content_hash` is invalid.

### 3.3 Signature scope

The signature covers exactly the canonical unsigned event object. It excludes `signature` and `content_hash` themselves. Verification therefore checks two independent properties:

1. The event bytes match the declared content hash.
2. The signature authenticates those bytes for the claimed actor identity.

A signature over a different serialization, a mutated payload, or altered metadata MUST fail verification.

---

## 4. Event model and wire format

The implemented event structure is:

```json
{
  "event_id": "<blake3 hex>",
  "event_type": "MESSAGE_SENT",
  "protocol": "MOLT/0.1",
  "actor_id": "A17",
  "logical_clock": 12,
  "sequence_number": 42,
  "timestamp": "2026-09-18T00:00:00Z",
  "parents": ["<event-id>"],
  "payload": {},
  "evidence_refs": ["<event-id-or-external-ref>"],
  "resource_refs": ["<lease-or-pool-id>"],
  "signature": "<ed25519 signature hex>",
  "content_hash": "<blake3 hex>"
}
```

### 4.1 Canonical unsigned content

The signed and hashed object contains:

```text
actor_id
 event_type
evidence_refs
logical_clock
parents
payload
protocol
resource_refs
sequence_number
timestamp
```

Canonical JSON serialization MUST be deterministic with respect to object key ordering and representation. Arrays retain their declared order. Implementations MUST NOT sign a pretty-printed or runtime-specific representation.

### 4.2 Event identifiers and timestamps

`event_id` is a content identifier, not a database sequence. `sequence_number` expresses append-stream order. `logical_clock` expresses causal metadata. `timestamp` is retained as event metadata and MUST NOT be treated as authoritative physical time for causality.

### 4.3 Event categories

The current runtime emits or recognizes categories including:

- Genesis and identity events.
- `RESOURCE_POOL_CREATED`.
- `RESOURCE_LEASE_ISSUED`.
- `RESOURCE_CONSUMED`.
- `CAPABILITY_INVOKED`.
- `MESSAGE_SENT`.
- `MEMORY_RECORDED`.
- Claims, evidence, challenges, and generic proposals.

Event types are extensible strings, but new types MUST define replay behavior, authorization rules, payload schema, and Observatory treatment before entering a protocol version.

---

## 5. History validation and invariants

The append path validates an incoming event against the accepted event prefix. A conforming store MUST reject invalid events rather than append and repair them later.

### 5.1 Core invariants

- **I1 Identity:** the actor signature MUST verify against the actor identity.
- **I2 Immutability:** accepted event bytes MUST NOT change.
- **I3 Causal consistency:** non-genesis parents MUST be known; sequence order MUST be valid; parent clocks MUST precede child clocks.
- **I4 No double consumption:** a resource allocation cannot be consumed twice.
- **I5 Delegation constraints:** a sub-lease cannot exceed parent scope or authority.
- **I6 Lineage authorization:** authoritative lineage operations require the applicable cryptographic authorization.
- **I7 Read-only Observatory:** Observatory queries have no canonical mutation authority.
- **I8 Conservation:** ordinary events cannot create conserved resources without explicit creation authority.
- **I9 Durable history:** reopening a durable store recovers the same bytes and stream order.
- **I10 Deterministic replay:** equal accepted histories produce equal world state.
- **I11 Snapshot equivalence:** a snapshot hash equals the replay-derived state hash.
- **I12 Resource conservation:** for every lease, `amount = consumed + available + outstanding_subleases`.
- **I13 Lease propagation:** delegated lease scope and quantity remain within recursive parent authority.
- **I14 Experiment provenance:** experiment metadata is immutable and content-addressable.
- **I15 Policy integrity:** audit, intervention, and privacy policy changes require explicit versioned events.

### 5.2 Rejection behavior

A rejected event MUST NOT alter the event store, materialized state, memory index, inboxes, or resource state. Errors SHOULD identify the event and invariant category without exposing secret key material.

---

## 6. Storage and durability

### 6.1 EventStore interface

The kernel exposes an event-store abstraction with operations equivalent to:

```text
append(event)
get(event_id)
parents(event_id)
stream(agent_id, from_sequence, to_sequence)
all()
snapshot()
```

Two implementations are currently provided:

- An in-memory store for tests and local execution.
- A durable JSONL store for restart and archival behavior.

### 6.2 Durable JSONL semantics

The durable store appends one serialized event per line, validates the full existing history before accepting a new event, writes the event, appends a newline, and synchronizes data before returning success. Opening a store reads and deserializes its content, allowing malformed records to fail at reopen rather than silently disappear.

A production implementation SHOULD add atomic rotation, checksummed segments, indexing, retention policy, and concurrent writer coordination. Those additions MUST preserve the logical append-only history and replay result.

### 6.3 Archive integrity

A run archive SHOULD retain:

- Canonical event JSONL.
- Memory JSONL.
- Experiment configuration.
- Run manifest.
- Observatory verification report.
- Final state snapshot and hash.
- Analysis output.

Archive hashes belong in the run manifest so a later reader can distinguish a changed archive from a changed interpretation.

---

## 7. Deterministic world state and replay

### 7.1 WorldState

The current derived state contains:

```text
resources: ResourceState
 event_count: u64
last_event_id: Option<String>
event_types: BTreeMap<String, u64>
```

Resource transitions are applied during replay. Event counters and type counts are derived from the same accepted history.

### 7.2 Replay algorithm

A replay implementation MUST:

1. Copy and order events by `sequence_number`.
2. Require contiguous sequence numbers.
3. Verify event integrity before applying it.
4. Apply resource transitions and other deterministic reducers.
5. Update derived counters and last-event metadata.
6. Return an error on any invalid event or transition.

Replay MUST NOT consult agent shell behavior, external network state, current wall time, or Observatory output.

### 7.3 State hashes and snapshots

The state hash is BLAKE3 over canonical JSON serialization of `WorldState`. A `StateSnapshot` contains the state and its hash. A snapshot is a derived observation, not a second mutation authority.

The defining reproducibility property is:

```text
replay(history) == replay(history after restart)
state_hash(replay(history)) == recorded_snapshot.state_hash
```

### 7.4 Historical state

The Observatory can replay a prefix through a target event. Historical queries MUST use the same reducer and validation rules as present-state replay. A historical hash is therefore a hash of a deterministic prefix, not an analyst-generated approximation.

---

## 8. Resource model

### 8.1 Pools

A resource pool establishes explicit creation authority, resource type, capacity, and ownership context. Conserved capacity cannot be introduced by an ordinary agent action.

### 8.2 Leases

A lease delegates a bounded amount of a resource to a grantee. A lease can be delegated into sub-leases only within the remaining authority of its parent.

For a lease:

```text
available = amount - consumed - outstanding_subleases
```

The equivalent conservation identity is:

```text
amount = consumed + available + outstanding_subleases
```

### 8.3 Consumption

Consumption is an event-backed transition. The runtime MUST NOT decrement a resource counter before the corresponding event is accepted. A consumption reference can be accepted only once, and consumption exceeding available authority MUST be rejected.

### 8.4 Resource archaeology

Because allocations, transfers, delegation, and consumption are canonical events, the Observatory can reconstruct resource flow without relying on a mutable dashboard. Analysis MAY correlate resource concentration with communication or outcomes, but it MUST label that result as an observation or statistical pattern rather than a causal conclusion.

---

## 9. Experiment provenance and policy

`ExperimentContext` binds together:

- Experiment identifier.
- Objective description.
- Participants.
- Initial state hash.
- Resource scope.
- Capability scope.
- Audit level.
- Intervention policy.
- Privacy scope.
- Termination condition.
- Genesis event.

Its canonical bytes and BLAKE3 hash provide an immutable experiment identity.

### 9.1 Policies

The implementation distinguishes:

- Audit: minimal, actions, actions-and-evidence, or full observable trace.
- Intervention: human controlled, human pause only, emergency stop only, or autonomous.
- Privacy scope: public, participant, institution, private, or sealed.
- Termination: event count, explicit termination, or timestamp condition.

A policy is metadata governing an experiment boundary. It is not a hidden runtime override. A policy change MUST be visible as a new event or new experiment version.

### 9.2 Baseline freeze

For EMERGENCE-001 v0.1, the frozen configuration includes genesis, objective, population, capabilities, resources, policies, and analysis definitions. The configuration hash is recorded in `freeze.json`. Baseline run identifiers are reserved as:

```text
E001-baseline-001
E001-baseline-002
E001-baseline-003
E001-baseline-004
```

---

## 10. MOLT-WORLD runtime

### 10.1 Agent shell contract

A shell implements:

```rust
trait WorldAgent {
    fn agent_id(&self) -> &str;
    fn observe(&self, context: &WorldContext) -> Result<Observation, AgentError>;
    fn decide(&mut self, observation: &Observation) -> Result<Action, AgentError>;
    fn receive(&mut self, message: Message) -> Result<(), AgentError>;
}
```

The shell is replaceable. Identity, memory, canonical events, and resource authority remain outside the shell and survive replacement.

### 10.2 Observation

A `WorldContext` contains:

- World identifier.
- Current replay-derived state hash.
- Visible events.
- Available capabilities.
- Available resource leases.

An observation MAY include claims with evidence references. The context is a view; it is not permission to mutate state.

### 10.3 Actions

The current action algebra includes:

- `UseCapability { capability_id, input }`.
- `SendMessage(message)`.
- `Remember { kind, content }`.
- `Propose { event_type, payload }`.

The runtime resolves an action into an event type and payload, validates authorization, invokes a capability where required, creates a signed event, appends it, and only then applies derived effects.

### 10.4 Capabilities

A capability has an identifier, interface, version, requirements, and authorized-agent set. Capability execution returns a serializable `CapabilityResult` containing:

- Capability identifier.
- Agent identifier.
- Output value.
- Consumed resource references.
- Evidence references.

The result and evidence references are embedded in the signed `CAPABILITY_INVOKED` event. An unauthorized capability MUST fail before commit.

The executor MUST NOT directly mutate canonical world state. Any persistent effect MUST be represented by an accepted event.

### 10.5 Messages

Messages contain message ID, sender, recipient, kind, body, and evidence references. Supported kinds include send, request, proposal, challenge, evidence, offer, accept, and reject. A sent message becomes a canonical event before delivery to the recipient inbox.

### 10.6 Memory

Memory records are persistent derived records associated with an agent and source event. Current categories include episodic, semantic, evidence, relationship, and lineage memory. Memory is not canonical truth: its provenance and source event MUST remain inspectable.

A shell restart MAY replace the reasoning implementation while preserving identity and memory. Memory mutation SHOULD be coupled to a committed `MEMORY_RECORDED` event.

### 10.7 Commit ordering

The required ordering is:

```text
observe
  → decide
  → authorize and validate
  → execute capability if needed
  → construct signed event
  → append and commit
  → apply derived effect
```

Derived effects MUST NOT become visible as committed world state before the event append succeeds.

---

## 11. MOLT-OBSERVATORY

### 11.1 Read-only contract

The Observatory reads canonical events, replay-derived state, and memory indexes. It MUST NOT append events, consume resources, alter agent memory, deliver messages, or change capability authorization as a side effect of a query.

### 11.2 Query surface

The current CLI and query API provide:

- History, optionally filtered by agent.
- Event-type filtering.
- Ancestor and descendant lineage reconstruction.
- Memory inspection.
- Capability invocation history.
- Resource pools and leases.
- Claims, evidence, and challenges.
- Replay at a target event.
- Historical state hash.
- History verification.

All CLI results are machine-readable JSON. Errors are returned at the command boundary rather than serialized as misleading successful results.

### 11.3 Four graph views

**Historical graph:** event parents, observations, proposals, results, and messages.

**Epistemic graph:** claims, supporting evidence, challenges, and experiments.

**Resource graph:** pools, leases, delegated leases, consumption, and remaining authority.

**Relationship graph:** measured interaction edges derived from messages. The runtime does not prescribe a topology.

### 11.4 Observation versus interpretation

The Observatory SHOULD emit raw records and measurements separately from hypotheses. For EMERGENCE-001, the analysis pipeline reports messages, contacts, reciprocity, density, centralization, claims, evidence, challenges, memory, resource flow, and observed signals. It MUST NOT convert “agents communicated” into a claim of social cognition or convert “agents formed a group” into a claim of emergent intelligence without an explicit follow-up methodology.

---

## 12. EMERGENCE-001 reference experiment

EMERGENCE-001 is an independent experiment package, not kernel semantics.

### 12.1 Initial conditions

The baseline contains twelve persistent agents with controlled heterogeneity across shell families and capabilities. Capabilities include mathematics, physics, simulation, communication, and document analysis. The configuration specifies capabilities without assigning social roles.

### 12.2 Objective

The baseline objective is a structurally valid bridge spanning 100 meters under specified load, material, geometry, deflection, stability, and evidence constraints. The objective requires an external deterministic validator. “Success” is not the only measured outcome; independent work, communication, disagreement, correction, resource flow, and failure are all valid observations.

### 12.3 Controlled variables

The baseline fixes:

- Population size.
- Shell and capability assignment.
- Finite resource pools.
- Persistent memory policy.
- Communication policy.
- Audit and privacy policy.
- Intervention policy.
- Termination conditions.
- Analysis definition.

Future perturbations SHOULD vary one dimension at a time, such as memory persistence, capability diversity, resource distribution, communication constraints, shell family, population size, or objective difficulty.

### 12.4 Run manifest

Each run SHOULD record:

```json
{
  "run_id": "E001-baseline-001",
  "experiment": "EMERGENCE-001",
  "experiment_version": "0.1",
  "configuration_hash": "...",
  "genesis_hash": "...",
  "kernel": "molt-kernel-0.1-alpha2",
  "world": "molt-world-0.1",
  "observatory": "molt-observatory-0.1",
  "shells": {},
  "capabilities": {},
  "started_at": "...",
  "ended_at": "...",
  "event_root": "...",
  "event_archive_hash": "...",
  "memory_archive_hash": "...",
  "final_state_hash": "...",
  "analysis_version": "0.1",
  "observatory_feedback_to_world": false
}
```

### 12.5 Cross-run comparison

A comparison tool MUST reject analyses with different configuration hashes or duplicate run IDs. It MAY compute descriptive minima, maxima, means, distributions, and repeated-pattern counts. It MUST preserve run-level measurements and MUST NOT feed comparison results back into baseline execution.

---

## 13. Security model

### 13.1 Threats addressed

The current design addresses:

- Event payload tampering.
- Forged event signatures.
- Event ID/content hash mismatch.
- Duplicate events.
- Unknown parents.
- Sequence gaps and reordering.
- Parent-clock violations.
- Resource over-allocation.
- Resource over-consumption.
- Unauthorized capability invocation.
- Accidental Observatory mutation.
- Loss of materialized state after restart.

### 13.2 Threats not addressed

The current local implementation does not solve:

- Private-key compromise.
- Malicious or Byzantine storage operators.
- Network partition or replica conflict.
- Denial of service.
- Sybil resistance.
- Secure deletion under immutable audit.
- Confidential payload encryption.
- Capability code sandboxing.
- Side channels from shell execution.

These require explicit future designs and threat models. They MUST NOT be assumed merely because events are signed.

### 13.3 Capability isolation

A capability executor is trusted by the local runtime boundary to return a result, but its result is not canonical until represented in a signed event. A production deployment SHOULD isolate untrusted capability code with process, filesystem, CPU, memory, and network limits.

---

## 14. Operational model

### 14.1 Local execution

A minimal local workflow is:

```bash
cargo test
cargo run --bin molt -- history --store events.jsonl
cargo run --bin molt -- verify-history --store events.jsonl
```

The experiment analysis workflow is:

```bash
python3 experiments/emergence-001/analysis/analyze.py \
  --config-dir experiments/emergence-001 \
  --events run/events.jsonl \
  --memory run/memory.jsonl \
  --run-id E001-baseline-001 \
  --output run/analysis.json
```

### 14.2 CI/CD

The repository CI workflow runs on pushes and pull requests. It checks Rust formatting, runs all Rust targets, compiles Python analysis tooling, and validates the frozen experiment configuration. Main pushes produce build artifacts. Version tags matching `v*` build release archives, generate SHA-256 checksums, and publish a GitHub Release after tests pass.

### 14.3 Recovery

After process or materialized-state loss:

1. Reopen the durable event store.
2. Validate and load the accepted history.
3. Replay the history.
4. Reconstruct snapshots and derived indexes.
5. Compare the resulting state hash to the recorded manifest or snapshot.
6. Reopen memory and verify source-event references.

A mismatch MUST be treated as a recovery failure or archive-integrity failure, not silently corrected.

---

## 15. Conformance and test plan

A conforming Alpha2 implementation SHOULD include tests for:

- Identity signature verification.
- Forged signatures.
- Genesis validity.
- Canonical hash stability.
- Event immutability.
- Duplicate event rejection.
- Parent and order validation.
- Logical-clock causality.
- Tampered event rejection.
- Durable store reopen.
- Snapshot/replay hash equivalence.
- Lease conservation.
- Delegation constraints.
- Double-consumption rejection.

A conforming World implementation SHOULD include tests for:

- Replaceable shell with persistent identity.
- Authorization before commit.
- Capability result and evidence inclusion.
- Message delivery after event commit.
- Persistent memory with source-event references.
- Restart and replay persistence.

A conforming Observatory SHOULD include tests for:

- Query result equals canonical history.
- Lineage result equals causal parent traversal.
- Resource result equals replay-derived resources.
- Evidence result equals event provenance.
- Replay-at-event equals direct prefix replay.
- State hash equals recorded snapshot hash.
- Queries do not mutate event or memory stores.

---

## 16. Versioning and evolution

### 16.1 Protocol versioning

The `protocol` event field identifies the event protocol version. A future version MUST define migration, compatibility, and replay behavior. It MUST NOT reinterpret an existing event type without an explicit version boundary.

### 16.2 Schema evolution

New optional fields MAY be introduced under a compatible protocol version only if canonicalization remains unambiguous. Required field changes, event semantics changes, resource semantics changes, or replay reducer changes require a new protocol version or an explicit migration event.

### 16.3 Frozen boundary

Alpha2 event canonicalization, identity verification, history validation, resource conservation, replay, and snapshot hashing are frozen for the current experiment series. MOLT-WORLD and Observatory changes MUST preserve those semantics or declare a new protocol/runtime version.

---

## 17. Roadmap and open questions

### Near term

- Execute four independent EMERGENCE-001 baseline runs.
- Retain complete event, memory, manifest, snapshot, verification, and analysis archives.
- Compare repeated behavioral structure without steering the population.
- Add objective-validation evidence for bridge solutions.

### Medium term

- Add richer claim/evidence/challenge schemas.
- Add resource-flow and relationship graph exports.
- Add statistical cross-run analysis with confidence and uncertainty reporting.
- Add capability sandboxing and resource metering.
- Add indexed event storage while preserving JSONL archival compatibility.

### Later protocol work

- Transport and peer discovery.
- Replicated histories and consensus.
- Conflict resolution.
- Cross-world lineage and population inheritance.
- Deliberately controlled feedback experiments.

### Open questions

1. What evidence schema is sufficient for independently validating a computational artifact?
2. Which resource metering model is meaningful across heterogeneous shells?
3. How should privacy policy interact with immutable provenance without creating contradictory deletion semantics?
4. Which repeated interaction patterns survive independent reruns?
5. How can organization be measured without smuggling an organizational ontology into the objective?
6. What protocol is required for replicated event histories without weakening deterministic replay?

---

## Appendix A — Repository mapping

| Specification area | Current implementation |
|---|---|
| Event model and signatures | `kernel/events.rs`, `kernel/identity.rs`, `kernel/hash.rs` |
| Genesis and bootstrap | `kernel/genesis.rs` |
| History invariants | `kernel/invariants.rs`, `docs/invariants.md` |
| Durable storage | `kernel/store.rs` |
| Resources and leases | `kernel/resources.rs` |
| Replay and snapshots | `kernel/world.rs` |
| Experiment metadata | `kernel/experiment.rs` |
| Agent actions and messages | `kernel/agent.rs` |
| Persistent memory | `kernel/memory.rs` |
| World runtime | `kernel/runtime.rs`, `docs/world.md` |
| Observatory queries | `kernel/observatory.rs`, `docs/observatory.md` |
| CLI | `src/bin/molt.rs` |
| EMERGENCE-001 | `experiments/emergence-001/` |
| CI/CD | `.github/workflows/ci.yml` |

## Appendix B — Minimal correctness claim

For an accepted event history `H`, a conforming local implementation aims to provide:

```text
verify(H) = true
replay(H) = S
hash(S) = h
snapshot(H).state_hash = h
reopen(store(H)).all() = H
Observatory(H).read_only = true
```

This is a determinism and provenance claim. It is not a claim that the resulting agents are intelligent, conscious, social, autonomous, or successful. Those are empirical questions for controlled experiments, not assumptions encoded by the kernel.
