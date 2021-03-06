# Polkadot Whitepaper Notes
> [Paper](https://github.com/polkadot-io/polkadotpaper/raw/master/PolkaDotPaper.pdf)

* [History](#history)
* [Properties](#properties)
    * [Roles](#roles)
    * [Interchain Communication](#communication)
* [Protocol](#protocol)
    * [Stake-token liquidity](#liquidity)
    * [Voting Norms and Disputes](#vnorms)
    * [Interchain Routing](#routing)
* [Networking](#network)
* [Typos](#typos)

**Why existing protocols maintain wide timing margins on the expected processing time?**<br>
The state transition mechanism, or the means by which parties collate and execute transactions, has its logic fundamentally tied into the consensus “canonicalisation" mechanism, or the means by which parties agree upon one of a number of possible, valid, histories.

*Polkadot <=> decouple consensus architecture from state transition mechanism*

## History <a name = "history"></a>

A more complex scalable solution known as Chain fibers, dating back to June 2014 and first published later that year, made the case for a single relay-chain and multiple homogeneous chains providing a transparent interchain execution mechanism. Decoherence was paid for through transaction latency-transactions requiring the coordination of disparate portions of the system would take longer to process. [source](https://github.com/ethereum/wiki/wiki/Chain-Fibers-Redux)

> high-frequency chains (with very low block times)

## Properties <a name = "properties"></a>

Polkadot is equivalent to a set of independent chains except for two important points:
1. Pooled security
2. Trust-free transaction transactability

Mark Twain:
> “Governments and diapers must be changed often, and for the same reason”

### Roles <a name = "roles"></a>

**Validator**:<br>
* must run a relay-chain client implementation with high availability and bandwidth
    *  the node must be ready to accept the role of ratifying a new block on a nominated parachain
        * ratifying => receiving, validating and republishing candidate blocks
* Once all new parachain blocks have been properly ratified by their appointed validator subgroups (**collators**), validators must then ratify the relay-chain block itself.
    * To ratify the relay-chain block: updating the state of the transaction queues (essentially moving data from a parachain’s output queue to another parachain’s input queue), processing the transactions of the ratified relay-chain transaction set and ratifying the final block, including the final parachain changes.
    
**Nominator**: <br>
* A nominator is a stake-holding party who contributes to the security bond of a validator.

**Collators**:<br>
* maintain a “full-node” for a particular parachain
* collate and execute transactions to create an unsealed block, and provide it, together with a zero-knowledge proof, to one or more validators presently responsible for proposing a parachain block

> collator pools who vie to collect the most transaction fees

> decentralised nominator pools would allow multiple bonded participants to coordinate and share the duty of a validator

**Fishermen**:<br>
* get their reward through a timely proof that at least one bonded party acted illegally
    * Illegal actions include signing two blocks each with the same ratified parent or, in the case of parachains, helping ratify an invalid block
* fishermen must post a small bond (to prevent sybil attacks from wasting validator compute resource)
    * they can achieve a hefty profit from identifying bad behavior (but this will occur relatively infrequently)

**???**<br>
Polkadot, however, also provides strong guarantees that the parachains’ state transitions are valid. This happens through the set of validators being cryptographically randomly segmented into subsets; one subset per parachain, the subsets potentially differing per block. This setup generally implies that parachains’ block times will be at least as long as that of the relay-chain. 
> WHY?

> a Bitcoin-like chain which has a much simpler fee model or some other, yet-to-be-proposed spam-prevention model

### Interchain Communication <a name="communication"></a>

* transactions executing in a parachain are (according to the logic of that chain) able to effect the dispatch of a transaction into a second parachain or, potentially, the relay-chain. 
    * Like external transactions on production blockchains, they are fully asynchronous and there is no intrinsic ability for them to return any kind of information back to its origin.

* Interchain transactions are resolved using a simple queuing mechanism based around a Merkle tree to ensure fidelity. It is the task of the relay-chain maintainers to move transactions on the output queue of one parachain into the input queue of the destination parachain.

* These queues are administered on the relay-chain allowing parachains to determine each other’s saturation status; this way a failed attempt to post a transaction to a stalled destination may be reported synchronously. 
**(Though since no return path exists, if a secondary transaction failed for that reason, it could not be reported back to the original caller and some other means of recovery would have to take place.)**

**Interoperability with Ethereum**<br>
We can imagine our Polkadot-side Ethereum-interface will have a few simple functions:
1. able to accept a new header from the Ethereum network and validate the PoW
2. able to accept some proof that a particular log was emitted by the Ethereum-side break-out contract for a header of sufficient depth (and forward the corresponding message within Polkadot)
3. able to accept proofs that a previously accepts but not-yet-enacted header contains an invalid receipt root

**What does the breakout contract look like?**

**How is this incentivization managed?**<br>
> To actually get the Ethereum header data itself (and any SPV proofs or validity/canonicality refutations), an incentivization for forwarding data is necessary.

## Protocol <a name="protocol"></a>
The relay-chain is state-based with the state mapping address to account information (mainly balances and (to prevent replays) a transaction counter).
> placing accounts here provides accounting for which identity possesses what amount of stake in the system

Notable differences from Ethereum:
* contracts cannot be deployed through transactions (to avoid application functionality on the relay-chain)
* compute resource usage ("gas") is not accounted; flat fee applies, thereby allowing for more performance from any dynamic code execution that may need to be done as well as a simpler transaction format
* special functionality supported for listed contracts that allows for auto-execution and network message outputs

### Stake-token liquidity <a name="liquidity"></a>

To tie the network security to the *market capitalization* of the staking token, as many tokens as possible should be staked within the network maintenance operations.
> we could incentivize this with a high interest rate paid out to validators

**Key problem**: if the token is locked in the Staking Contract under punishment of reduction, how can a substantial portion remain sufficiently liquid in order to allow price discovery?

Potential solution? straight-forward derivative contract, securing fungible tokens on an underlying staked token...but this is difficult to create in a trust-free manner and there is an inherent disparity between the value of the derivative and the underlying.

Parity's solution: forcibly make 20% of token supply liquid

**targeting via reverse auction:** token holders interested in becoming a validator post an offer to the staking contract stating their minimum payout rate for participation. At the beginning of each session (maybe once an hour), validator slots are filled according to each would-be validator's stake and payout rate

> **who is working on this? have they considered a COST for validator slots...this would make more sense for what I expect to be relatively volatile returns (in fiat terms)**

> question: if we use inflation to incentivize staking, wouldn't the liquid token supply continuously decline in value? How is such a scheme sustainable?

**bond confiscation/burning -- will parameters be static or dynamic?**

### Voting Norms and Disputes <a name = "vnorms" ></a>

**Sealing relay blocks** entails the collection of signed statements from validators over the validity, availability and canonicality of a particular relay-chain block and the parachain blocks that it represents.

The BFT consensus algorithm is motivated by Tangaora (a BFT variant of Raft), Tendermint, and HoneyBadgerBFT. The algorithm must reach an agreement on multiple parachains in parallel.

The basic rules for validity of the individual blocks (that allow the total set of validators as a whole to come to consensus on it becoming the unique parachain candidate to be referenced from the canonical rely):
* must have at least two thirds of its validators voting positively and none voting negatively
* must have over one third validators voting positively to the availability of egress queue information

> **unanimous voting is impractical** => allowing negative votes to deadlock the process seems unnecessary; but adaptive quorum biasing was introduced later

After the votes are counted from the full validator set, if the losing opinion has a substantial proportion, it is assumed to be an accidental parachain fork and the parachain is suspended from the consensus process. Otherwise, we assume it is a malicious act and punish the minority who were voting for the dissenting opinion.

> **do we punish dissenters?** and **how do forks work for parachains???**

The ongoing network resource consumption (in terms of bandwidth) grows with the square of the chains, an untenable property in the long-term...Ultimately, we are likely to keep baashing our heads against the fundamental limitations which states that for a consensus network to be considered available safe, the ongoing bandwidth requirements of the order of total validators times total input information. This is due to the inability of an untrusted network to properly distribute the task of data storage across many nodes, which sits apart from the eminently distributable task of processing.

**Introducing Latency**<br>
By requiring 33%+1 validators voting for availability only *eventually*, and not *immediately*, we can better utilize exponential data propagation and help even out peaks in data-interchange. A reasonable equality (though unproven) may be **latency = participants X chains**. 

Under the current model, the size of the system scales with the number of chains to ensure processing is distributed; since each chain will require at least one validator and we fix the availability attestation to a constant proportion of validators, then participants similarly grows with the number of chains. We end up with: **latency = size^{2}**.

**Mitigating the Data Availability Problem**<br>
1. *Public Participants*, similar to fishermen, police the validators who claim availability; specifically, their task is to find the validators that appear unable to demonstrate validity and lodging corresponding micro-complaints to other validators. **Speaker/Listener Fault Equivalence still exists here; we just increase complexity!**
2. *Availability Guarantors*: nominate a second set of bonded validators that attest to the availability of all important interchain data. 
> This has the advantage of relaxing the equivalence between *participants* and *chains*. Essentially chains can grow (along with the original chain validator set), whereas the participants, and specifically those taking part in data availability testament, can remain at the least sub-linear and quite possibly constant.

3. Collators have an intrinsic incentive to ensure that all data is available for their chosen parachain since without it they are unable to author further blocks from which they can collect transaction fees. Recent collators are given the ability to issue challenges to the availability of external data for a particular parachain block to validators for a small bond...Validators must contact those from the apparently offending validator sub-groups who testified and either acquire and return the data to the collator or escalate the matter by testifying the lack of availability (direct refusal to provide the data counts as a bond-confiscating offense, therefore the misbehaving validator will likely just drop the connection) and contacting additional validator to return the same test. In the latter case, the collator's bond is returned...once a quorum of validators who can make such non-availability testimonials is reached, they are released, the misbehaving sub-group is punished, and the block is reverted.

**In-Protocol Randomness**<br>
Taking the XOR distance measure between the collator's address and some cryptographically secure pseudorandom number determined close to the point of the block being created. This effectively gives each collator a random chance of their candidate block *winning* over all others...To mitigate the sybil attack of a single collator "mining" an address close to the winning ticket and thus being a favorite each block, we would add some inertia to the collator's address. 

*Collator Insurance*<br>
To disincentivize collators from sending invalid or overweight block candidates to validators, any validator may place in the next block a transaction including the offending block alleging misbehavior with the effect of transferring some proportion of the misbehaving collator's stake to the aggrieved validator. This transfer front-runs any others to ensure the collator cannot remove the funds prior to the punishment...To prevent malicious validators confiscating collators' funds, the collator may appeal the validator's decisions with a jury of randomly chosen validators in return for placing a small deposit. If they find in the validator's favor, the deposit is consumed by them; if not, the deposit is returned and the validator is fined.

> **Attacks/Critiques**: (1) I grief another validator who confiscates my funds before I can be punished from the validator I actually hurt (I know the validator who front-runs the valid accusation) (2) What if I appeal an invalid accusation and the validator bribes the random council of validators (by posting some bounty greater than the deposit necessary for my appeal) (3) What is the initial fine is illegitimate? The appeal process would need to be non-binary in this case, which would add a lot of complexity...if the punishment is non-binary then this would need to be figured out.

#### Validator Shuffling
Fixed total of c^{2} - c validators with c-1 validators in each sub-group. 

### Interchain Routing <a name = "routing"></a>
*Interchain Transaction Routing*: how a posted transaction gets from being a desired output from one *source* parachain to being a non-negotiable input of another *destination* parachain without any trust requirements.

Posts are structured as several FIFO queues; the number of lists is referred to as the *routing base* and may be around 16. In a way, this number represents the quantity of parachains that can be supported without having to resort to *multi-phase* routing (**hyper-routing** to scale past the initial set of parachains).

> page 14, routing algorithm outlined

**Non-Manipulateable Validator Selection**<br>
Assume strong, CSPR sub-block ordering to achieve a deterministic operation that offers no favoritism between any parachain block pairing. (check citation [9])

* data paths per node grow linearly with the overall complexity of the system
* multi-phase routing may be used to reduce the number of instantaneous pathways at a cost of introducing storage buffers and latency

#### Hyper-cube Routing
Rather than growing the node connectivity with the number of parachains and sub-group nodes, we grow only with the logarithm of parachains. Posts may transit between several parachains' queues on their way to final delivery.

> section 6.6.4

## Networking <a name = "network"></a>
For a properly scaled-out multi-chain, collators will either need to be continously reconnecting to the accordingly elected validators, or will need on-going agreements with a subset of the validators to ensure they are not disconnected during the vast majority of the time that they are useless for that validator. Collators will also naturally attempt to maintain one or more stable connections into their **availability guarantor** set to ensure swift propagation of their consensus-sensitive data.

*Availability guarantors* will mostly aim to maintain a stable connection to each other and to validators (for consensus and consensus-critical parachain data to which they attest), as well as to some collators (for the parachain data) and some fishermen and full clients (for dispersing information). 

Validators will tend to look for other validators, especially those in the same sub-group and any collators that can supply them with parachain block candidates.

Fishermen, as well as general relay-chain and parachain clients will generally claim to keep a connection open to a validator or gaurantor, but plenty of other nodes similar to themselves otherwise. Parachain light clients will similarly aim to be connected to a full client of the parachain if not just other parachain light-clients.

> look into Kademlia

# Typos <a name = "typos"></a>
1. page 15; 6.6.4 "*Hyper-cube routing is a mechanism which can mostly be build*"...should be "built"

