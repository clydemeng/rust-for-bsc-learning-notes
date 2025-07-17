Below is a concise but fairly exhaustive “status-and-road-map” FAQ that should clarify what Parlia is, what the _current_ `loocapro_reth_bsc` does during historical sync, and how much of zoro-reth’s extra BSC logic has or has **not** been ported yet.

------------------------------------------------------------------
1. What Parlia consensus actually does
------------------------------------------------------------------
(Think of Parlia as “Geth Clique 2.0 + fast-finality + on-chain validator-set”)

• Validator rotation (“turn-based PoA/POS”)
  –  A fixed‐length **epoch** (200 blocks on main-net).  
  –  Within an epoch each validator gets deterministic “turns” (`turn_length`
     == 1 before the Bohr HF; later can be 2,4,…).  
  –  If you propose **in-turn** the header’s difficulty must be **2**;  
     out-of-turn difficulty is **1**.  (That’s how light clients / other
     validators know whose turn it was.)

• Snapshots  
  –  A snapshot is an in-memory image of the **validator set + proposer history
     + vote state** at some block.  
  –  At every epoch boundary the validator-set can change (system contract
     `ParliaValidatorSet` emits events); we build a new snapshot and persist it
     (we’re now storing it CBOR-compressed in table `ParliaSnapshots`).  
  –  When validating a header we fetch the **parent** snapshot, check
     `miner ∈ snapshot.validators`, decide if the block is in-turn, etc.

• Fast-finality votes  
  –  Validators sign BLS votes (`VoteEnvelope`) over a
     source→target checkpoint pair.  
  –  When ⅔ validators have voted, the aggregated attestation is put in the
     **extra-data** field of the next block; that justifies / finalises the
     target.  (We still have to port the BLS verification part.)

• System / internal transactions  
  –  Two main kinds:  
     1. **Block reward** transfer (to the proposer) – executed inside the EVM
        by the `ParliaReward` precompile.  
     2. **Slashing / validator-set updates** – executed by
        `ParliaValidatorSet` contract; only possible at epoch boundaries and
        requires ≥ ⅔ vote in the previous epoch.

------------------------------------------------------------------
2. What happens in today’s `loocapro_reth_bsc` when you sync with a tip hash
------------------------------------------------------------------
• Header side  
  – Until this week we had only *debug* consensus: headers were trusted as long
    as they chained together.  
  – We just introduced `ParliaHeaderValidator` which _does_ check  
    (a) miner is authorised;  
    (b) difficulty matches in-turn/out-of-turn;  
    (c) basic parent‐number/timestamp monotonicity.  
    That is already enough to reject a block proposed by a non-validator or
    with forged difficulty.

• Execution side (system tx)  
  – The EVM executes _whatever_ is in the block; the generic Reth pipeline
    compares the resulting state-root with the header’s `stateRoot`.  
  – That means **malicious extra reward / arbitrary balance mint** would be
    caught automatically, because the root could not match Binance’s canonical
    chain.  
  – What is **not yet enforced** is the *semantics*: e.g. a validator could
    insert an *empty* system-reward tx and leave the reward unclaimed; the
    root would still match.  To enforce such domain-specific rules we need the
    pre- and post-execution hooks described below.

------------------------------------------------------------------
3. Status of zoro-reth’s `pre_execution.rs` / `post_execution.rs`
------------------------------------------------------------------
File | In our port? | Purpose
---- | ------------- | -------
`zoro_reth/.../pre_execution.rs` | **Not** yet | Before the EVM executes a block it:   • loads the current validator-set from `Snapshot`;   • injects reward & slashing system-transactions;   • verifies their *format*.
`zoro_reth/.../post_execution.rs` | **Not** yet | After execution it:   • checks the `vote_attestation` matches what the EVM wrote;   • updates the `Snapshot` with new proposer/vote/validator-set data;   • decides whether to persist a checkpoint.

`loocapro_reth_bsc` already contains **system-contract bindings** under
`src/system_contracts`, so normal user/SYSTEM calls will run fine, but the
orchestration logic (inject + validate) from the two EVM middleware files has
not been ported.

------------------------------------------------------------------
Next milestone outline
------------------------------------------------------------------
A. `DbSnapshotProvider`  
   – MDBX-backed provider with small LRU cache.  
   – `Snapshot::apply()` returns the **next** snapshot which we will persist
     every `CHECKPOINT_INTERVAL`.

B. Pre- and post-execution middleware  
   – Port the two zoro files, adapting them to Reth’s Stage/Executor
     interfaces.  
   – Hook them via the `BlockExecutorFactory` extension point.

C. Seal & vote verification in `validate_header`  
   – Recover signer from `extraData` – ensures proposer really signed.  
   – Verify aggregated BLS signature when a vote attestation is present.

With those three pieces we’ll have a fully-featured Parlia node that rejects
all invalid blocks (headers, reward logic, fast-finality).

Let me know if you want to start with part A now!