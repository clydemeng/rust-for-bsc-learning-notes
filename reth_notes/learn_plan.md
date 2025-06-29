# Reth Learning Plan  

A roadmap to systematically understand every major subsystem of the `reth` code-base. The plan is roughly sequential but you can reorder small items as needed.  
Whenever you finish a step, capture your insights in sibling notes (e.g. `reth_notes/storage.md`) and commit them.  

---

## 0. Orientation (½ day)
1. Read `README.md`, `book/`, and `docs/architecture/*.md` to grasp the **high-level goals** and vocabulary ("stage", "pipeline", "primitives", …).  
2. Draw a one-page diagram of the whole node: *network → downloader → pipeline → execution → rpc*.  

Deliverable: `reth_notes/overview.md` with diagram.

---

## 1. Core Primitives (½ day)
Crates: `primitives`, `primitives-traits`, `ethereum`.

Tasks:
- Skim the data-structures: `H256`, `Block`, `Header`, `Receipt`, `Log`, `Address`, etc.  
- Understand how *feature-gated* helper traits (`Sealable`, `Sealed`) encode fork-specific semantics.  
- Trace the `RlpEncodable` / `Decodable` implementations.  

Deliverable: `reth_notes/primitives.md` cheat-sheet.

---

## 2. Chain Configuration (½ day)
Crate: `chainspec` (+ `config`).

Tasks:
- Read `ChainSpec`, `Hardfork`, `ChainInfo` structs.  
- Inspect how fork activation blocks are queried (`active_at`, `spec_id()`).  
- Run `cargo run -p reth-cli -- specs list` to see built-ins.  

Deliverable: `reth_notes/chainspec.md`.

---

## 3. Storage & Database Layer (2 days)
Crates: `storage`, `fs-util`, plus `etl` (external-table loader).

Topics:
- Tables & key-layouts (see `storage/src/table.rs`).  
- **MDBX** usage via the `reth-libmdbx` wrapper + environment management.  
- Batch writers, cursors, and walkers.  
- Snapshotting & compaction.  

Exercise: write a small Rust binary that opens the DB, counts blocks, and prints the latest state root.

---

## 4. Execution Engine (3 days)
Crates: `evm`, `revm` (wrapper), `executor`.

- Understand how `reth` wraps **revm** with `EvmProcessor`.  
- Step through block execution during tests under `crates/executor/tests`.  
- Investigate precompiles registry & gas accounting.  
- Read how receipts are produced (`ReceiptBuilder`).  

Deliverable: `reth_notes/execution.md` with call-graph.

---

## 5. Consensus (2 days)
Crate: `consensus`.

- Explore the `ForkchoiceState`, `ConsensusEngine` trait.  
- Read the built-in *Ethash* & *Beacon* implementations (even if BSC will override later).  
- Follow transition logic between L1 and L2 (merge).  

---

## 6. Pipeline & Stages (3 days)
Crates: `stages`, `pipeline`, `chain-state`.

- Map each stage: `HeaderSync`, `BodySync`, `SenderRecovery`, `Execution`, `Hashing`, `Merkle`.  
- Observe how checkpoints allow resume.  
- Instrument a run with `RUST_LOG=debug` to see stage boundaries.  

---

## 7. Downloader & Sync (2 days)
Crates: `era-downloader`, `net` (peer mgr).

- Study request/response types (`BodiesRequest`, `HeadersResponse`).  
- Learn the *priority task* queue.  
- Simulate slow peer to trigger timeouts.  

---

## 8. P2P Networking (2 days)
Crates: `net`, `tasks`, `tokio-util`.

- Handshake (capabilities, fork id).  
- Sub-protocol multiplexing (ETH, Snap).  
- Peer scoring & disconnection reasons.  

---

## 9. Transaction Pool (1 day)
Crate: `transaction-pool`.

- Bounded Mempool, sub-pools (pending, queued).  
- Replacement & price-bump logic.  

---

## 10. RPC & CLI (1½ days)
Crates: `rpc`, `cli`, `static-file`.

- JSON-RPC method table; subscription engine (WS).  
- CLI opts → node config (`NodeConfig`, `RpcConfig`).  

Exercise: add a custom RPC method that returns DB stats.

---

## 11. Observability (½ day)
Crates: `tracing`, `tracing-otlp`, `metrics`.

- Trace layers, OpenTelemetry exporter.  
- Prometheus endpoint under `/metrics`.  

---

## 12. Testing & Tooling (ongoing)
- Property tests in `e2e-test-utils`.  
- Fixtures generation (`reth/scripts/*`).  
- Fuzz harness via `cargo fuzz`.  

---

## 13. Monthly Upkeep
- Pull upstream `main`, run `cargo xtask check-workspace`.  
- Update notes on breaking changes.

---

### Total Estimated Effort  ≈ 15–17 full-time days (can overlap).

After finishing these sections, you should not only **understand** `reth` but also be ready to start integrating BSC-specific consensus & system contracts. 