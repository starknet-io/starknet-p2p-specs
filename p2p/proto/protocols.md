# Starknet Protocols

## Terminology
In each Starknet protocol, one peer sends a `Request` message and the other peer responds with
multiple `Response` messages. Each protocol has a single `Request` and `Response` protobuf message
types.

The peer that sends the request is called the `Outbound` peer and the peer that receives the request and sends responses is called the `Inbound` peer

The request message and responses messages to that request all belong to a single `Session`.
Note that there can be multiple sessions open simultaneously between the same peers and of the same
protocol.

## Protocols Briefing
The following table describes the different protocols in Starknet, the name that should be used in
negotiation, and the protobuf messages related to the protocol.
| Protocol | Name (for negotiation) | Request Message | Response Message |
| ------------ | -------------- | -------------- | -------------- |
| Headers | /starknet/headers/1 | [BlockHeadersRequest](./block.proto) | [BlockHeadersResponse](./block.proto) |
| Bodies | /starknet/bodies/1 | [BlockBodiesRequest](./block.proto) | [BlockBodiesResponse](./block.proto) |
| Transactions | /starknet/transactions/1 | [TransactionsRequest](./transaction.proto) | [TransactionsResponse](./transaction.proto) |
| Receipts | /starknet/receipts/1 | [ReceiptsRequest](./receipt.proto) | [ReceiptResponse](./receipt.proto) |
| Events | /starknet/events/1 | [EventsRequest](./event.proto) | [EventsResponse](./event.proto) |
| Kademlia (for discovery) | /starknet/kad/<chain_id>/<kademlia_version> |
| Identify (for discovery) | /starknet/id/<identify_version> |

## Overview
### Iteration
The `Request` message in each protocol contains an [Iteration](./common.proto). 

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
The [Fin](./common.proto) message signals the end of a protocol. After sending all the data,
the inbound peer sends a Fin message with no error. If for some reason the inbound peer can't send
all the data, then it sends all the data it can and then sends a `Fin` message with the appropriate
error.

#### Fin errors
- Busy: The node has too many open sessions and can't open a new session.
- Too Much: The node has determined that if it will send all the data requested it will surpass
the limits defined [here](#limits). TODO add description which limits
- Unknown: The node either does not know some of the blocks (It didn't see them yet), or did not
download them yet
- Pruned: The node has erased some of the blocks because they were too old.

### Limits
- Each message must be less than 1MB
- TBD limits on the amount of total messages/bytes in a single session
- TBD timeout for a session
- TBD timeout between two messages

## Protocols Breakdown
### Headers
The Headers protocol is used to download block headers and signatures.

It's name for negotiation is /starknet/headers/1

We assume that there will never be a header that's bigger than the [limit](#limits) for each message
and that all the signatures for each header will be smaller than this limit.

This means that a single message can represent a header or its signatures.

A response is a [BlockHeadersResponse](./block.proto) message which contains repeated
[BlockHeadersResponsePart](./block.proto) messages.
The inbound peer should always send all the BlockHeadersResponsePart packed in a single
BlockHeadersResponse unless they surpass 1MB.

Each BlockHeadersResponsePart is either a header, signatures or fin message. The ordering of the
parts should follow the following pattern:

`(BlockHeader(Signatures)?)*Fin`

Meaning that the inbound peer sends all headers and after each header they may send the signatures
of that header. After all the headers the inbound peer can send are sent, they need to send a Fin message.

The Fin message should be empty if all headers were sent. Otherwise, the Fin should contain the
[reason](#fin-errors) why not all headers were sent.

The inbound peer may omit a header's signatures if it pruned them. E.g for a reason to prune
signatures is when the relevant block is accepted on L1.


