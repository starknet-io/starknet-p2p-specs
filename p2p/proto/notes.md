* Goals: consensus, scale, validator-fullnode separation, cleanup
* request_id is handled at the protocol level, specifically to support multiplexing
* the protocol needs to handle multiple messages, but not accept any - according to the request
* number of messages (especially repeated ones) is capped for ddos
* if the number of repeated messages is not known upfront, a header message is sent with the number
** Maybe optional field in the first message
* GetBlocks result is: BlockHeader, Signatures, Proof, StateDiff*, Classes*
** Getting transactions/events is separate
* Protocols: blocks (headers+bodies or separate?), transactions, events, receipts, mempool. States? fastsync?
* Using Merkle trees so can decommit individual transactions, events, etc. e.g. node that tracks just one contract for events
** Can maybe put all data into one tree (e.g. a receipt will contain events as well). Can support requesting just partial (and getting hashes for the rest, a mini merkle)
* TBD: consensus messages
* TBD: reverted transactions
* TBD: stark friendly hashes or not (calculate in os? so light nodes don't trust the consensus)



* TBD: many merkles in the block header


