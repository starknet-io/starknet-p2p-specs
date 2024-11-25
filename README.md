# StarkNet P2P Specification

This repo contains and tracks the specification of the P2P protocol for StarkNet nodes.

## Starknet P2P Networks
There are three different networks that serve different type of applications:
* [Sync](./p2p/proto/sync/protocols.md) - responsible for downloading information about blocks existing in Starknet
* [Mempool](./p2p/proto/mempool/mempool.md) - responsible for handling transactions pending for insertion to Starknet
* Consensus - responsible for creating and adding new blocks to Starknet via staking

Each network has a separate discovery network. The Kademlia protocol names for those networks are:
* /starknet/kad/<chain_id>/sync/1.0.0
* /starknet/kad/<chain_id>/mempool/1.0.0
* /starknet/kad/<chain_id>/consensus/1.0.0

A node that wants to connect to multiple networks should connect through a different port for each network.

The reasons to split the network by protocols and not have a singular network are:
* Allow adding custom security policies and authentication requirements for each network separately.
* Filter out nodes that don't support the capabilities you need
* Allow modular development of node's code stack by having a node comprised of different implementations for each network.
