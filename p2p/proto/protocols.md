# Starknet Protocols

## Terminology
`Request` - The message that is sent in order to request data from another node.

`Response` - The message that is sent in response to a `Request` message and contains the requested data.

`Outbound peer` - The peer that sent the `Request` message.

`Inbound peer` - The peer that received the `Request` message.

`Session` - The `Request` message and all the `Response` messages related to that request. This term 
is used to distinguish between `Response` messages from one `Request` to another.

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
In each Starknet protocol (except for protocols that exist outside of the scope of Starknet), one
peer sends a single request message and the other peer responds with multiple response messages

The request is always a range of blocks and the responses are data related to those blocks.

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
The [Fin](./common.proto) message signals the end of a protocol. After sending all the data,
the inbound peer sends a Fin message with no error. If for some reason the inbound peer can't send
all the data, then it sends all the data it can and then sends a `Fin` message with the appropriate
error.

#### Fin errors
- Busy: The node has too many open sessions and can't continue with this session.
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

It's name for negotiation is `/starknet/headers/1`

We assume that there will never be a header that's bigger than the [message size limit](#limits)
and that all the signatures for each header will be smaller than this limit.

This means that a single message can represent a header or its signatures.

A response is a [BlockHeadersResponse](./block.proto) message which contains repeated
[BlockHeadersResponsePart](./block.proto) messages.
The inbound peer decides how they want to pack parts into BlockHeadersResponse (as long as they're
not violating the [message size limit](#limits)), balancing the trade off between the overhead of
sending multiple messages and adding latency to the outbound peer when receiving the message.

Each BlockHeadersResponsePart is either a header, signatures or fin message. The ordering of the
parts should follow the following pattern:

`(BlockHeader(Signatures)?)*Fin`

Meaning that the inbound peer sends all headers and after each header they may send the signatures
of that header. After all the headers the inbound peer can send are sent, they need to send a Fin message.

The inbound peer may omit a header's signatures if it pruned them. E.g a reason to prune signatures
is when the relevant block is accepted on L1.

If all headers were sent, the Fin message at the end should have an empty error field. Otherwise,
the Fin should contain the [reason](#fin-errors) why not all headers were sent.

The headers should be sent ordered according to the request. If some of the headers are missing, the
inbound peer should send the headers they got according to the order (Even if it means they're
starting from a different block than requested).
