# BMW #0: Introduction and Index

Welcome, friend! These BOLT MimbleWimble (BMW) documents describe
a layer-2 protocol for off-chain MimbleWimble asset transfer by mutual
cooperation, relying on on-chain transactions for enforcement if
necessary.

Some requirements are subtle; we have tried to highlight motivations
and reasoning behind the results you see here. I'm sure we've fallen
short; if you find any part confusing or wrong, please contact us and
help us improve.

As these documents are designed to be implementation agnostic so for a
better reading experience we describe everything using the fictional
mw-coin (MWC) instead of any implementation specific currency.

This is version 0.

1. [BMW #1](01-messaging.md): Base Protocol
2. [BMW #2](02-peer-protocol.md): Peer Protocol for Channel Management
3. [BMW #3](03-transactions.md): Mw-coin Transaction and Script Formats
4. [BMW #4](04-onion-routing.md): Onion Routing Protocol
5. [BMW #5](05-onchain.md): Recommendations for On-chain Transaction Handling
7. [BMW #7](07-routing-gossip.md): P2P Node and Channel Discovery
8. [BMW #8](08-transport.md): Encrypted and Authenticated Transport
9. [BMW #9](09-features.md): Assigned Feature Flags
10. [BMW #10](10-dns-bootstrap.md): DNS Bootstrap and Assisted Node Location
11. [BMW #11](11-payment-encoding.md): Invoice Protocol for Lightning Payments

## The Spark: A Short Introduction to Lightning

Lightning is a protocol for making fast payments with any compatible
cryptocurrency using a network of channels.

### Channels

Lightning works by establishing *channels*: two participants create a
Lightning payment channel that contains some amount of mw-coin (e.g.,
0.1 MWC) that they've locked up on the network. It is spendable only with
both their signatures.

Initially they each hold a mw-coin transaction that sends all the
mw-coin (e.g. 0.1 mw-coin) back to one party.  They can later sign a new mw-coin
transaction that splits these funds differently, e.g. 0.09 mw-coin to one
party, 0.01 mw-coin to the other, and invalidate the previous mw-coin
transaction so it won't be spent.

See [BMW #2: Channel Establishment](02-peer-protocol.md#channel-establishment) for more on
channel establishment and [BMW #3: Funding Transaction Output](03-transactions.md#funding-transaction-output)
for the format of the mw-coin transaction that creates the channel.
See [BMW #5: Recommendations for On-chain Transaction Handling](05-onchain.md)
for the requirements when participants disagree or fail, and the cross-signed
mw-coin transaction must be spent.

### Conditional Payments

A Lightning channel only allows payment between two participants, but
channels can be connected together to form a network that allows payments
between all members of the network. This requires the technology of a
conditional payment, which can be added to a channel,
e.g. "you get 0.01 mw-coin if you reveal the secret within 6 hours".
Once the recipient presents the secret, that mw-coin transaction is
replaced with one lacking the conditional payment and adding the funds
to that recipient's output.

See [BMW #2: Adding an HTLC](02-peer-protocol.md#adding-an-htlc-update_add_htlc)
for the commands a participant uses to add a conditional payment, and
[BMW #3: Commitment Transaction](03-transactions.md#commitment-transaction) for the
complete format of the mw-coin transaction.

### Forwarding

Such a conditional payment can be safely forwarded to another
participant with a lower time limit, e.g. "you get 0.01 mw-coin if you reveal the secret
within 5 hours".  This allows channels to be chained into a network
without trusting the intermediaries.

See [BMW #2: Forwarding HTLCs](02-peer-protocol.md#forwarding-htlcs) for details on
forwarding payments, [BMW #4: Packet Structure](04-onion-routing.md#packet-structure)
for how payment instructions are transported.

### Network Topology

To make a payment, a participant needs to know what channels it can
send through.  Participants tell each other about channel and node
creation, and updates.

See [BMW #7: P2P Node and Channel Discovery](07-routing-gossip.md)
for details on the communication protocol, and [BMW #10: DNS
Bootstrap and Assisted Node Location](10-dns-bootstrap.md) for initial
network bootstrap.

### Payment Invoicing

A participant receives invoices that tell her what payments to make.

See [BMW #11: Invoice Protocol for Lightning Payments](11-payment-encoding.md)
for the protocol describing the destination and purpose of a payment such that
the payer can later prove successful payment.

## Glossary and Terminology Guide
FIXME: Update these definitions according to changes in other BMWs.

* *Announcement*:
   * A gossip message sent between *peers* intended to aid the discovery of a *channel* or a *node*.

* `chain_hash`:
   * The uniquely identifying hash of the target blockchain (usually the genesis hash).
     This allows *nodes* to create and reference *channels* on
     several blockchains. Nodes are to ignore any messages that reference a
     `chain_hash` that are unknown to them. Unlike `bitcoin-cli`, the hash is
     not reversed but is used directly.

* *Channel*:
   * A fast, off-chain method of mutual exchange between two *peers*.
   To transact funds, peers exchange signatures to create an updated *commitment transaction*.
   * _See closure methods: mutual close, revoked transaction close, unilateral close_
   * _See related: route_

* *Closing transaction*:
   * A transaction generated as part of a _mutual close_. A closing transaction is similar to a _commitment transaction_, but with no pending payments.
   * _See related: commitment transaction, funding transaction, penalty transaction_

* *Commitment number*:
   * A 48-bit incrementing counter for each *commitment transaction*; counters
    are independent for each *peer* in the *channel* and start at 0.
   * _See container: commitment transaction_
   * _See related: closing transaction, funding transaction, penalty transaction_

* *Commitment revocation private key*:
   * Every *commitment transaction* has a unique commitment revocation private-key
    value that allows the other *peer* to spend all outputs
    immediately: revealing this key is how old commitment
    transactions are revoked. To support revocation, each output of the
    commitment transaction refers to the commitment revocation public key.
   * _See container: commitment transaction_
   * _See originator: per-commitment secret_

* *Commitment transaction*:
   * A transaction that spends the *funding transaction*.
   Each *peer* holds the other peer's signature for this transaction, so that each
   always has a commitment transaction that it can spend. After a new
   commitment transaction is negotiated, the old one is *revoked*.
   * _See parts: commitment number, commitment revocation private key, HTLC, per-commitment secret, outpoint_
   * _See related: closing transaction, funding transaction, penalty transaction_
   * _See types: revoked commitment transaction_

* *Final node*:
   * The final recipient of a packet that is routing a payment from an _origin node_ through some number of _hops_. It is also the final *receiving peer* in a chain.
   * _See category: node_
   * _See related: origin node, processing node_

* *Funding transaction*:
   * An irreversible on-chain transaction that pays to both *peers* on a *channel*.
   It can only be spent by mutual consent.
   * _See related: closing transaction, commitment transaction, penalty transaction_

* *Hop*:
   * A *node*. Generally, an intermediate node lying between an *origin node* and a *final node*.
   * _See category: node_

* *HTLC*: Hashed Time Locked Contract.
   * A conditional payment between two *peers*: the recipient can spend
    the payment by presenting its signature and a *payment preimage*,
    otherwise the payer can cancel the contract by spending it after
    a given time. These are implemented as outputs from the
    *commitment transaction*.
   * _See container: commitment transaction_
   * _See parts: Payment hash, Payment preimage_

* *Invoice*: A request for funds on the Lightning Network, possibly
    including payment type, payment amount, expiry, and other
    information. This is how payments are made on the Lightning
    Network, rather than using Bitcoin-style addresses.

* *It's ok to be odd*:
   * A rule applied to some numeric fields that indicates either optional or
     compulsory support for features. Even numbers indicate that both endpoints
     MUST support the feature in question, while odd numbers indicate
     that the feature MAY be disregarded by the other endpoint.

* *MSAT*:
   * A millisatoshi, often used as a field name.

* *Mutual close*:
   * A cooperative close of a *channel*, accomplished by broadcasting an unconditional
    spend of the *funding transaction* with an output to each *peer*
    (unless one output is too small, and thus is not included).
   * _See related: revoked transaction close, unilateral close_

* *Node*:
   * A computer or other device that is part of the Lightning network.
   * _See related: peers_
   * _See types: final node, hop, origin node, processing node, receiving node, sending node_

* *Origin node*:
   * The _node_ that originates a packet that will route a payment through some number of _hops_ to a _final node_. It is also the first _sending peer_ in a chain.
   * _See category: node_
   * _See related: final node, processing node_

* *Outpoint*:
  * A transaction hash and output index that uniquely identify an unspent transaction output. Needed to compose a new transaction, as an input.
  * _See related: funding transaction, commitment transaction_

* *Payment hash*:
   * The *HTLC* contains the payment hash, which is the hash of the
    *payment preimage*.
   * _See container: HTLC_
   * _See originator: payment preimage_

* *Payment preimage*:
   * Proof that payment has been received, held by
    the final recipient, who is the only person who knows this
    secret. The final recipient releases the preimage in order to
    release funds. The payment preimage is hashed as the *payment hash*
    in the *HTLC*.
   * _See container: HTLC_
   * _See derivation: payment hash_

* *Peers*:
   * Two *nodes* that are in communication with each other.
      * Two peers may gossip with each other prior to setting up a channel.
      * Two peers may establish a *channel* through which they transact.
   * _See related: node_

* *Penalty transaction*:
   * A transaction that spends all outputs of a *revoked commitment
    transaction*, using the *commitment revocation private key*. A *peer* uses this
    if the other peer tries to "cheat" by broadcasting a *revoked
    commitment transaction*.
   * _See related: closing transaction, commitment transaction, funding transaction_

* *Per-commitment secret*:
   * Every *commitment transaction* derives its keys from a per-commitment secret,
     which is generated such that the series of per-commitment secrets
     for all previous commitments can be stored compactly.
   * _See container: commitment transaction_
   * _See derivation: commitment revocation private key_

* *Processing node*:
   * A *node* that is processing a packet that originated with an *origin node* and that is being sent toward a *final node* in order to route a payment. It acts as a _receiving peer_ to receive the message, then a _sending peer_ to send on the packet.
   * _See category: node_
   * _See related: final node, origin node_

* *Receiving node*:
   * A *node* that is receiving a message.
   * _See category: node_
   * _See related: sending node_

* *Receiving peer*:
   * A *node* that is receiving a message from a directly connected *peer*.
   * _See category: peer_
   * _See related: sending peer_

* *Revoked commitment transaction*:
   * An old *commitment transaction* that has been revoked because a new commitment transaction has been negotiated.
   * _See category: commitment transaction_

* *Revoked transaction close*:
   * An invalid close of a *channel*, accomplished by broadcasting a *revoked
    commitment transaction*. Since the other *peer* knows the
    *commitment revocation secret key*, it can create a *penalty transaction*.
   * _See related: mutual close, unilateral close_

* *Route*: A path across the Lightning Network that enables a payment
    from an *origin node* to a *final node* across one or more
    *hops*.
  * _See related: channel_

* *Sending node*:
   * A *node* that is sending a message.
   * _See category: node_
   * _See related: receiving node_

* *Sending peer*:
   * A *node* that is sending a message to a directly connected *peer*.
   * _See category: peer_
   * _See related: receiving peer_.

* *Unilateral close*:
   * An uncooperative close of a *channel*, accomplished by broadcasting a
    *commitment transaction*. This transaction is larger (i.e. less
    efficient) than a *mutual close transaction*, and the *peer* whose
    commitment is broadcast cannot access its own outputs for some
    previously-negotiated duration.
   * _See related: mutual close, revoked transaction close_

## Authors

[ FIXME: Insert Author List ]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
