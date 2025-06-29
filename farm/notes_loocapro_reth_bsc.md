Read file: src/node/storage.rs

# Notes 

────────────────────────────────────────
1. Big picture
────────────────────────────────────────
• The project piggy-backs on the Reth code-base (reth_* crates) and only adds the pieces that Binance Smart Chain needs to diverge from vanilla Ethereum.  
• At build time you assemble a `BscNode` through Reth’s composable “builder” pipeline.  
  ── pool  (tx mem-pool)  
  ── executor  (EVM)  
  ── payload   (block building)  
  ── network   (P2P, block import)  
  ── consensus (header/body checks)  
  ── storage   (on-disk DB)  
  All of those are thin BSC-specific wrappers over the equivalent Reth components.

Directory cheat-sheet
src/
  chainspec/        – chain ID, hard-fork activation, genesis loader  
  hardforks/        – helpers telling “is X fork active at timestamp t?”  
  evm/              – BSC-specific EVM config + extra pre-compiles  
  system_contracts/ – Solidity ABIs & byte-code snapshots (for reference)  
  node/             – the actual client: CLI, RPC, P2P, consensus, storage …  
      ├─ cli.rs     – defines the `reth node` sub-command for BSC  
      ├─ engine.rs  – payload builder service ⇢ future block production  
      ├─ consensus.rs – header/body validation rules  
      ├─ network/   – block import pipeline + fork choice  
      ├─ evm/       – wraps Reth’s executor with BSC params  
      ├─ rpc/       – JSON-RPC surface (eth_* and engine_* APIs)  
      └─ storage.rs – BSC view over Reth’s `EthStorage`  
  lib.rs, main.rs   – simple re-exports + binary entrypoint

────────────────────────────────────────
2. Answers to your concrete questions
────────────────────────────────────────
• “How does it communicate with the BSC low-layer storage?”
  Look at `src/node/storage.rs`.  
  `BscStorage` is just a new-typed `EthStorage` from `reth_provider`.  
  Under the hood it leverages `reth_db` (MDBX / RocksDB) via the `DBProvider` trait, so all reads & writes go through the same KV layout Ethereum Geth uses (headers, bodies, state, …).  The TODOs mark where BSC side-cars will be written once implemented.

• “Can it produce blocks like a validator?”
  Scaffolding is present but it is still a NO-OP:  
  `src/node/engine.rs` registers a `PayloadServiceBuilder`, yet the async loop only handles subscriptions and never constructs a payload.  The consensus layer (`node/consensus.rs`) validates incoming blocks but does not seal new ones.  So at the moment the node can *follow* BSC but not *author* blocks until `BscPayloadBuilder` is finished.

• “Does it have an RPC server?”
  Yes.  `src/node/rpc/` builds a `BscEthApi` (extending Reth’s `EthApi`).  
  It exposes standard `eth_*` endpoints plus Engine-API (`engine_` namespace) required for Merge-compatible consensus clients.

• “What components does it have / use?”
  – ChainSpec & hardfork helpers  
  – BSC-tuned EVM (extra pre-compiles: BLS, Tendermint, etc.)  
  – Transaction pool (from Reth)  
  – Storage wrapper  
  – Consensus checks (EthBeaconConsensus thin-wrapper)  
  – P2P network service + block import pipeline  
  – JSON-RPC server  
  – Future payload/builder service  
  – Collection of reference system-contract binaries & ABIs

────────────────────────────────────────
3. 10-day guided learning plan
────────────────────────────────────────
Day 0 – Prerequisites  
  • Install Rust nightly + stable, `cargo run --release`.  
  • Skim the Reth book (reth-docs.github.io) to grasp its abstractions.

Day 1 – Boot & play  
  • Read `src/main.rs` and run `cargo run --release -- --help`.  
  • Launch a dev-net node with a local MDBX db; inspect logs.

Day 2 – Chain spec & genesis  
  • Open `chainspec/bsc.rs` and `genesis.json`.  
  • Trace how `BscChainSpecParser` loads them.  
  • Confirm hard-fork activation table in `hardforks/bsc.rs`.

Day 3 – Storage path  
  • Follow a `BlockBodyReader` call:  
    client RPC (→ `BscEthApi`) → provider (`reth_provider`) → `BscStorage` → `EthStorage` → `reth_db`.  
  • Look at `reth_db::init_db` to see the folder layout.

Day 4 – EVM execution  
  • Open `evm/api/` builders & ctx; then `evm/handler.rs`.  
  • Review extra pre-compile contracts under `evm/precompiles/`.  
  • Compare with upstream Reth executor to spot what was overridden.

Day 5 – Transaction pool  
  • Walk through `reth_node_ethereum::node::EthereumPoolBuilder` (external crate) used in `node/mod.rs`.  
  • Trigger `eth_sendRawTransaction` via curl and watch pool logs.

Day 6 – Consensus & block verification  
  • Step through `node/consensus.rs`; set breakpoints (rust-gdb or VSCode LLDB) on `validate_header_against_parent`.  
  • Replay a historical main-net block file and inspect error cases.

Day 7 – Networking layer  
  • Explore `node/network/*`.  
  • Read `block_import/service.rs` to see how it pipes validated blocks into the DB.  
  • Start two local nodes and confirm peer discovery.

Day 8 – RPC surface deep dive  
  • Open `node/rpc/mod.rs`; list all active traits (`EthApiSpec`, `Trace`, …).  
  • Run `cargo run … -- --http` and experiment with curl / Postman.  
  • Compare answers with a geth/bsc client for parity.

Day 9 – Block production TODOs  
  • Study `node/engine.rs` and the commented `BscPayloadBuilder`.  
  • Draft a design doc on what extra fields (side-cars, staking data) BSC blocks need, and how to feed them into the builder.

Day 10 – System contracts reference  
  • Browse `system_contracts/*` folders to understand how BSC keeps canonical copies of validator set / cross-chain contracts.  
  • Sketch how future state overrides might inject them at genesis.

────────────────────────────────────────
4. Suggested reading order of code files
────────────────────────────────────────
1. `src/lib.rs` – one-screen index  
2. `src/main.rs` – entrypoint  
3. `src/node/mod.rs` – how all builders fit  
4. `src/chainspec/*`  
5. `src/hardforks/bsc.rs`  
6. `src/node/storage.rs`  
7. `src/evm/handler.rs` and `evm/precompiles/*`  
8. `src/node/consensus.rs`  
9. `src/node/network/block_import/*`  
10. `src/node/rpc/**`

Work through the plan; after each day try to re-explain the component in your own words or write a short test that exercises it.  By the end you’ll not only “read the repo” but own a mental model of how a BSC execution client is put together.

Happy hacking!