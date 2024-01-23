# Starknet Protocols

## Terminology
`Request` - The message that is sent in order to request data from another node.

`Response` - The message that is sent in response to a `Request` message and contains the requested data.

`Querying peer` - The peer that sent the `Request` message.

`Replying peer` - The peer that received the `Request` message.

`Session` - The `Request` message and all the `Response` messages related to that request. This term 
is used to distinguish between `Response` messages from one `Request` to another.

## Protocols Briefing
The following table describes the different protocols in Starknet, the name that should be used in
negotiation, and the protobuf messages related to the protocol.
| Protocol | Name (for negotiation) | Request Message | Response Message |
| ------------ | -------------- | -------------- | -------------- |
| Headers | /starknet/headers/1 | [BlockHeadersRequest](./header.proto) | [BlockHeadersResponse](./header.proto) |
| StateDiffs | /starknet/state_diffs/1 | [StateDiffsRequest](./state.proto) | [StateDiffsResponse](./state.proto) |
| Classes | /starknet/classes/1 | [ClassesRequest](./class.proto) | [ClassesResponse](./class.proto) |
| Transactions | /starknet/transactions/1 | [TransactionsRequest](./transaction.proto) | [TransactionsResponse](./transaction.proto) |
| Receipts | /starknet/receipts/1 | [ReceiptsRequest](./receipt.proto) | [ReceiptResponse](./receipt.proto) |
| Events | /starknet/events/1 | [EventsRequest](./event.proto) | [EventsResponse](./event.proto) |
| Kademlia (for discovery) | /starknet/kad/<chain_id>/<kademlia_version> |
| Identify (for discovery) | /starknet/id/<identify_version> |

## Overview
In each Starknet protocol (except for protocols that exist outside of the scope of Starknet), one
peer sends a single request message and the other peer responds with multiple response messages

The request is always a range of blocks and the responses are data related to those blocks.

This allows the querying peer to process data (e.g calculate hashes) before it downloaded all the
data that the replying peer sent in the session.

The data sent is always a `oneof` of messages containing data and a [Fin](#fin).

### Partitioning data into blocks
Except for the [headers](#headers) protocol, The partition of data into blocks does not appear in
the data itself.

This partition can be done based on the data in the header.

For each protocol, in the header, there's a commitment on the data given by this protocol.
The commitment is made of hash and number of elements

The commitment is included inside the block hash. Meaning that it can be validated once we
downloaded the header.

The length field can be used to determine in each protocol when does the data of one block stops and
the data of the next block after it begins.

### Length Prefix
Each response message is prefixed with the amount of bytes that the encoded message contains.

The length is written as a [varint](https://protobuf.dev/programming-guides/encoding/#varints)

The request message is not prefixed with length. The querying peer should close its writer end once
it sends the request message (it's the only message it should send in the session). The replying peer
should read bytes until it reaches EOF and thus it doesn't need to know the length before reading
the message

### Limits
- Each message must be less than 1MB, except for the [Class](./class.proto) message which is allowed
to be up to 4MB.
- TBD limits on the amount of total messages/bytes in a single session
- TBD timeout for a session
- TBD timeout between two messages

### Optional fields
In `proto3`, each field is optional, and the `optional` keyword does nothing.

We want to mark fields as required, so every field that is not marked as `optional` is considered
required.

If a peer sends a message with a missing required field, they are considered malicious.

### Iteration
The request message in each protocol contains an [Iteration](./common.proto). 

The Iteration message defines an ordered range of blocks.

The Iteration message has the following fields:
- **start**: The first block that we request, by hash or by number
- **direction**: Decides whether the blocks we send after the starting block are the blocks after it
or the blocks before it
- **limit**: The amount of blocks requested
- **step**: Given a step `i`, the responder will return every i'th block it sees until it reached `limit` blocks.

For example. If the request has the values:
- **start**: 10
- **direction**: Backward
- **limit**: 3
- **step**: 3

Then the blocks we'll receive will be blocks no. 10, 7, 4

### Fin
The [Fin](./common.proto) message is an empty message that signals the end of a protocol.
After sending all the data, the replying peer sends a Fin message.
If a replying peer sends any additional messages after sending a fin message, they are considered
malicious and the connection with them should be dropped.

### Missing Data
We currently assume that if a peer has some of the data for a block then it has all the data for
that block.

If the replying peer doesn't have all the data for all the blocks it was requested, it should send
all the data for the blocks it has according to the order of the iteration and stop and send Fin
when it encountered the first missing block (even if it's the first block).

## Protocols Breakdown
### Headers
The Headers protocol is used to download block headers and signatures.

Its name for negotiation is `/starknet/headers/1`

Each single message is a fin or a [SignedBlockHeader](./header.proto)

A header is comprised of:
* block hash
* hash of the parent block
* block number
* metadata for the block (timestamp, sequencer address, protocol version, gas prices)
* Commitments on the data of each of the other protocols. Each commitment is comprised of length and
hash. The commitment for each protocol will be described in that protocol's description.

There's a single message for each block. That's because we assume that there will never be a header
that's bigger than the [message size limit](#limits) including all signatures for this header.

### Transactions
The transactions protocol is used to download the transactions in a range of blocks.

Its name for negotiation is `/starknet/transactions/1`

Each single message is a fin or a [Transaction](./transaction.proto)

Each transaction represents a Starknet transaction. For more detail on the different transaction
types, their content and their hash calculation see [here](https://docs.starknet.io/documentation/architecture_and_concepts/Network_Architecture/transactions/).

The commitment of the transactions in the header is comprised of the number of transactions and
their merkle root.

In order to verify the transactions, the node needs to calculate their hash and compare it to the
transactions merkle root in the block header.

The number can be used to delimiter between the transactions of each block. 

In order for the querying peer to know that the replying peer is malicious as soon as possible, they
should do two things:

1. Calculate the transaction hash once each transaction arrives and validate it's the same hash.
Even though we can only verify that the hash appears in the commitment after we got all the
transactions for the block, this check is a proof of work. It checks that the replying peer did some
work at some point of time to calculate the hash of this transaction and delays a peer that sends
transactions with random fields
2. Reject the connection and declare the peer malicious once it sent more transactions than the sum
of transaction amounts in the headers of the requested blocks.

### Events
The Events protocol is used to download the events emitted in a range of blocks.

Its name for negotiation is `/starknet/events/1`

Each single message is a fin or an [Event](./event.proto)

Each event contains
* The index of the transaction that emitted this event
(If it's the first transaction in the block, the index will be 0).
* The address of the contract that emitted this event.
* The keys and data of this event.

The commitment of the events in the header is comprised of the number of events and
their merkle root.

In order to verify the events, the node needs to calculate their hash and compare it to the
events merkle root in the block header.

The number can be used to delimiter between the events of each block, and the transaction index field
of each event is used to delimiter between the events of each transaction within a block.

**Important Note**: The transaction index is currently not part of the headers hash. Until this is
fixed, the value of the `transaction_index` field in each event is trusted.

In order for the querying peer to know that the replying peer is malicious as soon as possible, they
should do the same two things recommended in the [transactions](#transactions) protocol.

### Receipts
The Receipts protocol is used to download the receipts of transactions in a range of blocks.

Its name for negotiation is `/starknet/events/1`

Each single message is a fin or a [Receipt](./receipt.proto)

There is a single receipt per transaction.

This means that the number of transactions inside the header can be used to delimiter between the
receipts of each block.

#### Important Note
The receipts commitment in the header is currently not part of the block hash.
Until this is fixed, everything in the receipts protocol except for the amount of receipts is trusted.

It is important not to validate the commitment of receipts until this is fixed because then the node
can get stuck if the peer that gave the header is malicious and the peer that gives receipts is honest.

### State Diff
The state diff protocol is used to download the state diffs created by a range of blocks.
It can be used to avoid running the transactions for computing the state diff.

Its name for negotiation is `/starknet/state_diffs/1`

In order to verify the state diff, the node needs to calculate its hash and compare it to the
`state_diff_commitment` field in the block header. The structure of the hash is detailed [here](https://community.starknet.io/t/introducing-p2p-authentication-and-mismatch-resolution-in-v0-12-2/97993)

#### Message structure
Each single message is either a fin, a [ContractDiff](./state.proto) or a [DeclaredClass](./state.proto).

A ContractDiff represents all the changes made for a given contract. This includes:
* Storage diffs - Represented by the `values` field.
* Nonce updates - Represented by the `nonce` field if present.
* Deployed contracts - Represented by the `class_hash` field if present. If it's present it means
that this contract was deployed or replaced in this block.

A DeclaredClass represents a declared class. If the class is Cairo1 then the `compiled_class_hash`
field will be present

There's no guarantee on what ordering will the messages have. The only constraints are:
1. If the same contract has multiple `ContractDiff`s, it's because they don't fit the
[limit](#limits) for a single message.
2. If that happens, then all `ContractDiff`s for the same contract need to appear continuously.

#### Length
As we've seen in other protocols like [Transactions](#transactions), The node needs to know the
amount of data to expect in order to reject malicious peers trying to DDoS the node.

In order to do that, the node needs to look at the block header at the field `state_diff_length`.
This field is equal to
`num_storage_diffs + num_nonce_updates + num_deployed_contracts + num_declared_classes`

Then, whenever the node receives a [StateDiffResponse](./state.proto), it needs to add a number to
a counter and declare the peer malicious if the counter becomes bigger than `state_diff_length`:
* For DeclaredClass, the counter should increase by 1.
* For ContractDiff, the counter should increase by the length of the field `values`,
and by 1 if the field `class_hash` is present, and by 1 if the field `nonce` is present.

##### Important Note
Right now, the `state_diff_length` is not part of the block hash. Until this is fixed, it's value is
trusted.

In order for the node to not be stuck if the peer that gave it the header is malicious, the node
should not validate the amount of state diff parts it got for a block and trust that the replying
peer won't try to DDoS it (and rely on the [limits](#limits) for DDoS prevention).

Aside from DDoS there's no possible attack that can happen. After the replying peer sent all the data,
the querying peer can verify it and see if it corresponds to the `state_diff_commitment` that the consensus signed on.


### Classes
The classes protocol is used to download the classes declared in a range of blocks.

Its name for negotiation is `/starknet/classes/1`

Each single message is a fin or a [Class](./class.proto). For more information on classes, see [here](https://docs.starknet.io/documentation/architecture_and_concepts/Smart_Contracts/contract-classes/)

You should use the [state diff protocol](#state-diff) before using this protocol.
The reason is that the class hashes and amount of declared classes per block are part of the
`state_diff_commitment` in the header, and these values are needed to validate the results of the
classes protocol and to delimiter between the classes of each block.
