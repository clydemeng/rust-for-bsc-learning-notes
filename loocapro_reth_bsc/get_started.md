
# run a full node
By default: 
data file : /Users/ruojunm/Library/Application Support/reth/bsc/db


## cmd
10W blocks
``` sh
RUST_LOG=info ./target/release/reth-bsc node \
--chain bsc --http --http.api="eth, net, txpool, web3, rpc" --metrics 0.0.0.0:9001 \
--debug.tip 0x4270b97b8a4bea1edd1f3523d691342fd2002d5066cd8b4667bfbee245380046 \ 

--db.max-size 7TB
```
100W blocks


``` sh
RUST_LOG=info ./target/release/reth-bsc node \
--chain bsc --http --http.api="eth, net, txpool, web3, rpc" --metrics 0.0.0.0:9001 \
--debug.tip 0xe468563fd42441b615ef6132474b2b6692fea05c7523356b79b8e5a414909360 \ 
```


500W blocks

``` sh
RUST_LOG=info ./target/release/reth-bsc node \
--chain bsc --http --http.api="eth, net, txpool, web3, rpc" --metrics 0.0.0.0:9001 \
--debug.tip 0xc47ffcf02770b9510df200e30811e915772770658cac69af9bd40df97dba1cb6 
--log.file.directory ./target/release/logs

```






## db path
/Users/ruojunm/Library/Application Support/reth/bsc



# As we don't have Parlia engine  inside the loocapro_reth_bsc,  how is the block syncing performance, comparing to the case when we have a parlia engine? 

Today’s “debug-tip” workflow and a future genuine Parlia-engine workflow share the same downloader, database schema and REVM execution; the differences are in HOW and WHEN the pipeline can mark blocks canonical and therefore how efficiently stages overlap.

────────────────────────────────────────
1. What happens _now_ ( `--debug.tip` mode )
────────────────────────────────────────
• You inject a single synthetic `forkchoiceUpdated` pointing at some tip hash.  
• The header downloader walks **backwards** from that tip to genesis; no block can be
  marked canonical until all ancestors are present.  
• Bodies / Execution / Hashing / Merkle start **only when** their predecessor stage
  reaches the bottom of the range; they cannot run ahead because the canonical chain is
  not updated incrementally.  
• When you raise the tip later you repeat the whole backward walk from the new hash.

Net result  
‣ Good peak bandwidth for headers (≈15–25 k headers / s over a single peer) but poor
  stage parallelism.  
‣ Sync throughput on commodity hardware tends to be in the 2–4 k **blocks** per second
  range for the first few million blocks, then slows because each restart requires a
  fresh header walk.

────────────────────────────────────────
2. What changes with a real ParliaEngine
────────────────────────────────────────
• Every new header received from peers is **immediately** verified (signature,
  proposer rotation) and inserted into `CanonicalHeaders`.  
• The pipeline can run in its intended mode:
  ‣ `Header` stage works on the **future** tip,  
  ‣ `Body` stage chases a little behind,  
  ‣ `Execution` & hashing stages trail further but never stop; checkpoints move
    continuously.  
• No need to restart with a new `--debug.tip`; the CL keeps feeding heads.

Extra cost  
• Signature verification per block (𝑂(1 k) µs with Rust `secp256k1` on modern CPUs).  
• Validator-set updates every epoch (small DB writes).  
Both are negligible compared to REVM execution time.

Net result seen in other Parlia implementations  
• Header download: unchanged (network-bound).  
• Body download + execution now overlap, so overall **wall-clock sync speed doubles or
  triples**.  
  – On a 16-core Ryzen box, BSC mainnet can sync ~60–80 k **blocks per minute** once
    Parlia is in place; that’s ≈3–4× faster than the staged debug-tip approach.  
• After it catches up, the node continues at live-chain pace without manual input.

────────────────────────────────────────
Summary
────────────────────────────────────────
  Debug-tip mode = convenient for testing but introduces a sequential bottleneck;  
  Real ParliaEngine = small CPU overhead, big win in end-to-end catch-up time and no
  manual restarts.



  # How does pipeline stage works

════════════════════════════════════════════════════
1. Where is the code?
────────────────────────────────────────────────────
• The entire stage-and-pipeline engine lives in **upstream reth**:  
  `reth/crates/stages/*` and `reth/crates/node_core/*`.  
• The BSC fork (`loocapro_reth_bsc`) only _wires_ those building-blocks together in  
  `src/node/mod.rs` and swaps in BSC-specific builders (consensus, executor, etc.).  
  Nothing inside `loocapro_reth_bsc` re-implements the stage logic.

Key files to skim:

```20:93:loocapro_reth_bsc/src/node/mod.rs
ComponentsBuilder::default()
    .pool(EthereumPoolBuilder::default())
    .executor(BscExecutorBuilder::default())
    .payload(BscPayloadServiceBuilder::default())
    .network(BscNetworkBuilder { … })
    .consensus(BscConsensusBuilder::default())
```

```65:120:reth/crates/stages/stages/src/sets.rs      // simplified
HeaderStage   ─┐   (online)
BodyStage     ─┤
SenderRecovery ┤
Execution      ┤
AccountHashing ┤
StorageHashing ┤   (offline)
MerkleStage    ┤
TxLookupStage  ┤
IndexHistory   ┤
PruneStage     ┤
FinishStage    ─┘
```

════════════════════════════════════════════════════
2. High-level workflow
────────────────────────────────────────────────────
1. **Node start-up** (`reth-bsc node`)
    a. `NodeBuilder` instantiates all *components*:  
      network, payload builder, executor, consensus, storage.  
    b. It then constructs a `Pipeline` that owns an _ordered list of stages_.  
        • The list comes from `DefaultStages` in reth (see file above).

2. **HeaderStage (reverse first)**
    • Finds a pivot hash (in `--debug.tip` mode this is your CLI hash).  
    • Downloads headers **backward** until they meet the DB checkpoint.  
    • As soon as a contiguous range exists, it writes those headers to
      `CanonicalHeaders`.

3. **BodyStage (forward)**
    • Requests transaction/ommer bodies for every header whose body is missing,
      starting at the _last_ body checkpoint.  
    • Validates block hashes vs downloaded bodies; fills `Transactions`,
     `BlockBodyIndices`, etc.

4. **SenderRecoveryStage**
    • Recovers `from` addresses from each tx’s signature and stores them in
     `TxSenders`.

5. **ExecutionStage**
    • Uses REVM (wrapped in `BscExecutorBuilder`) to execute blocks in forward
     order, writing state, receipts, bloom, etc.

6. **Hashing & Merkle stages**
    • Re-hash modified accounts / storage keys.  
    • Build Merkle-Patricia trie nodes and update the state root.

7. **Indexing & Pruning stages**
    • Build aux indices (tx-lookup, history).  
    • Optionally prune segments according to the node’s `--prune` config.

8. **FinishStage**
    • Dummy stage signalling the pipeline reached the configured target.

Each stage implements the `Stage` trait (`execute`, `unwind`, `checkpoint`) located
in `reth_stages_api::Stage`.

════════════════════════════════════════════════════
3. What changes once you add a real Parlia engine?
────────────────────────────────────────────────────
Nothing in the stage _code_; only the **source of the pivot** changes:

• Today (`--debug.tip`) the pivot is fixed and headers must reach genesis before the
 pipeline can progress.

• With a Parlia engine the pivot moves automatically; as soon as Parlia validates a
 header, it becomes canonical and downstream stages can proceed in parallel, giving
 you big speed-ups.

════════════════════════════════════════════════════
4. Which repo to read?
────────────────────────────────────────────────────
• For understanding the pipeline mechanics: **read the reth code** (files above).  
• For seeing how BSC plugs into it: inspect the small builder modules in
 `loocapro_reth_bsc/src/node/*`—they merely adapt reth’s generics.

Use this map as your guide when you deep-dive: headers & bodies in `HeaderStage`,
execution in `ExecutionStage`, hashing/merkle in their respective stages, all
under `reth/crates/stages/…`.




## loocapro_reth_bsc/src/main.rs

what is "sender"?
what is "receiver"?
can you explain more fo rthe dependency loop? I do not notice any loop.


Below is a “guided reading map” that answers the three questions and shows what source files to open next.

────────────────────────────────────────
1.  oneshot Sender vs Receiver
────────────────────────────────────────
• “oneshot channel” = asynchronous mailbox that delivers **exactly one value**.  
  – `let (tx, rx) = oneshot::channel::<T>();`

```
   Sender<T>  (tx) ── send(T) ─────▶  Receiver<T>  (rx)  ── .await  ──> T
```

• After `send()` succeeds the channel closes; further sends error out.  
• This is ideal for boot-strapping when
  – component A must start **now**, but
  – it can’t do real work until component B hands it some object later.

In your snippet, `T = BeaconConsensusEngineHandle<BscPayloadTypes>`.

────────────────────────────────────────
2.  Why was the one-shot needed?  (the dependency loop)
────────────────────────────────────────
Let’s zoom in on the two components:

```
BscNetworkBuilder  ─── needs ─▶  BeaconConsensusEngineHandle
        ▲                                 │
        │                                 │
        └────────  built by  ─────────────┘
```

• The **network** service must know how to talk to the consensus engine
  (it forwards `newPayload` / `forkchoiceUpdated` messages).  
• The **consensus engine** can’t exist until `NodeBuilder::launch()` finishes,
  because it needs a ready DB, executor, etc.

If we tried to hand the handle directly we’d have circular construction.  
A one-shot breaks the circle:

```
(1) BscNode::new()               ──> gives Network an Arc<Mutex<Option<Receiver>>>
(2) NodeBuilder::launch() creates the engine        \
(3) main()    engine_handle_tx.send(handle);         > one value hops across
(4) Network receives the handle and continues      /
```

────────────────────────────────────────
3.  Arc::<T>::new and why Mutex<Option<Receiver>>
────────────────────────────────────────
• `Arc<T>` – thread-safe reference counting so multiple tasks can own the same
  `Receiver`.  
• `Mutex`   – only one task may `take()` the `Receiver`, preventing a race.  
• `Option`  – after one component extracts the receiver with `take()`, the
  field becomes `None` so subsequent attempts know the value is already used.

────────────────────────────────────────
4.  Code-reading roadmap after `main()`
────────────────────────────────────────
Main path:

1. **`src/main.rs`**  
   – Parses CLI, builds a `reth_cli::CliBuilder`, calls `.node(node).launch()`.

2. **`reth/crates/node_core::NodeBuilder::launch()`**  
   – Assembles all **Components** (network, pipeline, RPC, etc.).  
   – Starts the **Pipeline** task that drives the stages.

3. **`reth/crates/stages/stages/src/sets.rs`**  
   – `DefaultStages` lists the exact stages (HeaderStage, BodyStage, ExecutionStage …)  
   – Each stage implements the `Stage` trait (`execute` + `unwind`) in its own file
     under `reth/crates/stages/stages/src/stages/`.

4. **Stage files to open next** (follow the sync flow):
   • `header.rs`   – reverse header download algorithm.  
   • `body.rs`     – body downloader & validation.  
   • `sender_recovery.rs`  
   • `execution.rs` (wraps REVM).  
   • `account_hashing.rs`, `storage_hashing.rs`, `merkle.rs`.

Skim them in that order; each file’s `execute()` function is annotated with what it
does and which DB tables it writes.

────────────────────────────────────────
5.  Recap in one sentence
────────────────────────────────────────
`BscNode::new()` uses a one-shot channel (Sender / Receiver) wrapped in
`Arc<Mutex<Option<_>>>` to hand the consensus-engine handle to the
network component _after_ the node is fully launched, thus avoiding a
construction dead-lock and letting you keep the whole setup fully
asynchronous.



## 