# rust-for-bsc-learning-notes

## workspace structure
- bsc_official: is the official bsc code. no changes is required, just as code learning reference.
- revm : is an fork from https://github.com/bluealloy/revm, no changes is made so far, just as code learning reference.
- reth: is an fork from https://github.com/paradigmxyz/reth, no changes is made so far, just as code learning reference.
- loocapro_reth_bsc: is an fork from https://github.com/loocapro/reth-bsc, no changes is made so far, just as code learning reference. 
- learning_notes: A repo will record my plan and notes for learning revm ,reth, reth-bsc and etc.


## Goal

### Goal 1
Know every code/tech aspects for reth, reth-bsc, revm, revmJIT

### Goal 2
1. Use rust to implement a fullnode for bsc network. 
2. We can use revm, reth, loocapro_reth_bsc as far as possible.
3. Making the bsc node in rust be compatible all hardforks.
4. Translate these code to rust: BSC consensus, fast finality, staking and governance, system contracts, super instruction
