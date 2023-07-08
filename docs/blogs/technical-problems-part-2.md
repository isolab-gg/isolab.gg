---
layout: post
author: Aaron Li @ Iso Lab
title: "Trustless Bridging Technical Problems - Part 2: Bridge Cost Tradeoff"
excerpt: "How do we reduce on-chain storage and computation cost for bridging, and what's the catch?"
redirect_from:
  - /blogs/problems-2
  - /blogs/bridge-cost-tradeoff
--- 

Let's first look at a few ways to reduce the storage and computation costs in [part 1](/blogs/problems-1) for bridging from a non-Ethereum chain to Ethereum:

## Do we need all block headers stored on-chain? 
  
  Not really. We don't use most fields in the block header[^1]. For verifying events, only the block hash and the receipt hash root are relevant. Given any event (e.g. Alice sent 100 USDC to the bridge) and the receipt of the transction in which the event occurred, we could verify whether the event is valid by recomputing the root of the receipt, and check whether the receipt root is the same as the one on the block header. Assuming the receipt hash is already vetted with a corresponding block header and its block hash ahead of time, the verification of the event is a only a series of operations based on Merkle Patricia Trie (MPT) inclusion proof and keccak256 hashes, which can be done quickly on-chain at a reasonbly low cost. The computation can be done on-demand, and the cost can be shifted to users who wishes to execute the bridge transaction without noticeable increase in transaction costs.

  To make the above trade off, we need to validate whether a block header is valid ahead of time and store relevant data (block hash, receipt hash, block height and time). The cost of block header validation can be significant. The process involves verifying the signatures of tens to hundreds of signers[^2], and in most cases, the algorithms for verifying the signatures have no precompiled or native implementations on-chain. Trade-offs can be potentially made here to move most of the computation off-chain, with monetary costs initially paid by voluntary participating nodes and eventually reimbursed by fees or tax charged on bridge transactions. The details of the trade-off will be discussed in the next section.
  
  Hence, by pruning all irrelevant fields of the block header before storing the headers, we can reduce storage costs down to only 96-128 bytes per block (depending on how we store block height and time) or slightly more if we want to store more metadata. Moreover, not all block headers are necessary for running the bridge. Many blockchains produce blocks far more frequently than Ethereum. As of today, it is common to see a block time of 2 second or less among popular blockchains. In many cases, the blocks are scarcely utilized. In practice, it is possible that many blocks simply do not have any transactions relevant to the bridge, and the bridge system itself may choose to batch and execute transactions every few blocks to save costs and to accommodate natural delays. Because of that, we can omit the irrelevant block headers in the on-chain storage, and reduce storage cost by several times or by an order of magnitude. 
  
  To enable this trade-off, we will need to be able to prove and verify a block's validity independent of whether its preceding block is included in the storage. For proof-of-stake chains where the proof of block validity relies on validator consensus, the independent verification of each block should not be an issue because the block header itself contains sufficient information on which signer performed the signing and the signature itself. The public keys of the validators (or sync committee members, in the case of Ethereum) and the occassional member rotation, addition, and removal can be processed and stored on-chain when those events occur. We just need to make sure the blocks of which these events occurred are not skipped and are handled with special care.

## What computations could be moved off-chain? 

1. Tracking the public keys of validators (or sync committee members in Ethereum) and the current set of validators tasked with signing the block headers 
  
2. Verifying the signature(s) in a block header using validator set and their public keys
  
3. Verifying a block header is part of the canonical chain - this can be optional or partly optional, depending on the specific chain. It is a nice-to-have for better security.
  
The canonical way of moving the computation off-chain is by generating a proof that the computation followed a pre-defined program off-chain, such that the proof can be verified on-chain for the specific program and input. A common technique to do that is through zero-knowledge proof, where some private input data (a.k.a witness) is fed into a specific pre-defined program to be executed off-chain. A proof is then computed given the output and the input (both private and public) of the program, and then submitted to a smart contract on-chain, which was generated (specifically for the pre-defined program) and deployed ahead of time to verify the proof for arbitrary input and output, without requiring the private input (hence, "zero knowledge").
  
In practice, the computation for generating the proof off-chain can be very slow and expensive, the constraints on the structure of the pre-defined program (to make proof-generation feasible) can be very limiting, and the cost of on-chain verification for the proof is non-trivial (around 100k gas per proof). Moreover, it can be very expensive in terms of engineering time and resources to make the pre-defined programs compatible, to implement the proof generation process (e.g. constructing circuits), and to ensure the proof generation process does not have security flaws. Considering these factors, it is hardly worthwhile to move the any computation off-chain, unless:
  
1. There are some input data that simply cannot be revealed on-chain. This is often a must-have requirement for private transfer apps (e.g. Tornado Cash) but rarely the case for bridging use cases, unless data privacy is required - which is a feature we will consider in the future, but not now.
  
2. The input data or the computation memory is too large, such that it is not economical or feasible to carry or store them on-chain, due to EVM's restrictive limits on input data, parameter size, and stack size. This could be relevant to bridging use cases, where the block header (or a batch of them) can be too big to fit into a function call on-chain, or when the intermediary steps of cryptography operations require too much memory.
  
3. The computation would take too much time on-chain (against EVM's 5-second limit), or cost too much gas to execute. For example, verifying a single EdDSA signature would cost around 1 million gas, which would translate to \$60 at 30gwei gas price and \$2000 per ETH price. Verifying a BLS12-381 signature would cost a much higher amount of gas [^2] since a pairing operation (which contains many exponentiation steps on 381 bit field elements) is part of the process. 
  
## Which bridging computations should stay on-chain and which should be moved off-chain?

This should be analyzed for each pair of source and target chains, on a case by case basis. For a given pair, if the validation of block headers can be done efficiently and securely on-chain, it would become wasteful to create elaborate and complex zero-knowledge proof systems to move them off-chain, unless it significantly boosts speed, cost-efficiency, or security. Although it is unlikely that we would achieve better speed or cost using common ZKP systems, some upcoming zkVM solutions which further unlock the potential of massively-parallel processors and specialized hardware could eventually help us get there. Better security can be achieved provided that we can do a lot more computations (and prove those computations) than what we otherwise would be able to do on-chain with our hands tied - for example, to check and prove the signatures of every block header and that they indeed form a linear chain, instead of skipping some blocks and merely checking validator signatures on-demand, as suggested above.

### Bridging To Ethereum

#### Polygon -> Ethereum

Polygon has a set of 100 validators and currently has stopped accepting new validators. The validator keys are unlikely to change except in rare instances which keys are rotated by the validators. Polygon has stated this mechanism is only likely to change in a future, unspecified protocol upgrade which allows validators to freely join or leave the effective set. Given the stable nature of the validator set, and the fact that validators sign block headers using ECDSA signatures which can be easily verified on-chain at 3000 gas for each verification, it would seem more effecieint, at least initially, to perform all bridging operations on-chain. 

#### Avalanche -> Ethereum

Avalanche's consensus mechanism is very different from the rest of the blockchains, and it is unlikely that we can verify an Avalanche block on Ethereum either on-chain or off-chain via zero-knowledge proofs. The main difficulty is Avalanche blocks (in C-chain and P-chain, as far as bridging use cases are concerned) do not contain sufficient information we need for verifying the block. Instead, a node (regardless whether it is a light-client) is expected to perform Avalanche consensus and to accept or reject blocks on its own.

Unlike Polygon, Avalanche has over 1000 validators active on the chain and any validator can join or leave at any time with a reasonable amount of stake. Avalanche achieves consensus[^3] by having individual nodes to sample from 20 known validators (with weights proportional to stakes), retrieve their preferences for next blocks, and if a quorum of [15 validators](https://github.com/ava-labs/avalanchego/blob/eb6e7973a900dc647d29a2da8e7a233ac0a5fe77/snow/consensus/snowball/parameters.go#L38) is reached for any block, repeat the same process until a block is confirmed for at least 20 times consecutively. In an improved version of the consensus protocol ([Snowman++](https://github.com/ava-labs/avalanchego/blob/eb6e7973a900dc647d29a2da8e7a233ac0a5fe77/vms/proposervm/README.md)), block proposal is limited to [pseudo randomly sampled](https://github.com/ava-labs/avalanchego/blob/master/vms/proposervm/README.md#snowman-proposers-selection-mechanism) validators unless sufficient time elapsed without sufficient blocks proposed. The block header is signed by its proposer using a RSA signature.

To verify a block header, we need to ensure the block header is signed by the correct proposer using the pseudo random sample algorithm when the proposers did not fail to propose the blocks in time. We also cannot ignore the rare cases of which the proposers fail to propose the blocks in-time, because another validator could have proposed a block which contains a bridge transaction initiated by the user, and the bridge system would not be able postpone and batch the transaction into another block, because it would not know the block would be proposed by a different validator. It is possible to handle the normal case on-chain or off-chain with zero-knowledge proof, but handling the rare case can be challenging. We also need to use a pseudo-random sampler to pick validators, ask them for next blocks, and be able to receive signed responses, so we can submit the signed responses on-chain for verification, or use ZKP to verify them offchain. We can then repeat the process for each block header for 20 times, just like in normal Snowman consensus. However, it is unclear whether the responses we received from validators will be signed, and what portion of the response is signed. More analysis into Avalanche consensus and gossip code is required before we can propose a potential solution.

#### BSC -> Ethereum

Similar to Polygon, blocks are signed by validators using ECDSA signatures, which can be efficiently verified on-chain. The validator set is elected daily from a small pool, and only 21 validators are elected at a time. Among the 21 validators, only 2 are randomly selected, whereas the rest is picked from the top stakers. This means the validator set is nearly as stable as Polygon's, and it is very likely that the most efficient way to verify block headers is purely on-chain, instead of using any off-chain computation or ZKP.  

#### Harmony -> Ethereum

Verifying block headers off-chain using ZKP might be the only way forward, unless efficient pre-compile functions are added on-chain (either on Ethereum, or on some L2s) for verifying BLS12-381 signatures (and computing pairings)[^4]. But tracking validator changes and public keys might not incur too much costs since 100 validators are elected from a pool of nearly 300 validators only every ~18 hours. Still, validator changes are encoded inside a block every time a new epoch starts (~18 hours), and BLS12-381 verification is still needed to verify the validity of the block (and any validator change)

#### Cosmos -> Ethereum

Since verifying one EdDSA signature costs around ~1M gas and at least 32 signatures must be verified before a block header can be deemed valid, it is not feasible to perform the verification on Ethereum itself. There are two possibel solutions: (1) verifying block headers off-chain using ZKP, similar to the case for verifying BLS12-381 signatures. (2) perform the operations in a L2 (cheaply) and make use of the verification result when the L2 operations are finalized. A potential L2-based solution could be far more interesting than constructing and optimizing ZKP circuits for EdDSA, as generating a proof for EdDSA is known to be [very costly](https://arxiv.org/pdf/2210.00264.pdf) even with the aid of distributed proof generation for data-parallel circuits.
  
### Bridging From Ethereum

Since all target chains have cheap gas costs, it becomes highly preferrable to verify everything on-chain, as long as the gas cost is within block size range and the verification can be done on-chain. Instead of tracking validators, it is sufficient to track sync committee members and their keys in Ethereum, which consists of 512 validators and are randomly selected every ~27 hours. 

However, Ethereum 2.0 consensus blocks are signed by BLS12-381 signatures, including signatures produced by sync committees. At this time, we are not aware of any precompile or primatives on any of the above chains for BLS12-381, nor any working smart contract implementation that makes it feasible to verify the signature on-chain.

This means, until native primitives or precompiles become available, we need to rely on off-chain verification for these signatures. In follow up articles, we will look into the detailed steps of how ZKP systems can solve this problem, and compare the performance among the latest systems such as Plonky2/3, Halo2, and others.


[^1] See [Ethereum block header](https://github.com/ethereum/go-ethereum/blob/27e59827d804b0112c4213465ff3b902e25cb6d9/core/types/block.go#L65) for reference, which many non-Ethereum chains derive their design from.
[^2] The exact amount is unknown at this time. We are not aware of any smart contract implementation for computing the pairing in BLS12-381 signature or for verifying the signature. We have seen some [unfinished implementation](https://github.com/semaraugusto/bls-verification-contract) but have not found any working prototype.
[^3] Linear subnets such as C-chain and P-chain. We are not considering subnets based on the non-linear DAG version (such as X-chain) since time and order are not well-defined in those chains and the fundamental differences in consensus structure would make bridging substantially more difficult. See [official documentations](https://docs.avax.network/learn/avalanche/avalanche-consensus#snowball) for a more comprehensive explaination.
[^4] There might happen, but has been postponed multiple times in previous forks. See [discussions](https://ethereum-magicians.org/t/eip-2537-bls12-precompile-discussion-thread/4187/31). Thus, a wiser choice is to develop workaround via ZKP rather than relying on hope alone.