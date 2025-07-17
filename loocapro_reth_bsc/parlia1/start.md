# let me introcude my workspace folder structure
- bsc_offiical :  is the bsc network official code  (https://github.com/bnb-chain/bsc).  I clone it here as a reference. 
- revm:  is the a fork of official  https://github.com/bluealloy/revm.  It is also as a reference. 
- loocapro_reth_bsc_offical_no_changes:  is the offiical repo of "https://github.com/loocapro/reth-bsc", which aims to build a BSC node in rust by using reth
- loocapro_reth_bsc:  this is a fork of "https://github.com/loocapro/reth-bsc", and it is the main workspace of my project. I will also work on this loocapro_reth_bsc project to make the BSC full node in rust.
- reth: is the fork of https://github.com/paradigmxyz/reth, and it is also the main workspace of my project. Because sometime it is necessary to make changes in reth, so that loocapro_reth_bsc can depends on it.
- zoro_reth:  this is also a fork of https://github.com/paradigmxyz/reth  .  Last year, bnbchain team extends the official https://github.com/paradigmxyz/reth  and try to make it as a BSC node. ANd it was working  correctly.  Its repo URL is https://github.com/bnb-chain/reth.  However, it is out-of-dated.  The original reth made a lots of updates but the https://github.com/bnb-chain/reth does not  sync it up later.  And bsc network has passed 2 new hardfork (Lorentz hardfork , see https://github.com/bnb-chain/BEPs/blob/master/BEPs/BEP-520.md and Maxwell hardfork, see https://github.com/bnb-chain/BEPs/blob/master/BEPs/BEP-524.md).  So, zoro_reth is a very decent reference for me to implement the BSC node on our loocapro_reth_bsc folder.   

# My currenct urgent mission
 is the implement the parlia consensus in loocapro_reth_bsc folder. This part was fully implemented in zoro_reth repo (except that it misses the 2 latest hardforks related changes).

I have been working on my local loocapro_reth_bsc repo and local reth repo for a while.  And my next outstading issue is to make the "loocapro_reth_bsc/tests/e2e_flow.rs" work. 

# can you 
1. read all code of these repo and reference first
2. help me to check what's missing in my local loocapro_reth_bsc , by comparing with local zoro_reth, from the perspective that I want to implement a full parlia consensu.  
3. continue the debugging for the "loocapro_reth_bsc/tests/e2e_flow.rs"

