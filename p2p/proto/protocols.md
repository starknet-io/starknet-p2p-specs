## Protocols Breakdown
| Protocol | Name (for negotiation) | Request Message | Response Message |
| ------------ | -------------- | -------------- | -------------- |
| Headers | /starknet/headers/1 | [BlockHeadersRequest](./block.proto) | [BlockHeadersResponse](./block.proto) |
| Bodies | /starknet/bodies/1 | [BlockBodiesRequest](./block.proto) | [BlockBodiesResponse](./block.proto) |
| Transactions | /starknet/transactions/1 | [TransactionsRequest](./transaction.proto) | [TransactionsResponse](./transaction.proto) |
| Receipts | /starknet/receipts/1 | [ReceiptsRequest](./receipt.proto) | [ReceiptResponse](./receipt.proto) |
| Events | /starknet/events/1 | [EventsRequest](./event.proto) | [EventsResponse](./event.proto) |
| Kademlia (for discovery) | /starknet/kad/<chain_id>/<kademlia_version> |
| Identify (for discovery) | /starknet/id/<identify_version> |
