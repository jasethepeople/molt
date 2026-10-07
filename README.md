# MOLT Kernel

Experimental protocol substrate for persistent computational agents. The kernel defines mechanisms, not social policy: Ed25519 identities, signed immutable events, deterministic replay, resources, and experiment provenance (crate `molt-kernel`, v0.1.0-alpha.2).

## Features

- **Signed immutable events** — Ed25519 identities, deterministic canonical event bytes, BLAKE3 content addressing (`kernel/identity.rs`, `events.rs`, `hash.rs`)
- **Genesis ceremony validation and causal history checks** (`kernel/genesis.rs`)
- **Append-only event stores** — in-memory and durable JSONL (`kernel/store.rs`)
- **Deterministic world replay** — snapshots and state hashing (`kernel/world.rs`, `runtime.rs`)
- **Resources** — pools, leases, delegated leases, consumption, conservation enforcement (`available = amount - consumed - outstanding_subleases`) (`kernel/resources.rs`)
- **Experiments** — immutable audit, intervention, privacy, capability, and termination metadata (`kernel/experiment.rs`)
- **MOLT-OBSERVATORY v0.1** — read-only query layer over event history and derived state: history, lineage, memory, resources, claims/evidence/challenges, replay, historical state-hash, history verification (`kernel/observatory.rs`, `docs/observatory.md`)
- **MOLT-WORLD v0.1** — runtime layer above the frozen Alpha2 substrate: agents with persistent identities, replaceable shells, persistent memory, authorized capabilities, leases (`docs/world.md`)
- **EMERGENCE-001** — first experiment package: 12-agent controlled population, frozen baseline definition (`experiments/emergence-001/freeze.json`)
- **Invariant + adversarial tests** — `tests/` covers alpha2, invariants, observatory, and world; property tests via proptest

Architectural law: no world operation may mutate state except through a valid kernel event — the event log is the source of truth and state is a derived materialization.

## Tech stack

Rust (edition 2021), blake3, ed25519-dalek, serde/serde_json, thiserror; proptest for dev-dependencies. No vendored core or external services — the kernel is self-contained. Transport/P2P is intentionally out of scope for Alpha2.

## Getting started

```bash
cargo build
cargo test
cargo run --bin molt -- history --store events.jsonl
cargo run --bin molt -- verify-history --store events.jsonl
```

Full CLI usage: `cargo run --bin molt` prints the command list (`history|events|lineage|memory|resources|claims|evidence|challenges|replay|state-hash|verify-history` with `--store`, `--memory`, `--agent`, `--type`, `--at`, `--kind` options).

## Project structure

```
kernel/           # the molt-kernel library (lib path = kernel/mod.rs)
src/bin/molt.rs   # CLI binary
protocol/schemas/ # protocol schemas
docs/             # invariants.md, observatory.md, world.md
experiments/emergence-001/
tests/            # alpha2, invariants, observatory, world
```

## Status

Real, active research project at v0.1.0-alpha.2 (Apache-2.0). Alpha2 adds the deterministic local laboratory; networking/transport is deliberately not yet introduced.
