Comprehensive Parlia-on-Reth Road-map  
(ordered – each milestone unblocks the next)

──────────── 0. Pre-work: “plumbing” layer (ongoing) ────────────  
• Finish snapshot-advance in the header validator (generic header bound)  
• Replace in-memory cache with MDBX snapshot provider + LRU wrapper  
• Register the Parlia header validator in the pipeline builder so header sync already enforces turn-based difficulty & timestamp/interval rules.  
(ETA 2–3 days – 100 % Rust side, no change to execution logic)

──────────────── 1. Fast-Finality (in-block) ────────────────  
Goal Block N can finalise N-200/N-500/N-1000 (epoch checkpoint) once ≥⅔ validators sign the attestation stored in `extraData`.

Tasks  
1.1 Parse `VoteAttestation` from header (already ported in `vote.rs`).  
1.2 Extend validator:  
 • verify attestation target==parent, source==highest-justified, bitset ≤ validators, aggregated BLS sig passes.  
 • maintain per-checkpoint LRU of justified/finalised status.  
1.3 Snapshot::apply: update `vote_data` and mark sources/targets finalised.  
1.4 Expose `fast_finalised_block()` helper for sync / JSON-RPC.  
Tests  
• Unit: craft minimal 3-validator epoch, assert finality after 2 votes.  
• Cross-test with Go node re-executing the same chain.  
(ETA 1 week – needs BLST bindings already present via `blst` crate)

──────────────── 2. On-chain Validator-Set & Turn-Length ─────────  
Goal At epoch boundary the first block’s `extraData` includes the new validator array and (since Bohr) `turnLength`.  Node must:  
 • verify bytes length & ABI rules (pre-/post-Luban),  
 • update snapshot.validators / .turn_length.

Tasks  
2.1 Port `parse_validators_{before,after}_luban` and `parse_turn_length` from `zoro_reth::bsc::consensus`.  
2.2 Snapshot::apply: when `next_header.number % epoch == 0` load the new set & turnLength, rebuild `validators_map`, reset proposer history.  
2.3 Validate header extra-data length matches epoch/non-epoch rules.  
2.4 Provide JSON-RPC method `parlia_getSnapshot(number)` (debug aid).  
(ETA 1 week)

──────────────── 3. Proposer-Slash & Double-Sign ────────────────  
Goal Detect equivocation (same height, different hash by same signer) and out-of-turn spamming, then produce slashing `systemTx`.

Tasks  
3.1 Keep recent proposer map in provider (block → signer, already exists).  
3.2 On new block, check if `(height, signer)` already seen with different hash → raise `SlashEvent`.  
3.3 Implement minimal `systemTx` builder that mints slash amount to burn address and marks validator penalised (details copied from Go impl).  
3.4 Post-Execution hook (see milestone 5) checks reward/slash correctness.  
(ETA 1½ weeks – requires execution hooks)

──────────────── 4. Engine-API glue (CL-EL hand-shake) ──────────  
Goal Reth’s Engine API layer (`newPayloadV3`, `forkChoiceUpdatedV2`, etc.) invokes Parlia consensus for payload validation & fork-choice updates.

Tasks  
4.1 Implement Parlia `FullConsensus` trait:  
 • use header validator (pre-execution)  
 • call pre-execution hooks (see next milestone)  
 • after EVM, call post-execution validation (gas-used, reward caps).  
4.2 Provide `fork_choice_state()` that prefers fast-finalised branch.  
4.3 Plug into `ExecutionStage::new_with_executor(...)` builder path.  
4.4 Wire CL-timestamp rules (slot → timestamp) to Parlia’s block-interval.  
(ETA 1 week; parallel to milestone 3)

──────────────── 5. Pre- / Post-Execution hooks ────────────────  
Goal Port `pre_execution.rs` & `post_execution.rs` from zoro-reth so block execution enforces all Parlia/BSC-specific state changes:

Pre-execution  
 • timestamp/back-off validation (Ramanujan fork)  
 • fast-finality attestation verification (if Plato active)  
 • seal (ECDSA) + proposer inturn/out-of-turn difficulty check

Post-execution  
 • correct block reward calculation (blockInterval-dependent)  
 • validator‐slashing reward / burned amount  
 • gas-limit divisor change at Lorentz  
 • call-depth etc. for Bohr/Pascal specific rules

Steps  
5.1 Create `bsc_execution` crate wrapping REVM with Parlia hooks.  
5.2 Translate zoro-reth logic; adapt to alloy-primitives types.  
5.3 Register hooks in `ExecutionStage` via generic `ConfigureEvm`.  
5.4 Golden-test: execute known Maxwell epoch on both Go BSC and Rust, compare state roots & balances.  
(ETA 2–3 weeks – largest chunk)

──────────────── 6. Snapshot Persistence & Caching ─────────────  
Goal Same behaviour as Go: snapshot stored every 1 024 blocks; memory keeps last 1 280 (≈ max epoch).

Tasks  
6.1 Finish MDBX table wrapper (`ParliaSnapshots` already stubbed).  
6.2 Provider loads snapshot for parent if not in memory.  
6.3 During stage unwind, delete snapshots > unwind point.  
6.4 Bench cold-start sync vs. Go.

──────────────── 7. Network / Dev-net QA ───────────────────────  
Goal Three-validator dev-net reaches finality; hot-upgrade from Bohr→Lorentz→Maxwell.

Tasks  
• Genesis builder, docker compose with two Rust + one Go node.  
• CI job runs 2 000 blocks, asserts equal state root on every node.  
• Stress-test with random double-sign / out-of-turn attempts.

──────────────── 8. Production hardening ───────────────────────  
• Prometheus metrics (finality lag, attestation failures, slash events).  
• Snapshot LRU instrumentation.  
• Doc-book chapter “Running BSC with Reth”.

Approximate timeline = ~8–9 weeks total (assuming 1 FTE).