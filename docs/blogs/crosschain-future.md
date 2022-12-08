# Crosschain Future

There is no doubt that Ethereum is the most secure blockchain (as of today) with the largest ecosystem. It is foreseeable that Ethereum could become the "Pareto frontier"<sup>[1](#f1)</sup> for running blockchain applications, but it is highly improbable that there will be "a blockchain for everything". When we zoom into specific use cases, not all properties of a blockchain<sup>[2](#f2)</sup> have equal importance. The choices made on algorithms and submodules<sup>[3](#f3)</sup> lead to vastly different user experiences and security guarantees. The tradeoffs are already manifested in layers 1 blockchains and their distinctive ecosystems today, which capture different types of builders and products.

<a name="f1">[1]</a>: Often used in [reforcement learning](https://en.wikipedia.org/wiki/Pareto_front) - the best solution for the general case (i.e., under one set of weights for each objective) would no longer be the most optimal when the weights change. 

<a name="f2">[2]</a>: For example: security guarantees, resistance to censorship, execution speed, transaction throughput, finality time

<a name="f3">[3]</a>: Consensus mechanisms, signatures schemes, block parameters (size, time, and more), execution models (ordering, parallelization, and more)

If this trend continues, we should see more blockchains dedicated to different purposes. For example, if we were to think of different blockchains as different cities and users of each blockchain as citizens in each city, we would not want to all live in one dystopian mega-city. Just like cities need roads to connect them, blockchains need bridges to talk to each other, so "citizens" of each chain can freely come and go as they please and experience the best each chain can offer. Therefore, assets and information must be able to flow between any two blockchains freely, safely, and securely. 

Thus, we believe the best bridge is the bridge that silently does its job in the background:

- automate the flow of information
- verify everything itself does and everything goes through the bridge 
- without trusting any third party, betting on third parties' economic incentives, or requiring any good-faith behaviors

It also needs not to be said why we don't simply bridge assets or information cross-chain using trusted agents, such as [Chainlink Oracle](https://chain.link/cross-chain) or a self-hosted server. It is akin to placing trust in MtGox or FTX - which recently has proven to be a \$10 billion mistake. 

In other words, the ideal bridge should be "trustless":

- the bridge itself should not trust any intermediary or make any assumption on participants' behaviors, and 
- it should not require the user to trust anything other than what the user can already verify and what they already trust<sup>[4](#f4)</sup>

<a name="f4">[4]</a>: The security of the source and destination chains, such as the integrity of smart contracts on these chains, the consensus of their validators, and that no more than 1/3 is malicious, among other assumptions




## The role of ZKP

In a separate article ["What to build next in ZKP"](https://github.com/isolab-gg/isomorph/blob/main/docs/blog/what-to-build-next.md), we showed a few ways ZKP could fix the "bridge problems"<sup>[5](#f5)</sup> without getting into too many technical details. We will discuss our solutions to specific problems (with a lot of math and empirical data) in separate documents. Without repeating what we said in other articles, here we want to explain on a high level why we believe ZKP would play a critical role going beyond solving immediate problems. 

Regardless of what assets or messages are sent across a blockchain, the destination chain must verify events and messages claimed to have been committed in other chains<sup>[6](#f6)</sup>. However, the verification is typically not feasible to fully compute on-chain because the computation requires using functions too complex by the chains' virtual machines (such as EVM) and processing a large amount of input data. To work around this, we have to partition the computation and move some parts off-chain while proving the off-chain computation is done correctly using some known parameters stored on-chain and some off-chain data the on-chain verifier doesn't care about (except for relevant parts)<sup>[7](#f7)</sup>. ZKP is the ideal tool for this because<sup>[8](#f8)</sup>:

- there is a lot of intermediary data from the source blockchain that needs not be recorded on the destination chain but is required as part of the verification process (hence the off-chain computation). With ZKP, proof can be generated for the off-chain computation. At the same time, the intermediary data does not need to be part of the proof (or provided as input to the on-chain verifier) if the data is used as the witness[8] in the ZKP proof generation process. 
- the off-chain computation needed to verify these events and validate messages are typically cryptographic functions that can be well represented by circuits<sup>[9](#f9)</sup>. The proof generation can be structured recursively to accommodate more extensive computations while incurring only minimal additional work to verify these proofs on-chain. This property, combined with the previous property (removal of intermediary input data in verification), makes the verification scale very well against high volumes of transactions, messages, and complex use cases if needed.

Besides verifying computation efficiently, ZKP also plays interesting roles in use cases requiring privacy. where both anonimity and confidentiality<sup>[10](#f10)</sup> could be applicable. In practice, both have already been achieved in existing protocols based on a single chain<sup>[11](#f11)</sup> using ZKP, which can be used to derive a cross-chain protocol. 

We will discuss the technical details of the above three areas in separate technical deep-dive articles.

<a name="f5">[5]</a>: Referring to problems of the present day, such as hacks with over \$2 billion worth of user funds in 2021 and 2022.

<a name="f6">[6]</a>: Strictly speaking, the events and messages are recursively aggregated into Merkle trees (such as block headers) or other checkpoints to reduce the amount of on-chain data and computation. The exact approaches are discussed in our detailed technical articles.

<a name="f7">[7]</a>: A simple example is the Merkle tree, where the root is often stored on a chain, and proof of membership of the tree is done by submitting a path of neighbors along with the member (i.e., leaf) to be proved. Verifying ZKP is more complex but with the same principle.    

<a name="f8">[8]</a>: To give readers unfamiliar with ZKP some context: On a high level, a ZKP is generated by taking public input x, a witness (private information) w, a (publicly known) program q such that q(w, x) = y and produces a proof p. The proof p has properties such that anyone can easily verify y is the output of running q with input x and some unknown witness w. "Easily" means the amount of computation in verification is insensitive to or much less than (in logarithm order to) the size of x, q, and w, and in practice, can be completed using a reasonable amount of gas on-chain. In practice, the form and complexity of the program q are quite limited. It needs to be represented as circuits and not require an absurd amount of computational resources (and time) to generate a ZKP. 

<a name="f9">[9]</a>: Such as in [Circom](https://docs.circom.io/), and other tools in-development by other teams that directly transform common programming language code, such as [RISC0](https://www.risczero.com)

<a name="f10">[10]</a>: Anonymity refers to the ability to trace a bridge transaction to its source. Confidentiality refers to protecting the content of the messages (or assets). See Wei Dai's [Navigating Privacy on Public Blockchains](https://wdai.us/posts/navigating-privacy/) for more discussions regarding these two concepts and how they are typically handled on-chain today, and how ZKP is used.

<a name="f11">[11]</a>: Such as Zcash and Tornado Cash (banned by US regulators).

## Use cases of the near future

Crypto gaming<sup>[12](#f12)</sup> has been a use case on the rise<sup>[13](#f13)</sup> despite the ban from large gaming platforms (Microsoft, Steam, and others). One tradeoff game developers have to make is where to store the assets, where to store game states and execute game logic, and where to onboard users. Ethereum is great for storing and trading player assets<sup>[14](#f14)</sup>, but it is also the least economically feasible to store any state or execute game logic. Other blockchains, such as Harmony, Avalanche, Polygon, and Solana, offer superior execution and storage properties<sup>[15](#f15)</sup>. Still, the ecosystem would not be as large, the NFT assets become less valuable and tradable, and average users could be more prone to hacks<sup>[16](#f16)</sup>. For more sophisticated games, the centralization risks on these blockchains could also be a concern, along with other issues such as price volatility, network stability, and players' trust.

A trustless bridge<sup>[17](#f17)</sup> could potentially enable the games to take advantage of the best parts of both Ethereum and a fast chain:

- Key events in the game (such as a player's death in a 5v5 shooter game) emitted by contracts in the fast chain could be batched, collected in snapshots, and transmitted to Ethereum for post-game analysis and verification<sup>[18](#f18)</sup>
- Initial state of the game (such as player weapons and skins in a 5v5 shooter game) could be bridged from Ethereum to the fast chain during the initialization of each game. 
- A verifiable snapshot with the ending state of the game is bridged to Ethereum. Afterward, rewards and achievements (NFTs and tokens) could be disbursed to players on Ethereum.
- Some information may need to be kept private. For example, a shooter game should not reveal player position, health, team communication log, and inventory data of each player until the game is completed. Sometimes, the existence of a game and who is playing the game should also be kept private. A trustless bridge supporting privacy options could potentially make both happen.

<a name="f12">[12]</a>: For a discussion of practical ideas, see ["Crypto Gaming: A Most Practical Thesis"](https://medium.com/collab-currency/crypto-gaming-a-most-practical-thesis-ec4f55f53408) by Arad. For crypto-native games, see [The Strongest Crypto Gaming Thesis](https://gubsheep.substack.com/p/the-strongest-crypto-gaming-thesis) by gubsheep who built [Dark Forest](https://zkga.me) with ZKP.

<a name="f13">[13]</a>: See also topics and discussions in [GAM3R](https://gam3r.org/agenda)

<a name="f14">[14]</a>: Because of its highly liquid NFT marketplaces, mature ecosystems, and toolings

<a name="f15">[15]</a>: For example: fast finality, low gas fees, more primitives (VRF, VDF), and larger block sizes

<a name="f16">[16]</a>: For example, thefts on out-of-maintenance wallets, such as this [\$56M incident](https://www.youtube.com/watch?v=uM8ufifIlDY) 

<a name="f17">[17]</a>: A not-so-trustless bridge could also achieve some of these goals but could open up the game to systematic, catastrophic risks when the bridge is hacked

<a name="f18">[18]</a>: This could be important when the stake is high for the game's outcome, such as in eSports tournaments. 

## Beyond bridging assets

Beyond making asset transfer cross-chain trustless, fast, and secure, what else should we do to connect the blockchains and help developers build the cross-chain future? To our knowledge, little has been built for cross-chain communication infrastructure, tooling, and standards. Here are some building blocks which could play significant roles in this.

- **message transmission**: formatting, serialization, verification of cross-chain messages, i.e., in web2 analogy, similar to protocol buffer<sup>[19](#f19)</sup> and ZMQ messages<sup>[20](#f20)</sup>, 
- **data transport and authentication**: authenticating and transporting the messages, managing sessions, channels, and connections. i.e., in web2 analogy, similar to the family of tools and protocols<sup>[21](#f21)</sup> built around protocol buffer
- **relayer/provers**: besides retrieving and processing block headers and signatures of source blockchains, the relayer could act on behalf of clients using the app to generate ZKP proof, encrypt and sign cross-chain messages, 
- **cross-chain accounts**: the ability to control accounts and interact with smart contracts on other chains by sending messages through the bridge
- **payment processing**: SDK and contracts to handle fees payment for gas, bridge usage costs, and on-chain services connected to the bridge
- **domain name services**: routing, resolving, registering, and synchronizing domain names across different chains and potentially mapping to web2 contexts

<a name="f19">[19]</a>: See [Protocol Buffers](https://developers.google.com/protocol-buffers), a language-neutral, platform-neutral extensible mechanism for serializing structured data

<a name="f20">[20]</a>: See [ZMQ Messages](https://zeromq.org/messages/) documentation and [guides](https://zguide.zeromq.org/docs/chapter2/#Messaging-Patterns)

<a name="f21">[21]</a>: Such as how one could use [gRPC](https://grpc.io/docs/what-is-grpc/core-concepts/) to define the services and generate RPC templates for various languages  

### Existing interchain protocols

Note that [IBC protocol](https://ibcprotocol.org/) already defined standards<sup>[22](#f22)</sup> for apps to communicate between blockchains provided that a set of required functions and light clients are implemented (by either the chain, the app, or third parties) as required by the standards. The chains also need to satisfy some essential pre-requisite, such as fast finality. Some proposals in the IBC protocol overlap with some building blocks we enumerated above, and some have well-maintained implementations<sup>[23](#f23)</sup>. The limitation of the IBC protocol is that little implementation for the core protocol exists outside the realm of Cosmos<sup>[24](#f24)</sup>. It can be very challenging for applications to implement the protocol for new blockchains and work with low-level requirements, and there is little incentive to do so. 

Nonetheless, even though ZKP is not used in the protocol, these proposals, the modular design, and some IBC implementations provide excellent starting points for our building blocks. 

<a name="f22">[22]</a>: See also [specs](https://github.com/cosmos/ibc/tree/main/spec), third-party technical overview ["How Cosmos's IBC Works to Achieve Interoperability Between Blockchains"](https://medium.com/@datachain/how-cosmoss-ibc-works-to-achieve-interoperability-between-blockchains-d3ee052fc8c3), and [Cosmos documentations on IBC](https://tutorials.cosmos.network/academy/3-ibc/1-what-is-ibc.html)

<a name="f23">[23]</a>: See [this list](https://ibcprotocol.org/implementations/)

<a name="f24">[24]</a>: i.e., including Tendermint-based blockchains

The framework could also help solve some immediate problems in asset transfer:

- In the Nomad bridge hack<sup>[25](#f25)</sup>, the developer pushed some custom, faulty logic to the smart contract during an upgrade regarding how the proof verifier is initialized and how proofs are processed. The actual bug is simple and could have been prevented with more careful testing. It could also have been prevented if they had used a more structured, principled way to handle data and message framework. 
- In the Binance bridge hack<sup>[26](#f26)</sup>, the community found a similarly faulty custom AVL tree implementation. Since the implementation was fairly complex and tailored to a custom bridging approach, no one caught the errors until it was too late. It could have been prevented if standard data structures had been used and the verification code had been generated and tested in templates.

<a name="f25">[25]</a>: See detailed analysis in [this report](https://drive.google.com/file/d/1GpSEeFe0xmC4WlOA8mm4JSgRnTEiyiTX/view). 

<a name="f26">[26]</a>: See the [Twitter thread by samczsun](https://twitter.com/samczsun/status/1578167198203289600) for a technical explanation

In the long term, cross-chain interoperability projects today<sup>[27](#f27)</sup> could also benefit from these building blocks. For example, as of today, apps based on LayerZero may extend their base layer "LzApp"<sup>[28](#f28)</sup> and pass cross-chain messages using smart contract calls defined in the base class, where the message is wrapped inside a byte array. However, defining, processing, parsing, and verifying the messages are left to the developers with little guidance. It is unclear whether there is any length requirement, when the message will arrive, in what order, whether there will be duplicate messages, and how the receiving app could respond to each message. We need far more than just "passing a byte array" to build an effective cross-chain app in production.

<a name="f27">[27]</a>: Such as LayerZero and Axelar - though neither is trustless. Their approach is discussed in our other article ["What to build next in ZKP"](https://github.com/isolab-gg/isomorph/blob/main/docs/blog/what-to-build-next.md)

<a name="f28">[28]</a>: See [LzApp implementation](https://github.com/LayerZero-Labs/solidity-examples/blob/main/contracts/lzApp/LzApp.sol) 

## Closing Remark 

Many technical problems mentioned here warrant a standalone deep-dive article for further discussion (and their relevane to ZKP). We will also write separate articles to discuss the approaches and algorithms we propose to make the trustless bridge work using ZKP and how we solve immediate problems today. If you are interested in working with us, please let us know at [hello@isolab.gg](mailto: hello@isolab.gg)

