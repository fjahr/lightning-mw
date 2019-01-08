# BMW #2: Peer Protocol for Channel Management

The peer channel protocol has three phases: establishment, normal
operation, and closing.

# Table of Contents

  * [Channel](#channel)
    * [Channel Establishment](#channel-establishment)
      * [The `open_channel` Message](#the-open_channel-message)
      * [The `accept_channel` Message](#the-accept_channel-message)
      * [The `funding_created` Message](#the-funding_created-message)
      * [The `funding_signed` Message](#the-funding_signed-message)
      * [The `funding_locked` Message](#the-funding_locked-message)
    * [Channel Close](#channel-close)
      * [Closing Initiation: `shutdown`](#closing-initiation-shutdown)
      * [Closing Negotiation: `closing_signed`](#closing-negotiation-closing_signed)
    * [Normal Operation](#normal-operation)
      * [Forwarding HTLCs](#forwarding-htlcs)
      * [`cltv_expiry_delta` Selection](#cltv_expiry_delta-selection)
      * [Adding an HTLC: `update_add_htlc`](#adding-an-htlc-update_add_htlc)
      * [Removing an HTLC: `update_fulfill_htlc`, `update_fail_htlc`, and `update_fail_malformed_htlc`](#removing-an-htlc-update_fulfill_htlc-update_fail_htlc-and-update_fail_malformed_htlc)
      * [Committing Updates So Far: `commitment_signed`](#committing-updates-so-far-commitment_signed)
      * [Completing the Transition to the Updated State: `revoke_and_ack`](#completing-the-transition-to-the-updated-state-revoke_and_ack)
      * [Updating Fees: `update_fee`](#updating-fees-update_fee)
    * [Message Retransmission: `channel_reestablish` message](#message-retransmission)
  * [Authors](#authors)

# Channel

## Channel Establishment

After authenticating and initializing a connection ([BMW #8](08-transport.md)
and [BMW #1](01-messaging.md#the-init-message), respectively), channel establishment may begin.
This consists of the funding node (funder) sending an `open_channel` message,
followed by the responding node (fundee) sending `accept_channel`. With the
channel parameters locked in, the funder is able to create the funding
transaction and both versions of the commitment transaction in collaboration
with the fundee.

The fundee picks a blinding factor and sends it to the funder with `blinding_selected`.
The funder builds the commitment using the fundees blinding factor and sends
it to fundee by sending `commitment_created`. Using the commitment the fundee
builds a range proof for the funding amount and sends it to the funder as
`proof_created`. The funder then also creates their own range proof and
aggregates it with the range proof of the fundee. This completes the multisig
output which the funder adds to the rest of the outputs of the transaction
she has created, then sends it to the fundee with `funding_created`.

The fundee start the construction of the kernel by building the message (including
fee and lock height), the schnorr challenge and their side of the signature and
sending it all to the funder in `kernel_created`. The funder can check the
challenge herself and then sends back her side of the signature with the
`kernel_signed` message. At this point the fundee has the final kernel and
full signature and can broadcast it to the network. The fundee acknowledges
this by sending another `kernel_signed` message to the funder. Once both parties
have observed the transaction on chain they both notify each other using a
`funding_locked` message. Once both have received a `funding_locked` message
the channel is operational.

Further details on the construction of proofs and the transaction are described
in [BMW #3](03-transactions.md#bolt-3-bitcoin-transaction-and-script-formats).

        +-------+                              +-------+
        |       |--(1)---  open_channel  ----->|       |
        |       |<-(2)--  accept_channel  -----|       |
        |       |                              |       |
        |       |<-(3)-  blinding_selected  ---|       |
        |       |--(4)-  commitment_created  ->|       |
        |       |                              |       |
        |       |<-(5)---  proof_created  -----|       |
        |   A   |--(6)--  funding_created  --->|   B   |
        |       |                              |       |
        |       |<-(7)---  kernel_created  ----|       |
        |       |--(8)---  kernel_signed  ---->|       |
        |       |<-(9)-  funding_completed  ---|       |
        |       |                              |       |
        |       |--(10)-- funding_locked  ---->|       |
        |       |<-(11)-- funding_locked  -----|       |
        +-------+                              +-------+

        - where node A is 'funder' and node B is 'fundee'

If this fails at any stage, or if one node decides the channel terms
offered by the other node are not suitable, the channel establishment
fails.

Note that multiple channels can operate in parallel, as all channel
messages are identified by either a `temporary_channel_id` (before the
funding transaction is created) or a `channel_id` (derived from the
funding transaction).

### The `open_channel` Message

This message contains information about a node and indicates its
desire to set up a new channel. This is the first step toward creating
the funding transaction and both versions of the commitment transaction.

1. type: 32 (`open_channel`)
2. data:
   * [`32`:`chain_hash`]
   * [`32`:`temporary_channel_id`]
   * [`8`:`funding`]
   * [`8`:`push_funds`]
   * [`8`:`dust_limit`]
   * [`8`:`max_htlc_value_in_flight`]
   * [`8`:`channel_reserve`]
   * [`8`:`htlc_minimum`]
   * [`4`:`feerate_per_kw`]
   * [`2`:`to_self_delay`]
   * [`2`:`max_accepted_htlcs`]
   * [`33`:`revocation_basepoint`]
   * [`33`:`payment_basepoint`]
   * [`33`:`delayed_payment_basepoint`]
   * [`33`:`htlc_basepoint`]
   * [`33`:`first_per_commitment_point`]
   * [`1`:`channel_flags`]

The `chain_hash` value denotes the exact blockchain that the opened channel will
reside within. This is usually the genesis hash of the respective blockchain.
The existence of the `chain_hash` allows nodes to open channels
across many distinct blockchains as well as have channels within multiple
blockchains opened to the same peer (if it supports the target chains).

The `temporary_channel_id` is used to identify this channel on a per-peer basis until the
funding transaction is established, at which point it is replaced
by the `channel_id`, which is derived from the funding transaction.

`funding` is the amount the sender is putting into the
channel. `push_funds` is an amount of initial funds that the sender is
unconditionally giving to the receiver. `dust_limit` is the
threshold below which outputs should not be generated for this node's
commitment or HTLC transactions (i.e. HTLCs below this amount plus
HTLC transaction fees are not enforceable on-chain). This reflects the
reality that tiny outputs are not considered standard transactions and
will not propagate through the network. `channel_reserve`
is the minimum amount that the other node is to keep as a direct
payment. `htlc_minimum` indicates the smallest value HTLC this
node will accept.

`max_htlc_value_in_flight` is a cap on total value of outstanding
HTLCs, which allows a node to limit its exposure to HTLCs; similarly,
`max_accepted_htlcs` limits the number of outstanding HTLCs the other
node can offer.

`feerate_per_kw` indicates the initial fee rate per 1000-weight
(i.e. 1/4 the more normally-used 'mw-coin per 1000 vbytes') that this
side will pay for commitment and HTLC transactions, as described in
[BMW #3](03-transactions.md#fee-calculation) (this can be adjusted
later with an `update_fee` message).

`to_self_delay` is the number of blocks that the other node's to-self
outputs must be delayed, using timelock delays; this
is how long it will have to wait in case of breakdown before redeeming
its own funds.

The various `_basepoint` fields are used to derive unique
keys as described in [BMW #3](03-transactions.md#key-derivation) for each commitment
transaction. Varying these keys ensures that the transaction ID of
each commitment transaction is unpredictable to an external observer,
even if one commitment transaction is seen; this property is very
useful for preserving privacy when outsourcing penalty transactions to
third parties.

`first_per_commitment_point` is the per-commitment point to be used
for the first commitment transaction,

Only the least-significant bit of `channel_flags` is currently
defined: `announce_channel`. This indicates whether the initiator of
the funding flow wishes to advertise this channel publicly to the
network, as detailed within [BMW #7](07-routing-gossip.md#bolt-7-p2p-node-and-channel-discovery).

#### Requirements

The sending node:
  - MUST ensure the `chain_hash` value identifies the chain it wishes to open
  the channel within.
  - MUST ensure `temporary_channel_id` is unique from any other channel ID with
  the same peer.
  - MUST set `push_funds` to equal or less than 1000 * `funding`.
  - MUST set `revocation_basepoint`, `htlc_basepoint`, `payment_basepoint`, and
  `delayed_payment_basepoint` to valid DER-encoded, compressed, secp256k1 pubkeys.
  - MUST set `first_per_commitment_point` to the per-commitment point to be used
  for the initial commitment transaction, derived as specified in [BMW #3](03-transactions.md#per-commitment-secret-requirements).
  - MUST set `channel_reserve` greater than or equal to `dust_limit`.
  - MUST set undefined bits in `channel_flags` to 0.

The sending node SHOULD:
  - set `to_self_delay` sufficient to ensure the sender can irreversibly spend
  a commitment transaction output, in case of misbehavior by the receiver.
  - set `feerate_per_kw` to at least the rate it estimates would cause the
  transaction to be immediately included in a block.
  - set `dust_limit` to a sufficient value to allow commitment transactions
  to propagate through the Bitcoin network.
  - set `htlc_minimum` to the minimum value HTLC it's willing to accept from
  this peer.

The receiving node MUST:
  - ignore undefined bits in `channel_flags`.
  - if the connection has been re-established after receiving a previous
 `open_channel`, BUT before receiving a `funding_created` message:
    - accept a new `open_channel` message.
    - discard the previous `open_channel` message.

The receiving node MAY fail the channel if:
  - `announce_channel` is `false` (`0`), yet it wishes to publicly announce
  the channel.
  - `funding` is too small.
  - it considers `htlc_minimum` too large.
  - it considers `max_htlc_value_in_flight` too small.
  - it considers `channel_reserve` too large.
  - it considers `max_accepted_htlcs` too small.
  - it considers `dust_limit` too small and plans to rely on the sending node
  publishing its commitment transaction in the event of a data loss (see [message-retransmission](02-peer-protocol.md#message-retransmission)).

The receiving node MUST fail the channel if:
  - the `chain_hash` value is set to a hash of a chain that is unknown to the receiver.
  - `push_funds` is greater than `funding` * 1000.
  - `to_self_delay` is unreasonably large.
  - `max_accepted_htlcs` is greater than 483.
  - it considers `feerate_per_kw` too small for timely processing or unreasonably large.
  - `revocation_basepoint`, `htlc_basepoint`, `payment_basepoint`, or `delayed_payment_basepoint`
are not valid DER-encoded compressed secp256k1 pubkeys.
  - `dust_limit` is greater than `channel_reserve`.
  - the funder's amount for the initial commitment transaction is not sufficient
  for full [fee payment](03-transactions.md#fee-payment).
  - both `to_local` and `to_remote` amounts for the initial commitment
  transaction are less than or equal to `channel_reserve` (see [BMW #3](03-transactions.md#commitment-transaction-outputs)).

The receiving node MUST NOT:
  - consider funds received, using `push_funds`, to be received until the
  funding transaction has reached sufficient depth.

#### Rationale

The *channel reserve* is specified by the peer's `channel_reserve`: 1% of the
channel total is suggested. Each side of a channel maintains this reserve so
it always has something to lose if it were to try to broadcast an old, revoked
commitment transaction. Initially, this reserve may not be met, as only one
side has funds; but the protocol ensures that there is always progress toward
meeting this reserve, and once met, it is maintained.

The sender can unconditionally give initial funds to the receiver using a
non-zero `push_funding`, but even in this case we ensure that the funder has
sufficient remaining funds to pay fees and that one side has some amount it
can spend (which also implies there is at least one non-dust output). Note that,
like any other on-chain transaction, this payment is not certain until the
funding transaction has been confirmed sufficiently (with a danger of
double-spend until this occurs) and may require a separate method to prove
payment via on-chain confirmation.

The `feerate_per_kw` is generally only of concern to the sender (who pays the
fees), but there is also the fee rate paid by HTLC transactions; thus,
unreasonably large fee rates can also penalize the recipient.

Separating the `htlc_basepoint` from the `payment_basepoint` improves security:
a node needs the secret associated with the `htlc_basepoint` to produce HTLC
signatures for the protocol, but the secret for the `payment_basepoint` can be
in cold storage.

The requirement that `channel_reserve` is not considered dust
according to `dust_limit_satoshis` eliminates cases where all outputs
would be eliminated as dust.  The similar requirements in
`accept_channel` ensure that both sides' `channel_reserve`
are above both `dust_limit`.

Details for how to handle a channel failure can be found in
[BMW #5:Failing a Channel](05-onchain.md#failing-a-channel).

#### Practical Considerations for temporary_channel_id

Note that as duplicate `temporary_channel_id`s may exist from different
peers, APIs which reference channels by their channel id before the funding
transaction is created are inherently unsafe. The only protocol-provided
identifier for a channel before funding_created has been exchanged is the
(source_node_id, destination_node_id, temporary_channel_id) tuple. Note that
any such APIs which reference channels by their channel id before the funding
transaction is confirmed are also not persistent - until you know the script
pubkey corresponding to the funding output nothing prevents duplicative channel
ids.

#### Future

It would be easy to have a local feature bit which indicated that a
receiving node was prepared to fund a channel, which would reverse this
protocol.

### The `accept_channel` Message

This message contains information about a node and indicates its
acceptance of the new channel. This is the second step toward creating the
funding transaction and both versions of the commitment transaction.

1. type: 33 (`accept_channel`)
2. data:
   * [`32`:`temporary_channel_id`]
   * [`8`:`dust_limit`]
   * [`8`:`max_htlc_value_in_flight`]
   * [`8`:`channel_reserve`]
   * [`8`:`htlc_minimum`]
   * [`4`:`minimum_depth`]
   * [`2`:`to_self_delay`]
   * [`2`:`max_accepted_htlcs`]
   * [`33`:`revocation_basepoint`]
   * [`33`:`payment_basepoint`]
   * [`33`:`delayed_payment_basepoint`]
   * [`33`:`htlc_basepoint`]
   * [`33`:`first_per_commitment_point`]

#### Requirements

The `temporary_channel_id` MUST be the same as the `temporary_channel_id` in
the `open_channel` message.

The sender:
  - SHOULD set `minimum_depth` to a number of blocks it considers reasonable to
avoid double-spending of the funding transaction.
  - MUST set `channel_reserve` greater than or equal to `dust_limit` from the
  `open_channel` message.
  - MUST set `dust_limit` less than or equal to `channel_reserve` from the
  `open_channel` message.

The receiver:
  - if `minimum_depth` is unreasonably large:
    - MAY reject the channel.
  - if `channel_reserve` is less than `dust_limit` within the `open_channel` message:
	- MUST reject the channel.
  - if `channel_reserve` from the `open_channel` message is less than `dust_limit`:
	- MUST reject the channel.
Other fields have the same requirements as their counterparts in `open_channel`.


### The `blinding_selected` Message

This message sends the elliptic curve point that corresponds to the blinding factor
that the fundee has chosen.

1. type: XX (`blinding_selected`)
2. data:
    * [`32`:`temporary_channel_id`]
    * [`32`:`x_coordinate`]
    * [`2`:`y_parity`]

#### Requirements

The sender MUST set:
  - `temporary_channel_id` the same as the `temporary_channel_id` in the
  `open_channel` message.
  - `x_coordinate` and `y_parity` must be the a generator point scaled by
  blinding factor the fundee selected.

The recipient:
  - if the x and y coordinates are not a valid curve point:
    - MUST fail the channel.


### The `commitment_created` Message

This message sends the commitment that the funder has built including her own
blinding factor.

1. type: XX (`commitment_created`)
2. data:
    * [`32`:`temporary_channel_id`]
    * [`32`:`x_coordinate`]
    * [`2`:`y_parity`]

#### Requirements

The sender MUST set:
  - `temporary_channel_id` the same as the `temporary_channel_id` in the
  `open_channel` message.
  - `x_coordinate` and `y_parity` must be the a generator point scaled by
  blinding factor the fundee selected.

The recipient:
  - if the x and y coordinates are not a valid curve point:
    - MUST fail the channel.


### The `proof_created` Message

This message sends a proof for the funding amount created by the fundee back to
the funder.

1. type: XX (`proof_created`)
2. data:
    * [`32`:`temporary_channel_id`]
    * [`32`:`proof`]

#### Requirements

The sender MUST set:
  - `temporary_channel_id` the same as the `temporary_channel_id` in the `open_channel` message.

The recipient:
  - if range proof is not valid:
    - MUST fail the channel.


### The `funding_created` Message

This message acknowledges that the funding transaction is complete, thus
signalling to the fundee to start the process of creating the transaction
kernel. She sends along the transaction and a random nonce.

1. type: XX (`funding_created`)
2. data:
    * [`32`:`temporary_channel_id`]
    * [`32`:`nonce_x_coordinate`]
    * [`2`:`nonce_y_parity`]
    * [`32`:`tx_x_coordinate`]
    * [`2`:`tx_y_parity`]

#### Requirements

The sender MUST set:
  - `temporary_channel_id` the same as the `temporary_channel_id` in the `open_channel` message.
  - `nonce_x_coordinate` and `nonce_y_parity`, as well as `tx_x_coordinate` and `tx_y_parity`
  must be the generator points scaled by blinding factor the fundee selected.

The recipient:
  - if the x and y coordinates are not a valid curve point:
    - MUST fail the channel.


### The `kernel_created` Message

The fundee computes her side of the kernel signature including her random nonce
and sends those values over to the funder.

1. type: XX (`kernel_created`)
2. data:
    * [`32`:`temporary_channel_id`]
    * [`32`:`nonce_x_coordinate`]
    * [`2`:`nonce_y_parity`]
    * [`32`:`signature`]

#### Requirements

The sender MUST set:
  - `temporary_channel_id` the same as the `temporary_channel_id` in the `open_channel` message.
  - `nonce_x_coordinate` and `nonce_y_parity`
  must be a generator point scaled by the nonce the funder selected.

The recipient:
  - if the x and y coordinates are not a valid curve point:
    - MUST fail the channel.


### The `kernel_signed` Message

The funder checks the signature of the fundee and then creates her side of the
signature.

1. type: XX (`kernel_signed`)
2. data:
    * [`32`:`temporary_channel_id`]
    * [`32`:`signature`]

#### Requirements

The sender MUST set:
  - `temporary_channel_id` the same as the `temporary_channel_id` in the `open_channel` message.


### The `funding_completed` Message

The fundee checks the signature of the funder and then is able to create the
final signature as well as the final kernel.

1. type: XX (`funding_completed`)
2. data:
    * [`32`:`temporary_channel_id`]
    * [`32`:`final_sig`]
    * [`32`:`final_kernel`]

#### Requirements

The sender MUST set:
  - `temporary_channel_id` the same as the `temporary_channel_id` in the `open_channel` message.

### The `funding_locked` Message

This message introduces the channel_id to identify the channel.

FIXME: Decide how the channel_id should be derived

This message indicates that the funding transaction has reached the
`minimum_depth` asked for in `accept_channel`. Once both nodes have
sent this, the channel enters normal operating mode.

1. type: 36 (`funding_locked`)
2. data:
    * [`32`:`channel_id`]
    * [`33`:`next_per_commitment_point`]

#### Requirements

The sender MUST:
  - wait until the funding transaction has reached
`minimum_depth` before sending this message.
  - set `next_per_commitment_point` to the
per-commitment point to be used for the following commitment
transaction, derived as specified in
[BMW #3](03-transactions.md#per-commitment-secret-requirements).

A non-funding node (fundee):
  - SHOULD forget the channel if it does not see the
funding transaction after a reasonable timeout.

From the point of waiting for `funding_locked` onward, either node MAY
fail the channel if it does not receive a required response from the
other node after a reasonable timeout.

#### Rationale

The non-funder can simply forget the channel ever existed, since no
funds are at risk. If the fundee were to remember the channel forever, this
would create a Denial of Service risk; therefore, forgetting it is recommended
(even if the promise of `push_funds` is significant).


## Channel Close

Nodes can negotiate a mutual close of the connection, which unlike a
unilateral close, allows them to access their funds immediately and
can be negotiated with lower fees.

Closing happens in two stages:
1. one side indicates it wants to clear the channel (and thus will accept no new HTLCs)
2. once all HTLCs are resolved, the final channel close negotiation begins.

        +-------+                              +-------+
        |       |--(1)-----  shutdown  ------->|       |
        |       |<-(2)-----  shutdown  --------|       |
        |       |                              |       |
        |       | <complete all pending HTLCs> |       |
        |       |                 ...          |       |
        |       |                              |       |
        |       |--(3)--- closing_fee   F1---->|       |
        |       |<-(4)--- closing_fee   F2-----|       |
        |   A   |              ...             |   B   |
        |       |--(?)--- closing_fee   Fn---->|       |
        |       |<-(?)--- closing_fee   Fn-----|       |
        |       |                              |       |
        |       |--(5)--  partial_closing  --->|       |
        |       |<-(6)----  full_closing  -----|       |
        |       |                              |       |
        |       |--(7)--  closing_signed  ---->|       |
        |       |<-(8)--  closing_complete  ---|       |
        +-------+                              +-------+

### Closing Initiation: `shutdown`

Either node (or both) can send a `shutdown` message to initiate closing. This
means the node will not accept any new HTLCs from this point forward.

1. type: 38 (`shutdown`)
2. data:
   * [`32`:`channel_id`]

#### Requirements

A sending node:
  - if it hasn't sent a `funding_created` (if it is a funder) or a `funding_signed` (if it is a fundee):
    - MUST NOT send a `shutdown`
  - MAY send a `shutdown` before a `funding_locked`, i.e. before the funding transaction has reached `minimum_depth`.
  - if there are updates pending on the receiving node's commitment transaction:
    - MUST NOT send a `shutdown`.
  - MUST NOT send an `update_add_htlc` after a `shutdown`.
  - if no HTLCs remain in either commitment transaction:
  - MUST NOT send any `update` message after a `shutdown`.
  - SHOULD fail to route any HTLC added after it has sent `shutdown`.

A receiving node:
  - if it hasn't received a `funding_signed` (if it is a funder) or a `funding_created` (if it is a fundee):
    - SHOULD fail the connection
  - if it hasn't sent a `funding_locked` yet:
    - MAY reply to a `shutdown` message with a `shutdown`
  - once there are no outstanding updates on the peer, UNLESS it has already sent a `shutdown`:
    - MUST reply to a `shutdown` message with a `shutdown`

#### Rationale

If channel state is always "clean" (no pending changes) when a
shutdown starts, the question of how to behave if it wasn't is avoided:
the sender always sends a `commitment_signed` first.

As shutdown implies a desire to terminate, it implies that no new
HTLCs will be added or accepted. Once any HTLCs are cleared, the peer
may immediately begin closing negotiation, so we ban further updates
to the commitment transaction (in particular, `update_fee` would be
possible otherwise).

The `shutdown` response requirement implies that the node sends
`commitment_signed` to commit any outstanding changes before replying; however,
it could theoretically reconnect instead, which would simply erase all
outstanding uncommitted changes.


### Closing Negotiation: `closing_fee`

Once shutdown is complete and the channel is empty of HTLCs, the final
current commitment transactions will have no HTLCs, and closing fee
negotiation begins.  The funder chooses a fee it thinks is fair, and sends
the suggestion in a `closing_fee` message. The other node then replies similarly,
using a fee it thinks is fair.  This exchange continues until both agree on the
same fee or when one side fails the channel.

1. type: 39 (`closing_fee`)
2. data:
   * [`32`:`channel_id`]
   * [`8`:`fee`]

#### Requirements

The funding node:
  - after `shutdown` has been received, AND no HTLCs remain in either commitment transaction:
    - SHOULD send a `closing_fee` message.

The sending node:
  - MUST set `fee` less than or equal to the
 base fee of the final commitment transaction, as calculated in [BMW #3](03-transactions.md#fee-calculation).
  - SHOULD set the initial `fee` according to its
 estimate of cost of inclusion in a block.

The receiving node:
  - if `fee` is equal to its previously sent `fee`:
    - SHOULD continue to build the closing transaction.
  - otherwise, if `fee` is greater than
the base fee of the final commitment transaction as calculated in
[BMW #3](03-transactions.md#fee-calculation):
    - MUST fail the connection.
  - if `fee` is not strictly
between its last-sent `fee` and its previously-received
`fee`, UNLESS it has since reconnected:
    - SHOULD fail the connection.
  - if the receiver agrees with the fee:
    - SHOULD reply with a `closing_fee` with the same `fee` value.
  - otherwise:
    - MUST propose a value "strictly between" the received `fee`
  and its previously-sent `fee`.

#### Rationale

The "strictly between" requirement ensures that forward
progress is made, even if only by small increments at a time. To avoid
keeping state and to handle the corner case, where fees have shifted
between disconnection and reconnection, negotiation restarts on reconnection.

Note there is limited risk if the closing transaction is
delayed, but it will be broadcast very soon; so there is usually no
reason to pay a premium for rapid processing.

### The `partial_closing` Message

The initiator or the channel closing constructs her part of the transaction
and the partial transaction along with a random nonce and her blinding
factor.

1. type: XX (`partial_closing`)
2. data:
    * [`32`:`channel_id`]
    * [`32`:`nonce_x_coordinate`]
    * [`2`:`nonce_y_parity`]
    * [`32`:`blinding_x_coordinate`]
    * [`2`:`blinding_y_parity`]

#### Requirements

The sender MUST set:
  - `nonce_x_coordinate` and `nonce_y_parity`, as well as `blinding_x_coordinate` and `blinding_y_parity`
  must be the generator points scaled by blinding factor the fundee selected.

The recipient:
  - if the x and y coordinates are not a valid curve point:
    - MUST fail the channel.


### The `full_closing` Message

The taker of the closing chooses a random nonce and a blinding factor for her
output as well and sends it along with her partial signature for the transaction.

1. type: XX (`full_closing`)
2. data:
    * [`32`:`channel_id`]
    * [`32`:`nonce_x_coordinate`]
    * [`2`:`nonce_y_parity`]
    * [`32`:`blinding_x_coordinate`]
    * [`2`:`blinding_y_parity`]
    * [`32`:`signature`]

#### Requirements

The sender MUST set:
  - `nonce_x_coordinate` and `nonce_y_parity` as well as `blinding_x_coordinate`
  and `blinding_y_coordinate` must be a generator point scaled by the nonce the
  funder selected.

The recipient:
  - if the x and y coordinates are not a valid curve point:
    - MUST fail the channel.

### The `closing_signed` Message

Using the data received from the taker, the initiator can create her side of the
signature and sends it over to the taker.

1. type: XX (`closing_signed`)
2. data:
    * [`32`:`channel_id`]
    * [`32`:`signature`]


### The `closing_complete` Message

The taker now has all the data and can complete the full closing transaction
which she sends to the initiator as well as broadcasting it to the network.

1. type: XX (`closing_completed`)
2. data:
    * [`32`:`channel_id`]
    * [`32`:`final_sig`]
    * [`32`:`final_kernel`]

#### Requirements

The sender MUST set:
  - `temporary_channel_id` the same as the `temporary_channel_id` in the `open_channel` message.


### The `closing_complete` Message

This message introduces the channel_id to identify the channel.

FIXME: Decide how the channel_id should be derived

This message indicates that the funding transaction has reached the
`minimum_depth` asked for in `accept_channel`. Once both nodes have
sent this, the channel enters normal operating mode.

1. type: 36 (`funding_locked`)
2. data:
    * [`32`:`channel_id`]
    * [`33`:`next_per_commitment_point`]

#### Requirements

The sender MUST:
  - wait until the funding transaction has reached
`minimum_depth` before sending this message.
  - set `next_per_commitment_point` to the
per-commitment point to be used for the following commitment
transaction, derived as specified in
[BMW #3](03-transactions.md#per-commitment-secret-requirements).

A non-funding node (fundee):
  - SHOULD forget the channel if it does not see the
funding transaction after a reasonable timeout.

From the point of waiting for `funding_locked` onward, either node MAY
fail the channel if it does not receive a required response from the
other node after a reasonable timeout.

#### Rationale

The non-funder can simply forget the channel ever existed, since no
funds are at risk. If the fundee were to remember the channel forever, this
would create a Denial of Service risk; therefore, forgetting it is recommended
(even if the promise of `push_funds` is significant).


## Normal Operation

Once both nodes have exchanged `funding_locked` (and optionally
[`announcement_signatures`](07-routing-gossip.md#the-announcement_signatures-message)),
the channel can be used to make payments via Hashed Time Locked Contracts.

Changes are sent in batches: one or more `update_` messages are sent before a
`commitment_signed` message, as in the following diagram:


        +-------+                              +-------+
        |       |--(1)------  add_htlc  ------>|       |
        |       |<-(2)- commitments_created  --|       |
        |       |                              |       |
        |       |--(3)--  proofs_created  ---->|       |
        |   A   |<-(4)- transactions_created --|   B   |
        |       |                              |       |
        |       |--(5)---  kernels_created  -->|       |
        |       |<-(6)---  kernels_signed  ----|       |
        |       |                              |       |
        |       |--(7)---  htlc_complete  ---->|       |
        |       |<-(8)---  revoke_and_ack  ----|       |
        +-------+                              +-------+


Counter-intuitively, these updates apply to the *other node's*
commitment transaction; the node only adds those updates to its own
commitment transaction when the remote node acknowledges it has
applied them via `revoke_and_ack`.

Thus each update traverses through the following states:

1. pending on the receiver
2. in the receiver's latest commitment transaction
3. ... and the receiver's previous commitment transaction has been revoked,
   and the HTLC is pending on the sender
4. ... and in the sender's latest commitment transaction
5. ... and the sender's previous commitment transaction has been revoked


As the two nodes' updates are independent, the two commitment
transactions may be out of sync indefinitely. This is not concerning:
what matters is whether both sides have irrevocably committed to a
particular HTLC or not (the final state, above).

### Forwarding HTLCs

In general, a node offers HTLCs for two reasons: to initiate a payment of its own,
or to forward another node's payment. In the forwarding case, care must
be taken to ensure the *outgoing* HTLC cannot be redeemed unless the *incoming*
HTLC can be redeemed. The following requirements ensure this is always true.

The respective **addition/removal** of an HTLC is considered *irrevocably committed* when:

1. The commitment transaction **with/without** it is committed to by both nodes, and any
previous commitment transaction **without/with** it has been revoked, OR
2. The commitment transaction **with/without** it has been irreversibly committed to
the blockchain.

#### Requirements

A node:
  - until the incoming HTLC has been irrevocably committed:
    - MUST NOT offer an HTLC (`update_add_htlc`) in response to an incoming HTLC.
  - until the removal of the outgoing HTLC is irrevocably committed, OR until the
  outgoing on-chain HTLC output has been spent via the HTLC-timeout transaction
  (with sufficient depth):
    - MUST NOT fail an incoming HTLC (`update_fail_htlc`) for which it has committed
to an outgoing HTLC.
  - once its `cltv_expiry` has been reached, OR if `cltv_expiry` minus
  `current_height` is less than `cltv_expiry_delta` for the outgoing channel:
    - MUST fail an incoming HTLC (`update_fail_htlc`).
  - if an incoming HTLC's `cltv_expiry` is unreasonably far in the future:
    - SHOULD fail that incoming HTLC (`update_fail_htlc`).
  - upon receiving an `update_fulfill_htlc` for the outgoing HTLC, OR upon discovering
  the `payment_preimage` from an on-chain HTLC spend:
    - MUST fulfill an incoming HTLC for which it has committed to an outgoing HTLC.

#### Rationale

In general, one side of the exchange needs to be dealt with before the other.
Fulfilling an HTLC is different: knowledge of the preimage is, by definition,
irrevocable and the incoming HTLC should be fulfilled as soon as possible to
reduce latency.

An HTLC with an unreasonably long expiry is a denial-of-service vector and
therefore is not allowed. Note that the exact value of "unreasonable" is currently unclear
and may depend on network topology.

### `cltv_expiry_delta` Selection

Once an HTLC has timed out, it can either be fulfilled or timed-out;
care must be taken around this transition, both for offered and received HTLCs.

Consider the following scenario, where A sends an HTLC to B, who
forwards to C, who delivers the goods as soon as the payment is
received.

1. C needs to be sure that the HTLC from B cannot time out, even if B becomes
   unresponsive; i.e. C can fulfill the incoming HTLC on-chain before B can
   time it out on-chain.

2. B needs to be sure that if C fulfills the HTLC from B, it can fulfill the
   incoming HTLC from A; i.e. B can get the preimage from C and fulfill the incoming
   HTLC on-chain before A can time it out on-chain.

The critical settings here are the `cltv_expiry_delta` in
[BMW #7](07-routing-gossip.md#the-channel_update-message) and the
related `min_final_cltv_expiry` in [BMW #11](11-payment-encoding.md#tagged-fields).
`cltv_expiry_delta` is the minimum difference in HTLC CLTV timeouts, in
the forwarding case (B). `min_final_cltv_expiry` is the minimum difference
between HTLC CLTV timeout and the current block height, for the
terminal case (C).

Note that if this value is too low for a channel, the risk is only to
the node *accepting* the HTLC, not the node offering it. For this
reason, the `cltv_expiry_delta` for the *outgoing* channel is used as
the delta across a node.

The worst-case number of blocks between outgoing and
incoming HTLC resolution can be derived, given a few assumptions:

* a worst-case reorganization depth `R` blocks
* a grace-period `G` blocks after HTLC timeout before giving up on
  an unresponsive peer and dropping to chain
* a number of blocks `S` between transaction broadcast and the
  transaction being included in a block

The worst case is for a forwarding node (B) that takes the longest
possible time to spot the outgoing HTLC fulfillment and also takes
the longest possible time to redeem it on-chain:

1. The B->C HTLC times out at block `N`, and B waits `G` blocks until
   it gives up waiting for C. B or C commits to the blockchain,
   and B spends HTLC, which takes `S` blocks to be included.
2. Bad case: C wins the race (just) and fulfills the HTLC, B only sees
   that transaction when it sees block `N+G+S+1`.
3. Worst case: There's reorganization `R` deep in which C wins and
   fulfills. B only sees transaction at `N+G+S+R`.
4. B now needs to fulfill the incoming A->B HTLC, but A is unresponsive: B waits `G` more
   blocks before giving up waiting for A. A or B commits to the blockchain.
5. Bad case: B sees A's commitment transaction in block `N+G+S+R+G+1` and has
   to spend the HTLC output, which takes `S` blocks to be mined.
6. Worst case: there's another reorganization `R` deep which A uses to
   spend the commitment transaction, so B sees A's commitment
   transaction in block `N+G+S+R+G+R` and has to spend the HTLC output, which
   takes `S` blocks to be mined.
7. B's HTLC spend needs to be at least `R` deep before it times out,
   otherwise another reorganization could allow A to timeout the
   transaction.

Thus, the worst case is `3R+2G+2S`, assuming `R` is at least 1. Note that the
chances of three reorganizations in which the other node wins all of them is
low for `R` of 2 or more. Since high fees are used (and HTLC spends can use
almost arbitrary fees), `S` should be small; although, given that block times are
irregular and empty blocks still occur, `S=2` should be considered a
minimum. Similarly, the grace period `G` can be low (1 or 2), as nodes are
required to timeout or fulfill as soon as possible; but if `G` is too low it increases the
risk of unnecessary channel closure due to networking delays.

There are four values that need be derived:

1. the `cltv_expiry_delta` for channels, `3R+2G+2S`: if in doubt, a
   `cltv_expiry_delta` of 12 is reasonable (R=2, G=1, S=2).

2. the deadline for offered HTLCs: the deadline after which the channel has to be failed
   and timed out on-chain. This is `G` blocks after the HTLC's
   `cltv_expiry`: 1 block is reasonable.

3. the deadline for received HTLCs this node has fulfilled: the deadline after which
the channel has to be failed and the HTLC fulfilled on-chain before its
   `cltv_expiry`. See steps 4-7 above, which imply a deadline of `2R+G+S`
   blocks before `cltv_expiry`: 7 blocks is reasonable.

4. the minimum `cltv_expiry` accepted for terminal payments: the
   worst case for the terminal node C is `2R+G+S` blocks (as, again, steps
   1-3 above don't apply). The default in
   [BMW #11](11-payment-encoding.md) is 9, which is slightly more
   conservative than the 7 that this calculation suggests.

#### Requirements

An offering node:
  - MUST estimate a timeout deadline for each HTLC it offers.
  - MUST NOT offer an HTLC with a timeout deadline before its `cltv_expiry`.
  - if an HTLC which it offered is in either node's current
  commitment transaction, AND is past this timeout deadline:
    - MUST fail the channel.

A fulfilling node:
  - for each HTLC it is attempting to fulfill:
    - MUST estimate a fulfillment deadline.
  - MUST fail (and not forward) an HTLC whose fulfillment deadline is already past.
  - if an HTLC it has fulfilled is in either node's current commitment
  transaction, AND is past this fulfillment deadline:
    - MUST fail the connection.

### Adding an HTLC: `update_add_htlc`

Either node can send `update_add_htlc` to offer an HTLC to the other,
which is redeemable in return for a payment preimage.

The format of the `onion_routing_packet` portion, which indicates where the payment
is destined, is described in [BMW #4](04-onion-routing.md).

1. type: 128 (`update_add_htlc`)
2. data:
   * [`32`:`channel_id`]
   * [`8`:`id`]
   * [`8`:`amount`]
   * [`32`:`payment_hash`]
   * [`4`:`cltv_expiry`]
   * [`1366`:`onion_routing_packet`]

#### Requirements

A sending node:
  - MUST NOT offer `amount` it cannot pay for in the
remote commitment transaction at the current `feerate_per_kw` (see "Updating
Fees") while maintaining its channel reserve.
  - MUST offer `amount` greater than 0.
  - MUST NOT offer `amount` below the receiving node's `htlc_minimum`
  - MUST set `cltv_expiry` less than 500000000.
  - for channels with `chain_hash` identifying the Bitcoin blockchain:
    - MUST set the four most significant bytes of `amount` to 0.
  - if result would be offering more than the remote's
  `max_accepted_htlcs` HTLCs, in the remote commitment transaction:
    - MUST NOT add an HTLC.
  - if the sum of total offered HTLCs would exceed the remote's
`max_htlc_value_in_flight`:
    - MUST NOT add an HTLC.
  - for the first HTLC it offers:
    - MUST set `id` to 0.
  - MUST increase the value of `id` by 1 for each successive offer.

A receiving node:
  - receiving an `amount` equal to 0, OR less than its own `htlc_minimum`:
    - SHOULD fail the channel.
  - receiving an `amount` that the sending node cannot afford at the current
  `feerate_per_kw` (while maintaining its channel reserve):
    - SHOULD fail the channel.
  - if a sending node adds more than its `max_accepted_htlcs` HTLCs to
    its local commitment transaction, OR adds more than its
    `max_htlc_value_in_flight` worth of offered HTLCs to its local commitment
    transaction:
    - SHOULD fail the channel.
  - if sending node sets `cltv_expiry` to greater or equal to 500000000:
    - SHOULD fail the channel.
  - for channels with `chain_hash` identifying the Bitcoin blockchain, if the
  four most significant bytes of `amount` are not 0:
    - MUST fail the channel.
  - MUST allow multiple HTLCs with the same `payment_hash`.
  - if the sender did not previously acknowledge the commitment of that HTLC:
    - MUST ignore a repeated `id` value after a reconnection.
  - if other `id` violations occur:
    - MAY fail the channel.

The `onion_routing_packet` contains an obfuscated list of hops and instructions
for each hop along the path.
It commits to the HTLC by setting the `payment_hash` as associated data, i.e.
includes the `payment_hash` in the computation of HMACs.
This prevents replay attacks that would reuse a previous `onion_routing_packet`
with a different `payment_hash`.

#### Rationale

Invalid amounts are a clear protocol violation and indicate a breakdown.

If a node did not accept multiple HTLCs with the same payment hash, an
attacker could probe to see if a node had an existing HTLC. This
requirement, to deal with duplicates, leads to the use of a separate
identifier; its assumed a 64-bit counter never wraps.

Retransmissions of unacknowledged updates are explicitly allowed for
reconnection purposes; allowing them at other times simplifies the
recipient code (though strict checking may help debugging).

`max_accepted_htlcs` is limited to 483 to ensure that, even if both
sides send the maximum number of HTLCs, the `commitment_signed` message will
still be under the maximum message size. It also ensures that
a single penalty transaction can spend the entire commitment transaction,
as calculated in [BMW #5](05-onchain.md#penalty-transaction-weight-calculation).

`cltv_expiry` values equal to or greater than 500000000 would indicate a time in
seconds, and the protocol only supports an expiry in blocks.

`amount` is deliberately limited for this version of the
specification; larger amounts are not necessary, nor wise, during the
bootstrap phase of the network.

### Removing an HTLC: `update_fulfill_htlc`, `update_fail_htlc`, and `update_fail_malformed_htlc`

For simplicity, a node can only remove HTLCs added by the other node.
There are four reasons for removing an HTLC: the payment preimage is supplied,
it has timed out, it has failed to route, or it is malformed.

To supply the preimage:

1. type: 130 (`update_fulfill_htlc`)
2. data:
   * [`32`:`channel_id`]
   * [`8`:`id`]
   * [`32`:`payment_preimage`]

For a timed out or route-failed HTLC:

1. type: 131 (`update_fail_htlc`)
2. data:
   * [`32`:`channel_id`]
   * [`8`:`id`]
   * [`2`:`len`]
   * [`len`:`reason`]

The `reason` field is an opaque encrypted blob for the benefit of the
original HTLC initiator, as defined in [BMW #4](04-onion-routing.md);
however, there's a special malformed failure variant for the case where
the peer couldn't parse it: in this case the current node instead takes action, encrypting
it into a `update_fail_htlc` for relaying.

For an unparsable HTLC:

1. type: 135 (`update_fail_malformed_htlc`)
2. data:
   * [`32`:`channel_id`]
   * [`8`:`id`]
   * [`32`:`sha256_of_onion`]
   * [`2`:`failure_code`]

#### Requirements

A node:
  - SHOULD remove an HTLC as soon as it can.
  - SHOULD fail an HTLC which has timed out.
  - until the corresponding HTLC is irrevocably committed in both sides'
  commitment transactions:
    - MUST NOT send an `update_fulfill_htlc`, `update_fail_htlc`, or
`update_fail_malformed_htlc`.

A receiving node:
  - if the `id` does not correspond to an HTLC in its current commitment transaction:
    - MUST fail the channel.
  - if the `payment_preimage` value in `update_fulfill_htlc`
  doesn't SHA256 hash to the corresponding HTLC `payment_hash`:
    - MUST fail the channel.
  - if the `BADONION` bit in `failure_code` is not set for
  `update_fail_malformed_htlc`:
    - MUST fail the channel.
  - if the `sha256_of_onion` in `update_fail_malformed_htlc` doesn't match the
  onion it sent:
    - MAY retry or choose an alternate error response.
  - otherwise, a receiving node which has an outgoing HTLC canceled by `update_fail_malformed_htlc`:
    - MUST return an error in the `update_fail_htlc` sent to the link which
      originally sent the HTLC, using the `failure_code` given and setting the
      data to `sha256_of_onion`.

#### Rationale

A node that doesn't time out HTLCs risks channel failure (see
[`cltv_expiry_delta` Selection](#cltv_expiry_delta-selection)).

A node that sends `update_fulfill_htlc`, before the sender, is also
committed to the HTLC and risks losing funds.

If the onion is malformed, the upstream node won't be able to extract
the shared key to generate a response — hence the special failure message, which
makes this node do it.

The node can check that the SHA256 that the upstream is complaining about
does match the onion it sent, which may allow it to detect random bit
errors. However, without re-checking the actual encrypted packet sent,
it won't know whether the error was its own or the remote's; so
such detection is left as an option.


### The `commitments_created` Message

This message sends the commitments to the updated remote commitment transaction.
These commitments are: The updated commitment transaction with it's new outputs
and the three HTLC transactions themselves (the hashlocked one
paying the local party, one with relative timelock to the remote party, one with
the revocation hash to the remote party). Since the latter two HTLC transactions
share the same inputs and outputs they can also use the same commitment.


1. type: XX (`commitments_created`)
2. data:
    * [`32`:`channel_id`]
    * [`8`:`id`]
    * [`32`:`commitment_x_coordinate`]
    * [`2`:`commitment_y_parity`]
    * [`32`:`htlc_local_x_coordinate`]
    * [`2`:`htlc_local_y_parity`]
    * [`32`:`htlc_remote_x_coordinate`]
    * [`2`:`htlc_remote_y_parity`]


#### Requirements

The sender MUST set:
  - `x_coordinate` and `y_parity` must be the a generator point scaled by
  blinding factor the fundee selected.

The recipient:
  - if the x and y coordinates are not a valid curve point:
    - MUST fail the channel.


### The `proofs_created` Message

This message sends proofs for the newly created transactions.

1. type: XX (`proofs_created`)
2. data:
    * [`32`:`channel_id`]
    * [`8`:`id`]
    * [`32`:`commitment_proof`]
    * [`32`:`htlc_local_proof`]
    * [`32`:`htlc_remote_proof`]

#### Requirements

The recipient:
  - if range proofs are not valid:
    - MUST fail the channel.


### The `transactions_created` Message

This message acknowledges that the transactions are all complete, thus
signalling to start the process of creating the transaction kernels.
For the kernels the selected nonces and transactions are sent along.

1. type: XX (`transactions_created`)
2. data:
    * [`32`:`channel_id`]
    * [`8`:`id`]
    * [`32`:`commitment_proof`]
    * [`32`:`commitment_nonce_x_coordinate`]
    * [`2`:`commitment_nonce_y_parity`]
    * [`32`:`commitment_tx_x_coordinate`]
    * [`2`:`commitment_tx_y_parity`]
    * [`32`:`htlc_timeout_proof`]
    * [`32`:`htlc_timeout_nonce_x_coordinate`]
    * [`2`:`htlc_timeout_nonce_y_parity`]
    * [`32`:`htlc_timeout_tx_x_coordinate`]
    * [`2`:`htlc_timeout_tx_y_parity`]
    * [`32`:`htlc_revoke_proof`]
    * [`32`:`htlc_revoke_nonce_x_coordinate`]
    * [`2`:`htlc_revoke_nonce_y_parity`]
    * [`32`:`htlc_revoke_tx_x_coordinate`]
    * [`2`:`htlc_revoke_tx_y_parity`]
    * [`32`:`htlc_hashlock_proof`]
    * [`32`:`htlc_hashlock_nonce_x_coordinate`]
    * [`2`:`htlc_hashlock_nonce_y_parity`]
    * [`32`:`htlc_hashlock_tx_x_coordinate`]
    * [`2`:`htlc_hashlock_tx_y_parity`]


#### Requirements

The sender MUST set:
  - `nonce_x_coordinate` and `nonce_y_parity`, as well as `tx_x_coordinate` and `tx_y_parity`
  must be the generator points scaled by blinding factor the fundee selected.

The recipient:
  - if the x and y coordinates are not a valid curve point:
    - MUST fail the channel.


### The `kernels_created` Message

The message includes one side of the kernel signatures including the random nonces
for all the open transactions and sends those values over to the other party.

1. type: XX (`kernels_created`)
2. data:
    * [`32`:`channel_id`]
    * [`8`:`id`]
    * [`32`:`commitment_nonce_x_coordinate`]
    * [`2`:`commitment_nonce_y_parity`]
    * [`32`:`commitment_signature`]
    * [`32`:`htlc_timeout_nonce_x_coordinate`]
    * [`2`:`htlc_timeout_nonce_y_parity`]
    * [`32`:`htlc_timeout_signature`]
    * [`32`:`htlc_revoke_nonce_x_coordinate`]
    * [`2`:`htlc_revoke_nonce_y_parity`]
    * [`32`:`htlc_revoke_signature`]
    * [`32`:`htlc_hashlock_nonce_x_coordinate`]
    * [`2`:`htlc_hashlock_nonce_y_parity`]
    * [`32`:`htlc_hashlock_signature`]


#### Requirements

The sender MUST set:
  - `nonce_x_coordinate` and `nonce_y_parity`
  must be a generator point scaled by the nonce the funder selected.

The recipient:
  - if the x and y coordinates are not a valid curve point:
    - MUST fail the channel.


### The `kernels_signed` Message

The signatures of the remote party are checked and then the local part of the
signatures sent back.

1. type: XX (`kernels_signed`)
2. data:
    * [`32`:`channel_id`]
    * [`8`:`id`]
    * [`32`:`commitment_signature`]
    * [`32`:`htlc_timeout_signature`]
    * [`32`:`htlc_revoke_signature`]
    * [`32`:`htlc_hashlock_signature`]


#### Requirements


### The `htlc_completed` Message

The local node checks the signature of the remote and then is able to create
the final signatures as well as the final kernels for all transactions.

1. type: XX (`htlc_completed`)
2. data:
    * [`32`:`channel_id`]
    * [`8`:`id`]
    * [`32`:`commitment_final_sig`]
    * [`32`:`htlc_timeout_final_sig`]
    * [`32`:`htlc_revoke_final_sig`]
    * [`32`:`htlc_hashlock_final_sig`]
    * [`32`:`commitment_final_kernel`]
    * [`32`:`htlc_timeout_final_kernel`]
    * [`32`:`htlc_revoke_final_kernel`]
    * [`32`:`htlc_hashlock_final_kernel`]


#### Requirements


### Completing the Transition to the Updated State: `revoke_and_ack`

Once the recipient of `htlc_completed` checks the signatures and knows
it has a valid new commitment transaction, it replies with the commitment
preimage for the previous commitment transaction in a `revoke_and_ack`
message.

This message also implicitly serves as an acknowledgment of receipt
of the `htlc_completed`, so this is a logical time for the `htlc_completed` sender
to apply (to its own commitment) any pending updates it sent before
that `htlc_completed`.

1. type: 133 (`revoke_and_ack`)
2. data:
   * [`32`:`channel_id`]
   * [`32`:`old_commitment_secret`]

#### Requirements

A sending node:
  - MUST set `old_commitment_secret` to the secret used to generate the hashlock
  of the old state.

A receiving node:
  - if `old_commitment_secret` does not solve the previous hashlock:
    - MUST fail the channel.

A node:
  - MUST NOT broadcast old (revoked) commitment transactions,
    - Note: doing so will allow the other node to seize all channel funds.
  - SHOULD NOT sign commitment transactions, unless it's about to broadcast
  them (due to a failed connection),
    - Note: this is to reduce the above risk.

### Updating Fees: `update_fee`

An `update_fee` message is sent by the node which is paying the
fee. Like any update, it's first committed to the receiver's
commitment transaction and then (once acknowledged) committed to the
sender's. Unlike an HTLC, `update_fee` is never closed but simply
replaced.

There is a possibility of a race, as the recipient can add new HTLCs
before it receives the `update_fee`. Under this circumstance, the sender may
not be able to afford the fee on its own commitment transaction, once the `update_fee`
is finally acknowledged by the recipient. In this case, the fee will be less
than the fee rate, as described in [BMW #3](03-transactions.md#fee-payment).

The exact calculation used for deriving the fee from the fee rate is
given in [BMW #3](03-transactions.md#fee-calculation).

1. type: 134 (`update_fee`)
2. data:
   * [`32`:`channel_id`]
   * [`4`:`feerate_per_kw`]

#### Requirements

The node _responsible_ for paying the fee:
  - SHOULD send `update_fee` to ensure the current fee rate is sufficient (by a
      significant margin) for timely processing of the commitment transaction.

The node _not responsible_ for paying the Bitcoin fee:
  - MUST NOT send `update_fee`.

A receiving node:
  - if the `update_fee` is too low for timely processing, OR is unreasonably large:
    - SHOULD fail the channel.
  - if the sender is not responsible for paying the Bitcoin fee:
    - MUST fail the channel.
  - if the sender cannot afford the new fee rate on the receiving node's
  current commitment transaction:
    - SHOULD fail the channel,
      - but MAY delay this check until the `update_fee` is committed.

#### Rationale

Fees are required for unilateral closes to be effective.

Given the variance in fees, and the fact that the transaction may be
spent in the future, it's a good idea for the fee payer to keep a good
margin (say 5x the expected fee requirement); but, due to differing methods of
fee estimation, an exact value is not specified.

Since the fees are currently one-sided (the party which requested the
channel creation always pays the fees for the commitment transaction),
it's simplest to only allow it to set fee levels; however, as the same
fee rate applies to HTLC transactions, the receiving node must also
care about the reasonableness of the fee.

## Message Retransmission

Because communication transports are unreliable, and may need to be
re-established from time to time, the design of the transport has been
explicitly separated from the protocol.

Nonetheless, it's assumed our transport is ordered and reliable.
Reconnection introduces doubt as to what has been received, so there are
explicit acknowledgments at that point.

This is fairly straightforward in the case of channel establishment
and close, where messages have an explicit order, but during normal
operation, acknowledgments of updates are delayed until the
`htlc_complete` / `revoke_and_ack` exchange; so it cannot be assumed
that the updates have been received. This also means that the receiving
node only needs to store updates upon receipt of `htlc_complete`.

Note that messages described in [BMW #7](07-routing-gossip.md) are
independent of particular channels; their transmission requirements
are covered there, and besides being transmitted after `init` (as all
messages are), they are independent of requirements here.

1. type: 136 (`channel_reestablish`)
2. data:
   * [`32`:`channel_id`]
   * [`8`:`next_local_commitment_number`]
   * [`8`:`next_remote_revocation_number`]
   * [`32`:`your_last_commitment_secret`] (option_data_loss_protect)
   * [`33`:`my_current_revokation_hash`] (option_data_loss_protect)

`next_local_commitment_number`: A commitment number is a 48-bit
incrementing counter for each commitment transaction; counters
are independent for each peer in the channel and start at 0.
They're only explicitly relayed to the other node in the case of
re-establishment, otherwise they are implicit.


### Requirements

A funding node:
  - upon disconnection:
    - if it has broadcast the funding transaction:
      - MUST remember the channel for reconnection.
    - otherwise:
      - SHOULD NOT remember the channel for reconnection.

A non-funding node:
  - upon disconnection:
    - if it has sent the `funding_signed` message:
      - MUST remember the channel for reconnection.
    - otherwise:
      - SHOULD NOT remember the channel for reconnection.

A node:
  - MUST handle continuation of a previous channel on a new encrypted transport.
  - upon disconnection:
    - MUST reverse any uncommitted updates sent by the other side (i.e. all
    messages beginning with `update_` for which no `htlc_complete` has
    been received).
      - Note: a node MAY have already used the `payment_preimage` value from
    the `update_fulfill_htlc`, so the effects of `update_fulfill_htlc` are not
    completely reversed.
  - upon reconnection:
    - if a channel is in an error state:
      - SHOULD retransmit the error packet and ignore any other packets for
      that channel.
    - otherwise:
      - MUST transmit `channel_reestablish` for each channel.
      - MUST wait to receive the other node's `channel_reestablish`
        message before sending any other messages for that channel.

The sending node:
  - MUST set `next_local_commitment_number` to the commitment number of the
  next `commitment_signed` it expects to receive.
  - MUST set `next_remote_revocation_number` to the commitment number of the
  next `revoke_and_ack` message it expects to receive.
  - if it supports `option_data_loss_protect`:
    - if `next_remote_revocation_number` equals 0:
      - MUST set `your_last_per_commitment_secret` to all zeroes
    - otherwise:
      - MUST set `your_last_per_commitment_secret` to the last `per_commitment_secret`
    it received

A node:
  - if `next_local_commitment_number` is 1 in both the `channel_reestablish` it
  sent and received:
    - MUST retransmit `funding_locked`.
  - otherwise:
    - MUST NOT retransmit `funding_locked`.
  - upon reconnection:
    - MUST ignore any redundant `funding_locked` it receives.
  - if `next_local_commitment_number` is equal to the commitment number of
  the last `commitment_signed` message the receiving node has sent:
    - MUST reuse the same commitment number for its next `commitment_signed`.
  - otherwise:
    - if `next_local_commitment_number` is not 1 greater than the
  commitment number of the last `commitment_signed` message the receiving
  node has sent:
      - SHOULD fail the channel.
  - if `next_remote_revocation_number` is equal to the commitment number of
  the last `revoke_and_ack` the receiving node sent, AND the receiving node
  hasn't already received a `closing_signed`:
    - MUST re-send the `revoke_and_ack`.
  - otherwise:
    - if `next_remote_revocation_number` is not equal to 1 greater than the
    commitment number of the last `revoke_and_ack` the receiving node has sent:
      - SHOULD fail the channel.
    - if it has not sent `revoke_and_ack`, AND `next_remote_revocation_number`
    is equal to 0:
      - SHOULD fail the channel.

 A receiving node:
  - if it supports `option_data_loss_protect`, AND the `option_data_loss_protect`
  fields are present:
    - if `next_remote_revocation_number` is greater than expected above, AND
    `your_last_per_commitment_secret` is correct for that
    `next_remote_revocation_number` minus 1:
      - MUST NOT broadcast its commitment transaction.
      - SHOULD fail the channel.
      - SHOULD store `my_current_per_commitment_point` to retrieve funds
        should the sending node broadcast its commitment transaction on-chain.
    - otherwise (`your_last_per_commitment_secret` or `my_current_per_commitment_point`
    do not match the expected values):
      - SHOULD fail the channel.

A node:
  - MUST NOT assume that previously-transmitted messages were lost,
    - if it has sent a previous `commitment_signed` message:
      - MUST handle the case where the corresponding commitment transaction is
      broadcast at any time by the other side,
        - Note: this is particularly important if the node does not simply
        retransmit the exact `update_` messages as previously sent.
  - upon reconnection:
    - if it has sent a previous `shutdown`: 
      - MUST retransmit `shutdown`.

### Rationale

The requirements above ensure that the opening phase is nearly
atomic: if it doesn't complete, it starts again. The only exception
is if the `funding_signed` message is sent but not received. In
this case, the funder will forget the channel, and presumably open
a new one upon reconnection; meanwhile, the other node will eventually forget
the original channel, due to never receiving `funding_locked` or seeing
the funding transaction on-chain.

There's no acknowledgment for `error`, so if a reconnect occurs it's
polite to retransmit before disconnecting again; however, it's not a MUST,
because there are also occasions where a node can simply forget the
channel altogether.

`closing_signed` also has no acknowledgment so must be retransmitted
upon reconnection (though negotiation restarts on reconnection, so it needs
not be an exact retransmission).
The only acknowledgment for `shutdown` is `closing_signed`, so one or the other
needs to be retransmitted.

The handling of updates is similarly atomic: if the commit is not
acknowledged (or wasn't sent) the updates are re-sent. However, it's not
insisted they be identical: they could be in a different order,
involve different fees, or even be missing HTLCs which are now too old
to be added. Requiring they be identical would effectively mean a
write to disk by the sender upon each transmission, whereas the scheme
here encourages a single persistent write to disk for each
`commitment_signed` sent or received.

A re-transmittal of `revoke_and_ack` should never be asked for after a
`closing_signed` has been received, since that would imply a shutdown has been
completed — which can only occur after the `revoke_and_ack` has been received
by the remote node.

Note that the `next_local_commitment_number` starts at 1, since
commitment number 0 is created during opening.
`next_remote_revocation_number` will be 0 until the
`commitment_signed` for commitment number 1 is received, at which
point the revocation for commitment number 0 is sent.

`funding_locked` is implicitly acknowledged by the start of normal
operation, which is known to have begun after a `commitment_signed` has been
received — hence, the test for a `next_local_commitment_number` greater
than 1.

A previous draft insisted that the funder "MUST remember ...if it has
broadcast the funding transaction, otherwise it MUST NOT": this was in
fact an impossible requirement. A node must either firstly commit to
disk and secondly broadcast the transaction or vice versa. The new
language reflects this reality: it's surely better to remember a
channel which hasn't been broadcast than to forget one which has!
Similarly, for the fundee's `funding_signed` message: it's better to
remember a channel that never opens (and times out) than to let the
funder open it while the fundee has forgotten it.

`option_data_loss_protect` was added to allow a node, which has somehow fallen behind
(e.g. has been restored from old backup), to detect that it's fallen-behind. A fallen-behind
node must know it cannot broadcast its current commitment transaction — which would lead to
total loss of funds — as the remote node can prove it knows the
revocation preimage. The error returned by the fallen-behind node
(or simply the invalid numbers in the `channel_reestablish` it has
sent) should make the other node drop its current commitment
transaction to the chain. This will, at least, allow the fallen-behind node to recover
non-HTLC funds, if the `my_current_per_commitment_point`
is valid. However, this also means the fallen-behind node has revealed this
fact (though not provably: it could be lying), and the other node could use this to
broadcast a previous state.

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
