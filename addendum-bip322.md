# Addendum: The Bitcoin Jurat with BIP-322 Signed Messages

### Time attestations for Bitcoin signed messages and proofs of reserves using the BIP-322 standard

**Francis Pouliot**
francis@satoshiportal.com
Addendum to "Bitcoin Jurat: the decentralized notary" (published 2018-10-13)
Addendum date: 2026-07-14

---

## Motivation

The Bitcoin Jurat protocol was designed for generic messages, documents and contracts. It uses the Bitcoin blockchain to prove the time window during which a digital signature was created, and it can be applied to any data that can be signed. The original paper demonstrated the protocol using PGP cleartext signatures, and the choice of the signature algorithm was left as an implementation detail. The ownership of bitcoins was not a subject of the original protocol.

In this addendum, we apply the Bitcoin Jurat protocol to a new use case, which is the proof of ownership of Bitcoin addresses and of Bitcoin coins.

There are many situations in which the owner of a Bitcoin wallet wants to prove that he owns a Bitcoin address, for example as a security measure before receiving funds to that address, or to prove that he controls a certain amount of coins, a process commonly referred to as a proof of reserves. These proofs are themselves digital signatures, created with the private keys of a Bitcoin wallet. Since May 2026, there exists a complete standard for creating such signatures with any type of Bitcoin address: the BIP-322 Generic Signed Message Format[^1]. Using BIP-322, the owner of a Bitcoin address signs a message by constructing a pair of virtual Bitcoin transactions which are valid according to the consensus rules, but which can never be included in the blockchain. The standard also specifies a proof of funds format, in which the signer proves control of specific unspent transaction outputs by signing them as if they were being spent, inside the same virtual transaction that can never be broadcast.

Because a BIP-322 signature is a digital signature like any other, it contains no information about the time at which it was created. However, when the object of the signature is the ownership of an address or of coins, the time of creation is the most important information: a counterparty receiving a proof of ownership wants to know that the address or the coins are controlled by the signer at the present moment, and an auditor receiving a proof of reserves wants to know that the coins were under the control of the signer at a specific date. By applying the Bitcoin Jurat protocol to BIP-322 signatures, we can prove the time window during which a proof of address ownership or a proof of reserves was created, using only the Bitcoin blockchain.

---

## The Bitcoin Jurat using BIP-322

The process is the same four-step process described in the original paper, and only the signature algorithm changes.

### Proof-of-Absence

The Bitcoin block hash marker is included at the beginning of the message that is to be signed. We suggest a simple format that software can easily parse:

```
jurat1:<block height>:<block hash in lowercase hexadecimal>
<the message itself, on the following lines>
```

In BIP-322, the message is the only data chosen by the signer that is covered by the signature, because the hash of the message is committed inside the virtual transactions which the signature signs. Data attached to the signing process in any other way, for example in the fields of a PSBT file, is not covered by the signature and therefore cannot be used as a proof-of-absence marker.

This construction is entirely retrocompatible with BIP-322, because a Bitcoin Jurat is itself a valid BIP-322 signature: it does not introduce a new signature format and it does not modify the virtual transactions. Verification software that is not aware of the Bitcoin Jurat protocol will verify the signature normally and simply ignore the block hash marker, which appears to it as an ordinary part of the message. Verification software that is aware of the Bitcoin Jurat protocol reads the marker and obtains the proof-of-absence. Consequently, no coordination with existing BIP-322 implementations is required, and no permission is needed to use the protocol: any wallet that can sign a BIP-322 message can create a Bitcoin Jurat today.

For the BIP-322 formats which serialize the complete virtual transaction (the "full" and "proof of funds" formats), the signer can additionally write the block height of the marker into the lock time field of the virtual transaction, which is also covered by the signature. The BIP-322 specification instructs validators to tolerate lock time values without enforcing them, so this addition does not affect verification by software that is unaware of the protocol, while software that is aware of it can check that the lock time matches the marker.

### Proof-of-Existence

This step is unchanged from the original paper. The BIP-322 signature is hashed using SHA256, and the hash is timestamped in the Bitcoin blockchain using the OpenTimestamps protocol[^2]. Because the signature binds the message, and the message contains the block hash marker, timestamping the signature alone is sufficient to complete the jurat.

---

## Proof of Reserves with a Provable Creation Time

The BIP-322 proof of funds format signs a set of unspent transaction outputs as additional inputs of the virtual transaction, with the witness data filled in exactly as if the coins were being spent. The resulting signature proves control of those specific coins, bound to a message, in a transaction that can never move them.

A signature by itself proves that the private keys were controlled at some moment, but it is impossible to know at which moment. There is nothing in a signature that distinguishes a signature created a minute ago from a signature created a year ago and kept in storage. When a verifier looks up the claimed outputs in the Bitcoin UTXO set, he learns that the coins have not moved, but he does not learn that the presenting party still knows the private keys, and he does not learn when control of the coins was actually exercised. A date written inside the message is only a claim made by the signer, and the verifier has no way to validate it.

Using the Bitcoin Jurat, the verifier can prove both of these facts himself:

1. The block hash marker proves that the signature did not exist before the marker block. If the marker block is one hour old, then control of the claimed coins was exercised within the last hour. A counterparty who demands proof of present control can therefore require a recent marker, for example a marker no older than 144 blocks, and verify this requirement himself using the Bitcoin block headers instead of trusting the signer.
2. The OpenTimestamps attestation proves that the signature existed before the attestation block. The claim that certain reserves were under the control of the signer at a certain date can therefore be demonstrated to any auditor, years later, without trusting the date written in the message or the word of the presenter.

In addition to these two proofs, the verifier independently consults the Bitcoin UTXO set to confirm that the claimed outputs are still unspent at the time of verification. This is a statement about the present, and it must be kept separate from the two proofs above.

The privacy considerations of the original paper apply here as well, with one addition: a proof of funds permanently links the disclosed outputs to one another and to the party presenting the proof, and with a jurat this linkage carries a provable date. The OpenTimestamps step itself does not disclose any information, since the calendar servers only ever receive a SHA256 hash.

---

## Verification Process

Verification requires BIP-322 signature verification software, access to the Bitcoin block headers, and an OpenTimestamps client. The verifier is given the address, the message, the signature, and the OpenTimestamps file.

1. **Signature verification**: Using BIP-322 verification software, verify the signature over the exact bytes of the message. This step is identical whether or not the verifier is aware of the Bitcoin Jurat protocol.

2. **Bitcoin block hash marker verification**: Read the block height and the block hash from the first line of the message. Using a Bitcoin full node, confirm that the block at this height has this hash. The timestamp of this block establishes the earliest time at which the signature can have been created.

3. **Signature hash verification**: Using any SHA256 hash function calculator, calculate the hash of the signature.

4. **Timestamp verification**: Using the OpenTimestamps client connected to a Bitcoin full node, verify that the hash of the signature was committed to the Bitcoin blockchain, and confirm the block in which the commitment was included. The timestamp of this block establishes the latest time at which the signature can have been created.

If the proof is a proof of funds, the verifier additionally extracts the claimed outputs from the signature and looks them up in the Bitcoin UTXO set.

The verifier has thus proven that the signature of this message, created by the owner of this address and of these coins, can only have been created during the time window between the marker block and the attestation block, using only the Bitcoin blockchain.

[^1]: https://github.com/bitcoin/bips/blob/master/bip-0322.mediawiki
[^2]: https://petertodd.org/2016/opentimestamps-announcement
