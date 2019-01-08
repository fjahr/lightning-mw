# BMW #3: Transaction Formats

This details the exact format of on-chain transactions, which both sides need
to agree on to ensure signatures are valid. This consists of the funding
transaction, the commitment transactions, and the HTLC transactions.

# Table of Contents

  * [Transactions](#transactions)
    * [Commitment Transaction](#commitment-transaction)
        * [Commitment Transaction Outputs](#commitment-transaction-outputs)
          * [`to_local` Output](#to_local-output)
          * [`to_remote` Output](#to_remote-output)
          * [Offered HTLC Outputs](#offered-htlc-outputs)
          * [Received HTLC Outputs](#received-htlc-outputs)
        * [Trimmed Outputs](#trimmed-outputs)
    * [HTLC-timeout and HTLC-success Transactions](#htlc-timeout-and-htlc-success-transactions)
	* [Closing Transaction](#closing-transaction)
    * [Fees](#fees)
        * [Fee Calculation](#fee-calculation)
        * [Fee Payment](#fee-payment)
  * [Keys](#keys)
    * [Key Derivation](#key-derivation)
        * [`localpubkey`, `remotepubkey`, `local_htlcpubkey`, `remote_htlcpubkey`, `local_delayedpubkey`, and `remote_delayedpubkey` Derivation](#localpubkey-remotepubkey-local_htlcpubkey-remote_htlcpubkey-local_delayedpubkey-and-remote_delayedpubkey-derivation)
        * [`revocationpubkey` Derivation](#revocationpubkey-derivation)
        * [Per-commitment Secret Requirements](#per-commitment-secret-requirements)
    * [Efficient Per-commitment Secret Storage](#efficient-per-commitment-secret-storage)
  * [References](#references)
  * [Authors](#authors)

# Transactions

## Funding Transaction

The funding transaction is a 2-of-2 multisig transaction that is constructed in
cooperation between the funder and the fundee (see BMW #2) and then commited
to the blockchain.

## Commitment Transaction

In MimbleWimble the commitment transaction is actually a set of transactions
that work together to achieve the same results as Bitcoin Lightning Network
commitment transactions without the use of scripts.

### Commitment Transaction Outputs

To allow an opportunity for penalty transactions, in case of a revoked
commitment transaction, all outputs that return funds to the owner of the
commitment transaction (a.k.a. the "local node") must be delayed for
`to_self_delay` blocks. This delay is done in a second-stage HTLC transaction
(HTLC-success for HTLCs accepted by the local node, HTLC-timeout for HTLCs
offered by the local node).

The reason for the separate transaction stage for HTLC outputs is so that HTLCs
can timeout or be fulfilled even though they are within the `to_self_delay` delay.
Otherwise, the required minimum timeout on HTLCs is lengthened by this delay,
causing longer timeouts for HTLCs traversing the network.

The amounts for each output MUST be rounded down to whole satoshis. If this
amount, minus the fees for the HTLC transaction, is less than the
`dust_limit_satoshis` set by the owner of the commitment transaction, the output
MUST NOT be produced (thus the funds add to fees).

#### `to_local` Output

This output it another 2-of-2 multisig output between the funder and the
fundee and there are two transactions that could possibly spend this output
when the `to_local` Output ends up on the blockchain:

1. The transaction that sending the funds to the local participant. This
transaction is timelocked to a certain maturity/sequence of the outputs it is
spending and only allows the local participant to redeem after the
`to_self_delay` period has passed.
2. The penalty transaction that allows the remote participant to take
this output as well if the local participant tries to cheat by broadcasting
an old state. This transaction can only be used with a revokation key
(a hashlock).

#### `to_remote` Output

This output sends funds to the other peer using the blinding factor that
the peer has added himself in the process.

#### HTLC Outputs

An offered HTLC outputs consists of three connected transactions: 1.
An adjusted commitment transaction that adds another 2-of-2 multisig
that spends part of the amount of the participant who is offering
the HTLC. The rest of the output to this participant is adjusted
accordingly. The 2-of-2 output is the basis for the following two
alternative transactions. 2. The HTLC timeout transaction allows the
participant who offered the HTLC to take the funds back after a
certain period. Until the period is expired the transaction can
not be submitted to the blockchain. 3. The HTLC Success transaction
is the actual hashlocked transaction that allows the participant
who got offered the HTLC to take the output of the 2-of-2 multisig
if she can present the hash-preimage before it has expired.

### Trimmed Outputs

Each peer specifies a `dust_limit` below which outputs should
not be produced; these outputs that are not produced are termed "trimmed". A
trimmed output is considered too small to be worth creating and is instead added
to the commitment transaction fee. For HTLCs, it needs to be taken into
account that the second-stage HTLC transaction may also be below the
limit.

#### Requirements

The base fee:
  - before the commitment transaction outputs are determined:
    - MUST be subtracted from the `to_local` or `to_remote`
    outputs, as specified in [Fee Calculation](#fee-calculation).

The commitment transaction:
  - if the amount of the commitment transaction `to_local` output would be
less than `dust_limit` set by the transaction owner:
    - MUST NOT contain that output.
  - otherwise:
    - MUST be generated as specified in [`to_local` Output](#to_local-output).
  - if the amount of the commitment transaction `to_remote` output would be
less than `dust_limit` set by the transaction owner:
    - MUST NOT contain that output.
  - otherwise:
    - MUST be generated as specified in [`to_remote` Output](#to_remote-output).
  - for every offered HTLC:
    - if the HTLC amount minus the HTLC-timeout fee would be less than
    `dust_limit` set by the transaction owner:
      - MUST NOT contain that output.
    - otherwise:
      - MUST be generated as specified in
      [HTLC Outputs](#htlc-outputs).
  - for every received HTLC:
    - if the HTLC amount minus the HTLC-success fee would be less than
    `dust_limit` set by the transaction owner:
      - MUST NOT contain that output.
    - otherwise:
      - MUST be generated as specified in
      [HTLC Outputs](#htlc-outputs).


## Fees

### Fee Calculation

The fee calculation for both commitment transactions and HTLC
transactions is based on the current `feerate_per_kw` and the
*expected weight* of the transaction.

### Fee Payment

Base commitment transaction fees are extracted from the funder's amount;
if that amount is insufficient, the entire amount of the funder's output is used.

Note that after the fee amount is subtracted from the to-funder output,
that output may be below `dust_limit`, and thus will also
contribute to fees.

A node:
  - if the resulting fee rate is too low:
    - MAY fail the channel.

## Commitment Transaction Construction

This section ties the previous sections together to detail the
algorithm for constructing the commitment transaction for one peer:
given that peer's `dust_limit_satoshis`, the current `feerate_per_kw`,
the amounts due to each peer (`to_local` and `to_remote`), and all
committed HTLCs:

1. Initialize the commitment transaction input and locktime, as specified
   in [Commitment Transaction](#commitment-transaction).
1. Calculate which committed HTLCs need to be trimmed (see [Trimmed Outputs](#trimmed-outputs)).
2. Calculate the base [commitment transaction fee](#fee-calculation).
3. Subtract this base fee from the funder (either `to_local` or `to_remote`),
   with a floor of 0 (see [Fee Payment](#fee-payment)).
3. For every offered HTLC, if it is not trimmed, add an
   [HTLC output](#offered-htlc-outputs).
4. For every received HTLC, if it is not trimmed, add an
   [HTLC output](#received-htlc-outputs).
5. If the `to_local` amount is greater or equal to `dust_limit`,
   add a [`to_local` output](#to_local-output).
6. If the `to_remote` amount is greater or equal to `dust_limit`,
   add a [`to_remote` output](#to_remote-output).
7. Sort the outputs into [BIP 69 order](#transaction-input-and-output-ordering).

# References

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).

