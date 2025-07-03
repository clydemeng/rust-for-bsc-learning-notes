# loocapro_reth_bsc — Implementation Plan  

Goal: run a **production-grade** BSC full node that (a) syncs from genesis, (b) tracks the live chain tip, and (c) stays compatible with all present & upcoming hard-forks (incl. Maxwell).  
Deliverables are grouped by layer; tackle them roughly in order, but many can be parallelised.

---

## 0 · Project bootstrap
- [ ] Update `reth` upstream rev & rebase fork if needed.  
- [ ] Pin Rust tool-chain (`rust-toolchain.toml`).  
- [ ] CI workflow: `cargo check`, `cargo test`, `cargo fmt --check`, `cargo clippy --all`.  

---

## 1 · Consensus ‑ ParliaEngine
1. **Engine scaffold**  
   - [ ] Create `parlia_engine.rs` implementing `FullConsensus<BscPrimitives>` and `ForkchoiceState`.  
   - [ ] Wire into `node::consensus` replacing the temporary `EthBeaconConsensus` wrapper.
2. **Header/Seal validation**  
   - [ ] Extra-data length & validator list parsing.  
   - [ ] Step-based proposer rotation (`block.number % epoch`).  
   - [ ] `ECDSA` signature check vs proposer address.  
3. **Validator-set updates**  
   - [ ] Read system-contract events (staking, slash).  
   - [ ] Update in-memory set; persist to `Validators` table.  
4. **Fast-finality**  
   - [ ] Implement `FINALITY_DELAY` rule (usually 15 blocks) to mark blocks as irreversible.  
5. **Unwind logic**  
   - [ ] Handle short re-orgs (< epoch) gracefully.

---

## 2 · ChainSpec & Hard-fork data
- [ ] Extend `hardforks/bsc.rs` to include **Maxwell** (mainnet & testnet activation).  
- [ ] Add **Pectra** placeholder (timestamp TBD).  
- [ ] Update `hardforks/mod.rs` helper trait with `is_maxwell_active_*` helpers.  
- [ ] Bump `chainspec/bsc.json` files accordingly.

---

## 3 · EVM Integration
1. **Spec mapping** (`evm/spec.rs`)  
   - [ ] Map Maxwell → `CANCUN` (temporary) or new `SpecId` when revm adds Prague/Electra.  
   - [ ] Map Pectra → new `SpecId` once supported.
2. **Gas-schedule / opcode deltas**  
   - [ ] Compare BSC fork docs against upstream Ethereum spec for divergences after Luban.  
   - [ ] Implement overrides in `handler.rs` (e.g., system-reward cap introduced in Kepler).  
3. **Pre-compiles**  
   - [ ] Audit list vs `bsc_official` (e.g., `0x65…` SIMD precompile if any) and port missing ones.  
4. **Blob tx / EIP-4844**  
   - [ ] Verify revm path works under Kepler/Haber; adjust blob-gas accounting.

---

## 4 · System contracts
- [ ] Port staking, slash, governance contracts (Go impl in `bsc_official/core/systemcontracts`) to Rust equivalents callable from runtime.  
- [ ] Ensure `evm::executor` pays **system reward** rules introduced in Kepler & later.

---

## 5 · Database & Stages
- [ ] Add any new tables needed (`Validators`, `SlashEvents`).  
- [ ] Ensure `Execution`, `Hashing`, `Merkle` stages are Parlia-aware (e.g., skip uncle rewards).  
- [ ] Implement **Historical Sync**: start pipeline from genesis, save checkpoints.  
- [ ] Implement **Live Sync**: keep downloader running while execution lags.

---

## 6 · Networking
- [ ] ETH sub-protocol is fine; BSC uses same wire formats.  
- [ ] Enforce `minPeerBlockHeight` per Parlia spec.  
- [ ] Throttle peers that send invalid signatures.

---

## 7 · CLI & Config
- [ ] Expose `--chain bsc` and new flags `--epoch-size`, `--system-reward-cap`.  
- [ ] Auto-generate `jwt.hex` so users can attach external tools (traces, etc.).

---

## 8 · Observability & Tooling
- [ ] Prometheus: add Parlia metrics (validator set size, step lag).  
- [ ] Tracing spans for each stage.  
- [ ] `reth-bsc debug validator-set` CLI sub-command.

---

## 9 · Test suite
- [ ] Unit-tests for header validation edge cases.  
- [ ] Integration test: 3-validator dev-net, produce 100 blocks, ensure state roots match Geth.  
- [ ] Fuzz transaction vs revm diff against official BSC node.

---

## 10 · Documentation
- [ ] Update project `README.md` with Maxwell & Pectra notes.  
- [ ] Add `docs/hardforks.md` explaining mapping.

---

### Milestones & Timeline (tentative)
| Week | Milestone |
|------|-----------|
| 1-2  | ParliaEngine header validation & checkpoints |
| 3-4  | Historical sync to block 5 M on testnet |
| 5-6  | Pre-compiles audited & Maxwell added |
| 7-8  | Full mainnet sync catch-up; live head tracking |
| 9    | Performance tuning & release v0.1 |
