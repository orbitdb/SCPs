# Publish & Maintain Centric Threat Model for OrbitDB

This proposal is to draft, approve, publish, and maintain an *invariant-centric threat model* for OrbitDB—that is, a list of security invariants and known weaknesses of OrbitDB that can be easily understood by all users and stakeholders, from teams building products on OrbitDB, to security auditors of those products, to end users of those products.

See [Invariant-Centric Threat Modeling](https://github.com/defuse/ictm#readme) for a more thorough explanation of this methodology and its advantages.

## Steps

These are, roughly, the steps we'd go through to create this threat model and make it useful:

1. **Draft** - Begin by drafting a threat model and seeking feedback  
2. **Approve** - Get review and sign-off from all OrbitDB devevelopers and leading products. 
3. **Audit** - Seek funding for a security audit. 
4. **Maintain** - Consider the impact of any code changes on the threat model, and update the threat model with these code updates in such a way that projects building on OrbitDB are informed of any changes.

## Draft 

### Adversaries:

* PEER - Any peer on the same libp2p network as an OrbitDB, i.e. anyone who can discover at least one peer on that network.
* READER - A peer without write access to an OrbitDB.
* WRITER  - A peer with write access to a OrbitDB (see [access control](https://github.com/orbitdb/orbit-db/blob/main/GUIDE.md#access-control)) 
* FAULTY PEER - A peer that can violate the OrbitDB protocol in arbitrary ways (due to failure or malicious reasons.)
* SYBIL ATTACK- An arbitrarily large number of faulty peers.
* MALWARE - An attacker with the ability to infect a peer's device and obtain all private keys.
* NETWORK PROVIDER - An attacker who can surveil the network activity of one or more peers.
* NETWORK ACTIVE ATTACKER - An attacker who can block or modify data being sent between one or more peers. 

### Invariants

These are the security invariants that OrbitDB attempts to guarantee.

READER

* Cannot write to any OrbitDB

WRITER

* Cannot write to an OrbitDB they are not authorized to write to.
* Cannot write entries to an OrbitDB that violate invariants enforced by an access controller, provided those invariants satisfy Invariant Confluence (see: https://arxiv.org/pdf/2012.00472.pdf)
* Cannot write entries that appear to be written by another peer. 

FAULTY PEER

* Cannot write entries that appear to be written by another peer. 
* Cannot write to an OrbitDB they are not authorized to write to.
* Cannot censor entries to or from peers who are connected to non-faulty peers.

SYBIL ATTACK, NETWORK PROVIDER, NETWORK ACTIVE ATTACKER

* Cannot write entries that appear to be written by another peer. 
* Cannot write to an OrbitDB they are not authorized to write to.

MALWARE

* Cannot write entries that appear to be written by any peer except the compromised peers.
* Cannot write entries to an OrbitDB unless writable by the compromised peers.

NETWORK PROVIDER

* Cannot write entries that appear to be written by another peer. 
* Cannot write to an OrbitDB they are not authorized to write to.

### Known Weaknesses

OrbitDB has known weaknesses against the following adversaries:

PEER

* Can break or degrade performance on the libp2p network by broadcasting very large amounts of data. (?)  
* Can discover all OrbitDBs on the libp2p network (?)
* Can discover the peerIDs and multi-addresses (typically including IP address) of all participants on the network. (?)
* Can learn which PEER is responsible for any given OrbitDB entry.
* Can learn the activity patterns of all peers, e.g. when they are online, when they are creating entries.

WRITER

* Can do anything PEER can do.
* Can publish an unlimited amount of data to an OrbitDB 
* Can modify or delete data in a [keystore](https://github.com/orbitdb/orbit-db-keystore)
* Can act without rate limiting imposed by peers, since rate limiting would break eventual consistency of the OrbitDB.
* Can maliciously write entries that violate any invariants not satisfying Invariant Confluence (https://arxiv.org/pdf/2012.00472.pdf) -- NOTE: this concept will be meaningless to most people and will require explanation. 
* Can unknowingly write entries that violate any invariants not satisfying Invariant Confluence.
* Can write entries that, while valid to OrbitDB, cause the application built on OrbitDB to behave in unexpected ways.

FAULTY PEER

* Can do anything PEER can do.
* Can selectively slow the delivery of entries
* Can undetectably censor entries when there is only one other peer online.
* Can prevent a peer from connecting to non-faulty peers, provided they are the only available peer the peer knows about (by not providing peer discovery information) and censor entries.
* Can censor entries to a given peer or peers by selectively DoS'ing all nodes they are connected to. 
* Can corrupt a keystore, if a WRITER. (can they?)
* Can make it slow or improbable for a peer to connect to non-faulty peers by providing them with a large number of fake peer addresses or known faulty peers. 

SYBIL ATTACK

* Can do anything FAULTY PEER can do.
* Can censor entries to and from an arbitrary number of peers with an eclipse attack: https://bitcoinops.org/en/topics/eclipse-attacks/
* Can do this undetectably to affected peers.

NETWORK PASSIVE ATTACKER

* Can discover all OrbitDBs on the libp2p network (?)
* Can discover the peerIDs and multi-addresses (typically including IP address) of all participants on the network. (?)
* Can learn which PEER is responsible for any given OrbitDB entry.
* Can learn the activity patterns of all peers, e.g. when they are online, when they are creating entries.

NETWORK ACTIVE ATTACKER

* Can do anyting PEER can do.
* Can do anything SYBIL can do.
* Can prevent any or all peers from connecting to each other, even if they know the addresses of other non-faulty peers.
* Can arbitrarily degrade service for any peer. 

MALWARE

* Can create entries that seem to be from the compromised peers.
* Can write to any DB writable by compromised peers.

## Questions

In drafting this document, the following questions came up. We can use this as a space for unresolved questions that need to be resolved before publishing:

1. Can any peer on a libp2p network discover OrbitDBs on that network? Or do they need an identifier?
2. What does a non-faulty peer need to receive out-of-band to minimally follow the OrbitDB protocol with other peers? Do they need an id?
3. Should we assume that a peer has connected to at least one non-faulty peer? That is, should "address of non-faulty peer" be one of the things you need to know to join the network successfully? If you don't know at least one, you're instantly eclipsed, right? Or is it enough to connect to a correct libp2p peer who is not an orbitdb peer?
4. Does orbitdb use the latest hardening features of libp2p/gossipsub? Are there known DoS weaknesses that have been fixed? See [this presentation on GossipSub](https://matrix.org/media/Enter-Gossipsub-Slidedeck.pdf) for more on hardening.
5. Can keystore modifications be easily rolled back? Does keystore provide a history of modifications? Like, can you query the log beneath the keystore?
6. Is there a clearer way of explaining what types of invariants are enforceable by access controllers and what types are not? Should we explain with a set of examples? 
7. Are there any limits on a faulty peer's ability to censor entries?
8. Can a faulty peer corrupt a keystore?
9. What's the best way to explain the idea of invariant confluence, i.e. what kinds of things validation and access controllers can achieve, and what kinds of things they can't achieve? For example, they can't enforce uniqueness, or "balance > 0". See https://arxiv.org/pdf/2012.00472.pdf for more.
10. What are the possible attacks from faulty peers intentionally rebroadcasting messages out of order?


