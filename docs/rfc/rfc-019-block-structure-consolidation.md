# RFC 019: 

## Changelog

- 19-Apr-2022: Initial draft (@williambanfield).

## Abstract

* TODO

## Background

The current block structure contains multiple fields that are not required for
validation or execution of a Tendermint block. Some of these fields had vestigial
purposes that they no longer serve and some of these fields exist as a result of
internal Tendermint domain objects leaking out into the external data structure.

In so far as is possible, we should consolidate and prune these superfluous
fields before releasing a 1.0 version of Tendermint. All pruning of these
fields should be done with the aim of simplifying the structures to what
is needed while preserving information that aids with debugging and that also
allow external protocols to function more efficiently than if they were removed.

### Current Block Structure

The current block structures are included here to aid discussion.

```proto
message Block {
  Header                        header      = 1;
  Data                          data        = 2;
  tendermint.types.EvidenceList evidence    = 3;
  Commit                        last_commit = 4;
}
```

```proto
message Header {
  tendermint.version.Consensus version              = 1;
  string                       chain_id             = 2;
  int64                        height               = 3;
  google.protobuf.Timestamp    time                 = 4;
  BlockID                      last_block_id        = 5;
  bytes                        last_commit_hash     = 6;
  bytes                        data_hash            = 7;
  bytes                        validators_hash      = 8;
  bytes                        next_validators_hash = 9;
  bytes                        consensus_hash       = 10;
  bytes                        app_hash             = 11;
  bytes                        last_results_hash    = 12;
  bytes                        evidence_hash        = 13;
  bytes                        proposer_address     = 14;
}

```

```proto
message Data {
  repeated bytes txs = 1;
}
```

```proto
message EvidenceList {
  repeated Evidence evidence = 1;
}
```

```proto
message Commit {
  int64              height     = 1;
  int32              round      = 2;
  BlockID            block_id   = 3;
  repeated CommitSig signatures = 4;
}
```

```proto
message CommitSig {
  BlockIDFlag               block_id_flag     = 1;
  bytes                     validator_address = 2;
  google.protobuf.Timestamp timestamp         = 3;
  bytes                     signature         = 4;
}
```

```proto
message BlockID {
  bytes         hash            = 1;
  PartSetHeader part_set_header = 2;
}
```

### On Tendermint Blocks

#### What is a Tendermint 'Block'?

A block is the structure produced as the result of an instance of the Tendermint
consensus algorithm. At its simplest, the 'block' can be represented as a Merkle
root hash of all of the data used to construct and produce the hash. Our current
block proto structure includes _far_ from all of the data used to produce the
hashes included in the block.

It does not contain the full `AppState`, it does not contain the `ConsensusParams`,
nor the `LastResults`, nor the `ValidatorSet`. Additionally, the structure of
the block message is not inherently tied to this Merkle root hash. Different
structures of the same set of data could trivially be used to construct the
exact same hash. The thing we currently call the 'Block' is really just a view
into a subset of the data used to construct the root hash. Large amounts of it
can be removed or added to the 'Block' as long as alternative methods exist to
query and retrieve the constituent values.

#### Why this digression?

This digression is aimed at informing what it means to consolidate 'fields' in the
'block'. The discussion of what should be included in the block can be teased
apart into a few different lines of inquiry.

1. What values need to be included as part of the Merkle tree so that the
consensus algorithm can use proof-of-stake consensus to validate all of the
properties of the chain that we would like?
1. How can we create views of the data that can be easily retrieved, stored, and
verified by the relevant protocols.

These two concerns are intertwined at the moment as a result
of how we store and propagate our data but they don't necessarily need to be.
This document focuses primarily on the first concern by suggesting fields that
can be completely removed without any loss in the function of our consensus
algorithm.

This document also suggests ways that we may update our storage and propagation
mechanisms to better take advantage of Merkle tree nature of our data.

## Discussion 

### Data to completely remove

This section proposes a list of data that could be completely removed from the 
Merkle tree with no loss to the functionality of our consensus algorithm.

#### CommitSig.timestamp

This field was once used to 

### Updates to propagation

### Updates to views

This section proposes a list of changes to the current block structure accompanied
by discussion of the merits of the changes. Where the change is possible but
would hamper external protocols or make debugging more difficult, that is noted
in discussion.

### Consolidate CommitSig Information

The CommitSig contains multiple fields that are redundant. The structure
contains the `validator_address` of the validator whose signature it contains
and the timestamp corresponding to the timestamp in the precommit message. Both
of these data are not strictly necessary and are candidates for removal.

The `validator_address` does not need to be included in the block. The order
of the validator signatures is stored in the node and verification does not
rely on being able to ensure that the validator address matches the signature.
This field _is_ used by the light client. The light client may not have the
exact validator set that matches a particular block but can use the addresses
to check if it trusts _enough_ of the validators to verify the block. Removing
the addresses from the block would force the light client to redownload the
validator set each time it encounters a block with a `validators_hash` that the
client had not previously encountered. In an experimental [run][chain-experiment]
on the Cosmos Hub, this appears to occur frequently. A program counting the
validator set changes from height 5200791 to 5210791 counted 4297 changes in the
validator hash. That's nearly half of the blocks in the range. 

This change appears

The `timestamp` field is no longer meaningful. This field was previously used
to create the block time when the BFTTime algorithm was in use. With the
implementation of the proposer-based timestamps algorithm, this timestamp field
is no longer used at all and can be deleted from the block.

## Add Proof Of Lock Round Number To Header

The *proof of lock round* is the round of consensus for a height in which the
Tendermint algorithm observed a super majority of voting power on the network for
a block.

Including this value in the block will allow validation of currently
un-validated metadata. Specifically, including this value will allow Tendermint
to validate that the `proposer_address` in the block is correct. Without knowing
the locked round number, Tendermint cannot calculate which validator was supposed
to propose a height. Because of this, our validation logic does not check that
the `proposer_address` included in the block corresponds to the validator that
proposed the height. Instead, the validation logic simply checks that the value
is an address of one of the known validators.

Currently, we include this value in the `Commit` for height `H` which is
written into the block at height `H+1`. Tendermint can add this value directly
into the block instead of delaying it until the next block.

## Remove Chain ID

The `chain_id` is a string selected by the chain operators, usually a
human-readable name for the network. This value is immutable for the lifetime
of the chain and is defined in the genesis file. Because this value never
changes, there's no particular reason to maintain it in every block. Aesthetically,
it is somewhat pleasing, as if every block in a chain is stamped as part of
some particular chain. 

I'm pretty ambivalent on removing it from the structure
and would be in favor of keeping it despite its redundancy. Additionally, if
we are worried about the amount of space used by storing this with every block,
we could easily store the value separately from the block structure in the
database, although on-disk compression should be able to handle de-duplication
of that data without any additional code.

## Remove Validators Hash

Both `validators_hash` and `next_validators_hash` are included in the block
header. However, there is no strong reason to maintain both of these. The
`next_validators_hash` contains the same information as is included in the
previous block's `validators_hash`. Any participant that wants to validate the
current list of validator votes could do so by fetching the previous block's
header. # How does this impact the light client?

## Remove Proposer Address

With the addition of `round` to the header, there is no _strong_ reason to keep
the `proposer_address` field. This value can be calculated by repeatedly
running the [Proposer Selection Algorithm][proposer-selection] `round` times to determine which
validator proposed the block.

## Transition to a SignatureSet field form CommitSig

The `CommitSig` structure currently holds the signature of one validator. Each
`CommitSig` holds the validator address, the signature, the timestamp and a
flag, `BlockIDFlag`, indicating if the corresponding validator voted for the
block, for nil, or abstained from voting.

All of the relevant information for the block could be represented by a list of
sets containing a bit arrays and a list of signatures. An example of this 
structure is listed below.

```proto
message SignatureSet {
  Type                          type       = 1;
  tendermint.libs.bits.BitArray signers    = 2;
  repeated bytes                signatures = 3;

  enum Type {
    UNKNOWN    = 0;
    MULTI      = 1;
    AGGREGATED = 2;
  }
}
```

The `Commit` can be updated to include a list of `SignatureSet`s instead
of list of `CommitSig`s. This would shrink the space needed for each signature,
albeit mildly, while preparing Tendermint to adopt additional signing
mechanisms in the future.

## Consolidate Block ID

The [BlockID][block-id] field comprises the [PartSetHeader][part-set-header] and the hash of the block.
The `PartSetHeader` is used by nodes to gossip the block by dividing it into
parts. Nodes receive the `PartSetHeader` from their peers, informing them of
what pieces of the block to gather. There is no strong reason to include this
value in the block. Validators will still be able to gossip and validate the
blocks that they received from their peers using this mechanism even if it is
not written into the block. The `BlockID` can therefore be consolidated into
just the hash of the block. This is by far the most uncontroversial change
and there appears to be no good reason _not_ to do it. Further evidence that
the field is not meaningful can be found in the fact that the field is not
actually validated to ensure it is correct during block validation. Validation
only checks that the [field is well formed][psh-check].

## References

[light-verify-trusting]: https://github.com/tendermint/tendermint/blob/208a15dadf01e4e493c187d8c04a55a61758c3cc/types/validation.go#L124
[part-set-header]: https://github.com/tendermint/tendermint/blob/208a15dadf01e4e493c187d8c04a55a61758c3cc/types/part_set.go#L94
[block-id]: https://github.com/tendermint/tendermint/blob/208a15dadf01e4e493c187d8c04a55a61758c3cc/types/block.go#L1090
[psh-check]: https://github.com/tendermint/tendermint/blob/208a15dadf01e4e493c187d8c04a55a61758c3cc/types/part_set.go#L116
[proposer-selection]: https://github.com/tendermint/tendermint/blob/208a15dadf01e4e493c187d8c04a55a61758c3cc/spec/consensus/proposer-selection.md
[chain-experiment]: https://github.com/williambanfield/tmtools/blob/master/hash-changes/RUN.txt
