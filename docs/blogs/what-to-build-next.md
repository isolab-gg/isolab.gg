# What to build next in Zero Knowledge Proof

ZKP could potentially “fix the bridge problem” - i.e. the transfer of assets from one chain to another without going through any centralized exchange<sup>[1](#f1)</sup>, the single use case that suffered [$2 billion](https://blog.chainalysis.com/reports/cross-chain-bridge-hacks-2022/) loss in 2021 and 2022 from [hacks](https://messari.io/report/a-year-of-bridge-exploits). Among these incidents, the top root causes are leaked private keys from negligent operators, and bugs in cross-chain event verification code

<a name="f1">[1]</a>:  There are more use cases to bridging than transferring assets. This is discussed later in the article 

Bridges generally function by locking assets on the source chain while simultaneously releasing some equivalent assets on the target chain. At the core of the bridge problem is how a bridge smart contract on the target chain can reliably verify assets are indeed locked in the source chain. When such proofs can be forged (either through operator’s private key or by exploiting bugs), the bridge gets hacked and all assets locked under the bridge may be stolen. Therefore, it makes sense to create mechanisms that minimize the trust given to entities that produce proofs that authorize fund movement, and multiple layers of verifications against those proofs before releasing any fund. A new generation of bridge projects are already doing this. Their approaches fall into the following categories<sup>[2](#f2)</sup>:

- relying trusted watchers to monitor transactions on hold, and report fraudulent submissions before finalizing them and releasing the funds (Nomad),
- processing sensitive operations (verifying transactions, signing off the release of funds) inside trusted SGX enclaves and verify attestations of execution before finalizing the transactions  (Avalanche)
- using an independent network of validators to trace events and synchronize communications across chains, where trust over the validators is established by economic mechanisms, such as deposits and bonds (Axelar, Map Protocol<sup>[3](#f3)</sup>) 
- requiring trusted multi-party consensus (LayerZero, Multichain). 

<a name="f2">[2]</a>: See also [“Security Stack-Up: How Bridges Compare”](https://jumpcrypto.com/security-stack-up-how-bridges-compare/) and [“A classification of various bridging technologies”](https://medium.com/harmony-one/harmonys-cross-chain-future-41d02d53b10) for some different perspectives 

<a name="f3">[3]</a>: Also uses ZKP, but creates its own chain to make proof generation and verification substantially easier

However, these solutions primarily focus on how and where trust is placed and partitioned.  Somewhere in each approach, there are some trusted third-parties which the users may have no idea about, yet their security and good-faith performance of their role are critical for the security of the bridge. If they get hacked, users’ funds locked under the bridge could be completely stolen. For example, watchers can be bribed or made offline temporarily (plus users have to wait for a long time for fraud challenge periods). Tens of different hacks on SGX enclaves are published every year, which can be used to execute malicious transactions. Validators from an independent network usually only post million dollars in bonds, compared to billions of dollars locked under the bridge. The validator network could go down or malfunction, and the validator themselves may be bribed.

Ideally, we should be “trustless” - i.e. in the bridging flow, eliminate the need to trust any third-parties doing their jobs correctly or not getting hacked. Third parties are the entities that are external to the source and target blockchains (and their validators) where the user already trusts implicitly<sup>[4](#f4)</sup>. This is achievable with ZKP, since we can prove an event (such as an asset-locking transaction) has occurred with ZKP and have the proof verified by smart contract. Here, the challenges are how ZKP can be used efficiently, affordable, and made universally accessible to all users on a variety of blockchain networks, for example: 

- different blockchain networks use different consensus mechanisms, signatures, curves, and each combination may require construction of new circuits;
- With existing tools, generating proofs for very complex circuits could consume hundreds to thousands of gigabytes in memory and hours of computations; 
- ZKP verification and recording events on-chain frequently can be very expensive

<a name="f4">[4]</a>: as in, the user is already transacting and placing assets on these two chains

The concept of creating a “trustless” bridge is not new. Multiple projects have proposed customized variations<sup>[5](#f5)</sup>. Some of them have already achieved some success. The NEAR bridge<sup>[6](#f6)</sup> is based on a trustless architecture with the exception that it relies on economic incentives and watchers to secure bridging assets from NEAR to Ethereum. The smart contract light client on Ethereum requires anyone who submits block headers (summarizing events from NEAR) on the light client (on Ethereum) to first post a bond in Ether, and allows watchers (which can be anyone) to challenge the submissions, show proof that some events are fraudulent, and receive rewards taken from submitter’s bonds upon a successful challenge. The drawback created by this exception is similar to the approach by Nomad as discussed previously. The transactions must be held back for 16 hours before they can be finalized, to create enough economic incentives and give watchers enough time to challenge the submitted block headers. If watchers took a long time to identify frauds, all transactions in-between must be rolled back. It also creates a trust assumption that some watchers will always be online and faithfully monitor all submitted events to catch frauds. If all watchers are attacked, bribed, or made offline, the mechanism may collapse.

<a name="f5">[5]</a>: for example, the Harmony Horizon trustless Ethereum bridge (not launched, still in development) 

<a name="f6">[6]</a>: https://near.org/blog/eth-near-rainbow-bridge/

Instead of creating an exception, we could verify the submitted block headers using ZKP. NEAR validators sign the block headers using Ed25519. A zero knowledge proof of a single Ed25519 signature can be generated offline within a few seconds, and verified on-chain using a small amount of gas (close to minting 3-5 NFTs). There are still challenges remaining regarding how all (100+) validators’ signatures can be efficiently verified and aggregated, and how multiple block headers can be submitted, verified, and committed in an expensive way. 

Finally, we should also keep in mind that since only the occurrence of events need to be proved by ZKP, the functionality of “bridging” can be extended to cross-chain communication, such as allowing apps to send messages from one chain to another<sup>[7](#f7)</sup>, thereby support use cases other than mere transferring of assets<sup>[8](#f8)</sup>. In some use cases, the communication needs to be private or semi-private<sup>[9](#f9)</sup>. Nonetheless, to make these happen in reality, we need to build the tools and infrastructure, to name a few: ZKP circuits for signature verification, cross-chain communication data formats, standardized proof formats and verification flow, data serialization packages, efficient cross-chain communication and handshake protocol, cross-chain message encryption packages, and many others. For developers, each of these areas could present interesting challenges and enormous opportunities.

<a name="f7">[7]</a>: Similar to the apps based on LayerZero

<a name="f8">[8]</a>: as also noted by [zkBridge](https://arxiv.org/abs/2210.00264) paper

<a name="f9">[9]</a>: For example, a gaming application could use Ethereum for gaming asset trading and storage, and use another faster chain for game state update and interactions between players, which should not be revealed