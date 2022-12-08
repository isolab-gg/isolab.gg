---
layout: post
author: Aaron Li @ Iso Lab
title: "Trustless Bridging Technical Problems - PART 1: Problem Overview, Consensus Protocols, Signature Schemes"
excerpt: "What problems are we solving in building a trustless bridge, and why? This article is the beginning of a series that deep dives into the technical issues, designs, and open problems of trustless bridging and ZKP"
redirect_from:
  - /blogs/tech-problems-1
--- 

## Technical Problems Overview

In an ideal world, our bridge is a smart contract, and it has access to light-client smart contracts on each chain that could<sup>[1](#f1)</sup>

1. store all valid block headers from another chain;
2. validate all incoming block headers submitted by anyone before accumulating them in storage;
3. use the block headers to verify any event occurred within that block from another chain.

If so, the bridge would be trustless and could finalize any transaction instantaneously<sup>[2](#f2)</sup>. Here, (3) is easy given (1) and (2). For any event *e* which a user would claim to have occurred on the source chain at block height *h*, the user could submit to the bridge smart contract with a transaction receipt *r* of which event *e* was emitted along with a merkle inclusion proof showing that *r* belongs the block<sup>[3](#f3)</sup>. The bridge smart contract could then access the receipt trie root in the block header at height *h* stored (which is guaranteed to be available by (1) and guaranteed to be valid by (2)), verify that the receipt root matches the submitted root in merkle inclusion proof, verify the merkle inclusion proof, and finally verify the receipt indeed contains the claimed event *e*<sup>[4](#f4)</sup>. In a typical lock-mint bridge, the event *e* is transferring (i.e., locking) or burning tokens in typical ERC20 fashion, emitted by the smart contract address on the source blockchain. 

<a name="f1">[1]</a> The [zkBridge](https://arxiv.org/abs/2210.00264) paper made an attempt to do so for Ethereum <-> Cosmos, albeit operating at a very high cost. The paper proposes several gas optimization techniques in Appendix G, but the optimization does not fundamentally change the order of magnitudes in costs.

<a name="f2">[2]</a> Trustless means the bridge can operate without relying on any central operator, and any user who uses the bridge can predict and verify its behavior, and no user needs to trust anything more than what they already implicitly trust, i.e., the continued operation of the source and destination chains and their validators. See our blog [Crosschain Future](...)

<a name="f3">[3]</a> In Ethereum-like blockchains, this means to show the receipt's hash is a member of the Merkle-Patricia Trie for the receipt hashes,

<a name="f4">[4]</a> In Ethereum-like blockchains, this can be done simply by [extracting logs from the receipt](https://github.com/ethereum/go-ethereum/blob/53d1ae096ac0515173e17f0f81a553e5f39027f7/core/types/receipt.go#L85) and search for topic signatures corresponding to the event *e*.


Unfortunately, doing (1) would incur an excessive cost for some combinations of source and destination chains, especially if the destination chain is Ethereum and the source chain produces blocks at a high frequency<sup>[5](#f5)</sup>. But we may find ways to resolve this issue if we can answer these questions:

- Can we store only a tiny part of the block headers instead of the entire header?
- Do we need to store every block header or partial block header? 
	- If not, how do we get the missing headers when they are needed to prove the occurrence of an event?
	- And how can we verify that the missing headers are valid?
- What's the minimal amount of information we have to store on-chain? Without compromising security, what is the trade-off between the amount of information stored on-chain versus the user experience (e.g., delay in finalization of a bridge transaction, amount of client-side computation required)?

To achieve (2), the validation (and rejection) of proposed block headers both have to be completed efficiently enough such that the cost of continuously doing so for all block headers remains economically feasible. For scenarios in which the source chain uses proof-of-work consensus, this is not an issue as long as the block hash can be verified on-chain efficiently<sup>[6](#f6)</sup>. For proof of stake blockchains, the bulk of the validation process is to verify the validity of some signatures in the block header, given some validator keys<sup>[7](#f7)</sup>. However, because different chains use different consensus mechanisms, signature schemes, and curves for the signatures (which are why the chains are created in the first place), the potential computational cost of verifying these signatures can be problematic. The open questions are:

- How much would it cost to validate a block header from blockchain X on blockchain Y?
- Can the cost be reasonably covered by fees paid by users of the bridge from X to Y?
- If not, is it feasible to operate the bridge at a cost such that blockchain X and Y could reasonably cover the excess cost? Given the security benefits, a few million US dollars per year could be considered reasonable initially if hundreds of millions of user funds are locked in the bridge.


<a name="f5">[5]</a> In most modern blockchains, this means 1-2 seconds per block header, compared to Ethereum's 15 seconds per block header. To estimate the cost, let us start with the fact that most Ethereum-like blockchains have a header size of more than 512 bytes, which requires 16 storage slots (at 32-byte per slot) on Ethereum. Each storage slot costs 20000 gas. At 2 second block time and a gas price of 10gwei, storing all block headers in an hour would cost 16 * 20000 * 1800 * 10gwei = 5.76 ethers per hour, or roughly $6000 per hour using the lower bound of 2022 ether price.

<a name="f6">[6]</a> In Ethereum-like blockchains, popular hash algorithms are implemented as a precompile function in smart contracts, for example, SHA256, which is used by Bitcoin, where the gas cost of execution is almost negligible. Before version 2.0, Ethereum used Ethash, which was not implemented as a precompile function but can be validated within reasonable gas cost on-chain. See notes in [SmartPool: Practical Decentralized Pooled Mining, section "Verifying Ethereum PoW](https://www.usenix.org/system/files/conference/usenixsecurity17/sec17-luu.pdf) and notes in [ETH Relay](https://www.dsg.tuwien.ac.at/team/sschulte/paper/bc2020a.pdf), where the cost is reported to be 3 million gas. Nonetheless, in practice, we only need to verify the hash in chains where per unit of gas is extremely low. Thus, verifying block headers from proof-of-work chains (where computation cost is high) only incurs negligible costs. From a design perspective, newer chains rarely use proof-of-work consensus anymore. However, even among legacy chains, a key element of proof-of-work consensus is to make block header validation computationally simple by design.

<a name="f7">[7]</a> In most chains, the keys change after a fixed amount of blocks are produced following an internal selection process. The transition of keys is usually reflected in the block header at the change interval. The change interval is generally defined as the length of an epoch. Within an epoch, the keys remain unchanged.


## Scope 

There are 10-100 blockchains in production today with reasonable usage, and we cannot analyze the problems for each pair of chains. At this time, to reduce the scope, we limit the analysis to a few blockchains with high usage and are the most representative in terms of consensus and signature schemes in use. We consider the blockchains pairs (X, Y) where X is the source chain and Y is the destination chain, and:

1. Either X or Y is Ethereum
2. X or Y is one of the following blockchains with proof-of-stake consensus

### Polygon

Polygon is representative because it uses ECDSA on secp256k1 and a relatively fixed validator set. 

The consensus protocol is based on Peppermint<sup>[8](#f8)</sup>, a modified version of Tendermint. Validators sign produced blocks using the ECDSA  signature scheme on secp256k1 curves<sup>[9](#f9)</sup>. Currently, the validator set size is fixed at 100 and only changes when a current validator resigns. This restriction will change when a new auction mechanism is implemented.<sup>[10](#f10)</sup>


<a name="f8">[8]</a><a name="f9">[9]</a> See notes and links to code in [Peppermint summary](https://wiki.polygon.technology/docs/pos/peppermint/)

<a name="f10">[10]</a> See Polygon validator [documentations](https://wiki.polygon.technology/docs/maintain/validate/validator-responsibilities/)

### Avalanche

Avalanche is a good candidate because it samples from over a thousand validators to produce blocks, uses generic methods for signing blocks (RSA on an X.509 certificate), is moving to transition to BLS signatures for validators, and has numerous subnets.

In Avalanche, there are two types of consensus mechanisms (Avalanche, partially ordered, and Snowman, linearly ordered, similar to other blockchains). Users can create arbitrary subnets in Avalanche, and any validator is free to participate in the consensus for any subnet<sup>[11](#f11)</sup>, besides the mandatory participation of the special subnet - the Primary Network. Each subnet has three types of chains, each with different roles and running different consensus mechanisms and processing different transaction types: (1) P-Chain, which defines validator sets and process validator-related transactions; (2) X-Chain, for exchanging assets, where blocks are partially ordered; (3) C-Chain, which runs an EVM and handles smart contract interactions<sup>[12](#f12)</sup>. 

We limit our scope to only the **Primary Network** since any bridging implementation is likely replicable in subnets, and subnets will likely be interoperable soon. Furthermore, for trustless bridging, only events from **C-Chain** are relevant since the bridge must be a smart contract, and one could conveniently wrap cross-chain operations inside contract interactions. 

The active Avalanche validator set is unrestricted and permissionless and has more than 1000 members at this time<sup>[13](#f13)</sup>. Block proposers are randomly sampled from the active validator set. Therefore any validator could potentially sign a block<sup>[14](#f14)</sup>. The validators use X.509 (TLS)certificate to sign and verify blocks <sup>[15](#f15)</sup>, and the block headers contain both the certificate and the signature <sup>[16](#f16)</sup>. Neither Avalanche documentation nor code specifies the key and signing algorithms for the X.509 certificate, but the certificate auto-generated by the code (invoked via validator command-line tools) creates a 4096-bit RSA key by default <sup>[17](#f17)</sup>. In recent releases<sup>[18](#f18)</sup> of Avalanche, validators may also load or generate an optional BLS key. This change suggests that the protocol may replace its signature scheme from RSA to BLS in the near future.


<a name="f11">[11]</a><a name="f12">[12]</a> See Avalanche [introductory documentation](https://docs.avax.network/overview/getting-started/avalanche-platform) and technical explanations for [Snowman VM](https://github.com/ava-labs/avalanchego/blob/51c5edd85ccc7927443b945b427e64d91ff99f67/vms/README.md)

<a name="f13">[13]</a> See [Avalanche explorer](https://subnets.avax.network/)

<a name="f14">[14]</a> See technical details in [Snowman++](https://github.com/ava-labs/avalanchego/blob/51c5edd85ccc7927443b945b427e64d91ff99f67/vms/proposervm/README.md)

<a name="f15">[15]</a><a name="f16">[16]</a> See ProposerVM source code for [blocks](https://github.com/ava-labs/avalanchego/blob/51c5edd85ccc7927443b945b427e64d91ff99f67/vms/proposervm/block/block.go#L51) and [verification](https://github.com/ava-labs/avalanchego/blob/51c5edd85ccc7927443b945b427e64d91ff99f67/vms/proposervm/block/block.go#L119)

<a name="f17">[17]</a> See staking code, where [new certificates are auto-generated](https://github.com/ava-labs/avalanchego/blob/51c5edd85ccc7927443b945b427e64d91ff99f67/staking/tls.go#L121). Note that RSA signature can be cheaply verified on-chain, per [EIP-198](https://github.com/ethereum/EIPs/blob/f2db669da93ca4ce1605866e147bfa4f56303fc6/EIPS/eip-198.md). Solidity [libraries](https://github.com/adria0/SolRsaVerify) are also available for RSA signature verification. In the worst case, even if any validator chooses to use a non-RSA custom-made certificate, most of the signing algorithms (ECDSA, EDDSA) supported by the Golang crypto library can also be verified on-chain.

<a name="f18">[18]</a> See [release notes on GitHub](https://github.com/ava-labs/avalanchego/releases/tag/v1.9.2) and [code commit search result](https://github.com/ava-labs/avalanchego/search?q=bls&type=commits)

### BSC

BSC has similar signature schemes to Polygon but with a much smaller set of validators and some degree of random (yet predictable and deterministic) perturbation to the active validator set. 

The consensus protocol is based on Parlia<sup>[19](#f19)</sup>, a variation that adds staking, validators, and elections to the proof-of-authority consensus protocol Clique, initially proposed in the Ethereum community. The protocol uses 21 validators for producing and signing blocks, with 19 of them picked from stakers with top voting power and 2 randomly chosen every 200 blocks <sup>[20](#f20)</sup>. Blocks are signed using ECDSA on secp256k1 curves, and block headers can be verified following the standard signature verification process<sup>[21](#f21)</sup>.


<a name="f19">[19]</a> See [BSC Consensus Engine documentations](https://docs.bnbchain.org/docs/learn/consensus/#consensus-protocol)

<a name="f20">[20]</a> Following BEP-131, see a [summary](https://www.bnbchain.org/en/blog/bep131-introducing-candidate-validators-bnb-smart-chain/) and [detailed specifications](https://github.com/bnb-chain/BEPs/pull/131). Note that the proportion of randomly selected validators may increase, as proposed in the BEP.

<a name="f21">[21]</a> See [code](https://github.com/bnb-chain/bsc/blob/cb9e50bdf62c6b46a71724066d39f9851181a5af/consensus/parlia/parlia.go#L546) for full procedure and how ecrecover is used for signature verification.

### Harmony

Despite relatively lower usage than other candidates, Harmony has a mature, battle-tested implementation for fast consensus using BLS-based signature schemes, which has been in production for over 2 years. Additionally, many other chains are also moving towards using BLS for signing blocks in their consensus protocols.

Harmony follows a two-round Fast Byzantine Fault Tolerance consensus derived from PBFT, where BLS signatures (on the BLS12-381 curve) are used to reduce communication costs<sup>[22](#f22)</sup>. Blocks are produced by validator leaders, a minimal subset of validators, then further broadcasted to all validators and confirmed when more than 2/3 of validators sign the block with their own BLS signatures. The leader then aggregates the signatures into a single one and broadcasts again. The validators may verify the aggregated signature and sign the block again before sending the signed block back to the leader. Finally, the leader (after receiving signatures from 2/3 of the validators) may aggregate the signature for one last time and finalize the block. In the block header, the leader records which validators' signatures are received in each round.

The protocol uses a slot-bidding mechanism to elect a variable number of validators to fill 800-slots, where each validator may occupy multiple slots if their total delegated stake per slot is greater than the effective median<sup>[23](#f23)</sup>. 

<a name="f22">[22]</a> See [Harmony consensus documentation](https://docs.harmony.one/home/general/technology/consensus). As an implementation detail, note that custom [generator points are used](https://github.com/herumi/bls/issues/74)

<a name="f23">[23]</a> See [Harmony Slot Bidding and Election](https://docs.harmony.one/home/network/validators/definitions/slots-bidding-and-election)

### Cosmos

Cosmos is the hub to almost 50 blockchains based on the Tendermint consensus engine and Inter-Blockchain Communication (IBC) protocol. It is also one of the earliest proponents for cross-chain communication and defined the first set of communication specificiations<sup>[24](#f24)</sup>. From a purely technical point of view, the signature scheme for signing blocks, Ed25519, is also often used in many other protocols, such as NEAR.

Cosmos Hub itself has 175 validators<sup>[25](#f25)</sup> and is built upon Tendermint, in which validators sign blocks using EdDSA on Curve25519 (i.e., Ed25519)<sup>[26](#f26)</sup>. 


<a name="f24">[24]</a> See [Cosmos IBC documentation](https://tutorials.cosmos.network/academy/3-ibc/1-what-is-ibc.html)

<a name="f25">[25]</a> See [Cosmos Hub overview](https://hub.cosmos.network/main/validators/overview.html)

<a name="f26">[26]</a> See [Tendermint Core documentation](https://docs.tendermint.com/v0.34/tendermint-core/validators.html#validator-keys)



## Coming Next

We will discuss a few potential approaches to minimize the blockheader footprint on-chain, and introduce checkpoint mechanism, inclusion proof, and proof data structures. We will discuss various ways implement them and challenges in practice, from both a blockchain protocol perspective and a ZKP perspective.

## About the series

Later in the series, we will deep dive into signature scheme verifications, tracking validators, the necessity of ZKP, and various options for proving systems and tooling in practice. We will also present some benchmark and would welcome community participation and feedback
