
Updated details after pulling the latest `bsc_official` (Maxwell hard-fork)

1  Parlia consensus – quick recap  
• Proof-of-Staked-Authority derived from Geth’s Clique: validators take deterministic “turns” to propose blocks, using header.difficulty = 2 (in-turn) or 1 (out-of-turn).  
• Validator set lives on-chain; every “epoch” the current validator array (and the `turnLength`) can change via a contract call that is checkpointed into the first block of the epoch.  
• Recent hard-forks stretch the epoch and adjust the cadence of turns/blocks:

Hard-fork   EpochLength TurnLength Block-interval (ms)  
Bohr / default  200   1 (or 4)  2000  
Lorentz     500   8     1500  
Maxwell (new)  1000   16     750  

`EpochLength` is the number of blocks between successive checkpoint-blocks; `turnLength` is how many consecutive blocks one validator stays “in-turn”.  
Maxwell doubles the epoch again (500 → 1000) and halves the target block time to ~0.75 s, so a full epoch still lasts ~12½ min.

Why BSC needs epochs  
An epoch boundary is the only moment the protocol will accept a header whose `extraData` carries:  
• the fresh validator list,  
• optionally a new `turnLength`.  
Having a predictable boundary lets nodes cache the validator set cheaply (one snapshot per epoch) and lets the staking contract batch expensive operations.

2  Fast Finality (unchanged)  
When ≥⅔ validators attest to an epoch checkpoint, that block becomes *finalised*; re-orgs behind it are rejected. Parlia stores these attestations in a bloom-filter–style bitmask inside each header’s `extraData`.

3  “Geth Clique” vs Parlia (unchanged)  
Clique is a simpler PoA engine with fixed, off-chain validators and 30 000-block checkpoints. Parlia keeps its validator set and turn-parameters on-chain and adds the fast-finality attestation mechanism.

4  Status inside `loocapro_reth_bsc`  
• Historical sync currently **does not** execute system transactions or verify self-mint / slashing rules – it only checks beneficiary + difficulty via the new `ParliaHeaderValidator`.  
• The Maxwell epoch/turn constants aren’t wired in yet; our Rust snapshot logic still assumes `defaultEpochLength = 200` and `turnLength = 1`.  
• `zoro_reth/crates/bsc/evm/src/{pre,post}_execution.rs` (which enforce gas-limit rules, reward caps, validator slashing, etc.) have **not** been ported. Those files are purely in the Go reference stack for now.

Next steps (after incorporating Maxwell)  
1. Extend the Rust `Snapshot` update code to switch:  
   – `EpochLength` to 500 when `IsLorentz` becomes true,  
   – `EpochLength` to 1000 when `IsMaxwell` becomes true,  
   – and update `TurnLength` (8 & 16 respectively).  
2. Import pre-/post-execution hooks from `zoro_reth` so system-tx reward and slashing rules are actually enforced during execution.  
3. Add Maxwell block-interval logic to proposer-backoff / timestamp-validation.

Let me know if you’d like me to start modifying the Rust snapshot logic for Maxwell right away or first port the execution-layer hooks.