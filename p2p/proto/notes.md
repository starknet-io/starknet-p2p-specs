* request_id is handled at the protocol level, specifically to support multiplexing
* the protocol needs to handle multiple messages, but not accept any - according to the request
* number of messages (especially repeated ones) is capped for ddos
* if the number of repeated messages is not known upfront, a header message is sent with the number
** Maybe optional field in the first message
* GetBlocks result is: BlockHeader, Signatures, Proof, StateDiff*, Classes*
** Getting transactions/events is separate

* TBD: support subscription (e.g. a node wants to know of any new transaction that is in the pool
  of its peer, or any that is removed, or get a new block immediately.
