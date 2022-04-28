# RFC 019: 

## Changelog

- 19-Apr-2022: Initial draft (@williambanfield).

## Abstract

* TODO

## Background

### Current Block Structure

The current block structure is included here to aid discussion.

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
  // does this need to be here at all?
  string                       chain_id             = 2;


  int64                        height               = 3;



  google.protobuf.Timestamp    time                 = 4;


  BlockID                      last_block_id        = 5;


  bytes                        last_commit_hash     = 6;


  bytes                        data_hash            = 7;

  // Do both of these need to be included here?
  bytes                        validators_hash      = 8;
  bytes                        next_validators_hash = 9;
  // what is this, oh yeah the hash of the consensus params.
  // ugh.
  bytes                        consensus_hash       = 10;
  bytes                        app_hash             = 11;
  // What is the use of this?
  bytes                        last_results_hash    = 12;
  // Does this need to be kept separately?
  bytes                        evidence_hash        = 13;
  // This is not accurate, what if we changed to POL round.
  bytes                        proposer_address     = 14;
  // why not just have a 'fields hash' that's the hash of all of the
  // fields in the block?
}
```

```proto
message Data {
  repeated bytes txs = 1;
}
```

```proto
message Commit {
  // Why is this needed here at all? Gossip and storage maybe?
  int64              height     = 1;
  // can this instead be contained in the header? Not sure why this is here either
  // Ah, it contains information that will have been signed by the committers.
  int32              round      = 2;
  // Can definitely 
  BlockID            block_id   = 3;
  repeated CommitSig signatures = 4;
}
```

```proto
message CommitSig {
  BlockIDFlag               block_id_flag     = 1;
  // If the list is in order, I'm not sure this needs to be in the block
  // at all.
  bytes                     validator_address = 2;
  // remove
  google.protobuf.Timestamp timestamp         = 3;
  bytes                     signature         = 4;
}
```

```proto
message BlockID {
  bytes         hash            = 1;
  // remove
  PartSetHeader part_set_header = 2; // Q Do we ever use the PartSetHeader from the actual block header to rebuild the block or figure out what to grab?
}
```

## Consolidate CommitSig Information

The CommitSig contains multiple fields that are redundant.  The structure
contains the `validator_address` of the corresponding validator and the
timestamp corresponding to the timestamp in the precommit message. Both of these
data are not necessary and can be removed.

The `validator_address` does not need to be included in the block. The order
of the validator signatures is stored in the node and verification does not
rely on being able to ensure that the validator address matches the signature.

The `timestamp` field is no longer meaningful. This field was previously used
to create the block time when the BFTTime algorithm was in use. With the
completion of the proposer-based timestamps algorithm, these timestamp field
is no longer used at all and can be deleted from the block.

### What does this improve?

### Is this backwards compatible?

## Add Proof Of Lock Round Number To Header

The proof of lock round is the most recent round in which the Tendermint
algorithm observed a super majority of voting power on the network for a block.

Including this value in the block is allow validation of currently
un-validated metadata. Specifically, including this value will allow Tendermint
to be certain that the `proposer_address` is correct. Without knowing the round
number, Tendermint cannot calculate which validator was supposed to propose
the previous height. Because of this, our validation logic does not check that
the `proposer_address` included in the block corresponds to the validator that
proposed the round. Instead, it simply checks that the value is an address
of one of the known validators.

### What does this improve?

### Is this backwards compatible?

## Remove Chain ID

The `chain_id` is a string selected by the chain operators, usually a
human-readable name for the network. This value is immutable for the lifetime
of the chain and is defined in the genesis file. Because this value never
changes, there's no particular reason to maintain it in every block. Aesthetically,
it is somewhat pleasing, as if every block in a chain is stamped as part of
some particular chain. I'm pretty ambivalent on removing it from the structure
and would be in favor of keeping it despite its redundancy. 

HERE !!!

### What does this improve?

### Is this backwards compatible?

## Remove Validators Hash

## Remove Proposer Address

## Remove Validator Address

## Add Generic Signature Field

### What does this improve?

### Is this backwards compatible?

## Consolidate Block ID

* Remove the silly extra field for block parts. this is data that's useful for
* fetching the block using the tendermint p2p protocol but isn't very meaningful to
* the execution or validity of the block.

### What is the goal of this project?

Remove as much as possible from the block so that:

1. ABCI can still receive enough information to function.
2. Light clients can quickly prove the state of the blockchain. #REWORD
3. Nodes performing blocksync to catchup can restore state and verify that
consensus proceeded correctly using only the contents of the block.
4. Operators and developers can debug problems that arise with the state
machine using the contents of the block.

## Discussion

* TODO

## References

* TODO
