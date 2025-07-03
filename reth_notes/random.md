
# Alloy, ommers, blooms
• Alloy = zero-cost, `no_std` Ethereum structs reused by reth.  
• `CanonicalHeaders` = MDBX table mapping *height → header hash* of the current canonical chain.  
• Ommers = uncle headers stored alongside transactions in a block body.  
• Blooms = 2048-bit filters per block (and sometimes per topic index) that make log searches fast.


# Primitives

Purpose: give the entire project one, canonical set of basic data-types so every crate speaks the same language.
Key points
Domain objects — Ethereum data that never depends on node internals:
Block, BlockHeader, Transaction, Receipt, Log, Address, Bloom, U256, H256, etc.
Enums such as TxKind, Signature, SpecId (hard-fork IDs).
Traits that let those types flex with forks/features:
Sealable / Sealed → header fields that only exist after sealing a block.
RlpEncodable / Decodable → canonical wire format for P2P & DB.
Into<alloy_primitives::…> conversions so you can swap alloy or ethers types without pain.
Located mainly in crates primitives, primitives-traits, and a tiny shim ethereum.



# lighthouse
Lighthouse – a Rust implementation of Ethereum’s Consensus-Layer (“Beacon-chain”) client, maintained by Sigma Prime. It handles fork-choice, finality, block gossip and validator duties.
Why pair it with Reth? Since “the Merge”, Ethereum is split into two logically separate pieces:
Consensus Layer (CL): decides which blocks are canonical.
Execution Layer (EL): checks transactions inside those blocks and updates state.
Reth is only an EL client, so it needs a CL client (Lighthouse, Prysm, Teku, Nimbus…) to keep up with the real network. They talk over the Engine API (JWT-auth HTTP on port 8551 by default).