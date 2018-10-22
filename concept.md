

# Subspace Protocol -- Conceptual Design Document

This document describes the conceptual framework for understanding the operation and flow of the subspace protocol. The goal is to remain at a high level of abstraction though at times it will delve into lower level implementation details as required for understanding. 

## Goals

1. Provide a decentralized No-SQL database as a service with a simple put/get API that can run in all javascript environments.
2. Incentivize clients to reserve space on the network by ensuring the database is easy to use, fault tolerant, persistent, and fast.
3. Incentivize hosts to share spare space and bandwidth on compute devices by earning a crypto token.
4. Incentivize farmers to secure the Subspace Credit Ledger (SSCL) through a Proof of Space consensus mechanism.

## Credit Ledger

To begin, imagine a network consisting of a single node, acting as both a host and a farmer. The node must first pledge some amount of free disk space to the network. Let's assume it is 10 GB. This space is used to generate a hard to pebble graph, resulting in a proof of space that will fill this entire 10 GB of the sector pledged. The merkle hash of this proof is wrapped into a *pledge* transactoin that is gossiped to all farmers and held in the memory pool of pending transactions. The node will now validate it's proof and add the transaction to draft of the next block of the ledger. Since this is the only node on the network it will publish the block and allocate the coinbase transaction back to it's own account on the ledger. 

Block 1
Coinbase Tx Node 1 -> 100 Subspace Credits (block reward)
Pledge Tx: Node 1 -> 10 GB Space
Cost of Storage: 10 SSC / GB - month

The node will also compute the Cost of Storage (CoS) based on the total credit supply and total amount of space pledged. In this case the CoS is 10 SSC / GB (100 / 10). Block 1 will have a block id, which is the hash of the block content. This block id will be used as the challenge for the next block. Each farmer will check it's proof of space, which includes a solution set for block challenges. A quality function will be applied between the challenge and each solution, with the solution that provides the closest solution to the challenge being proposed by the farmer (and gosspied to all other farmers). In this case since there is only a single farmer, it will have the best solution by default. If there are multiple farmers, each farmer will have a probablity of having the best solution in proportion to the amount of space they have pledged as a fraction of all space pledged to the network by farmers. Note that the block itself in a different sector than the 10 GB sector it has pledged to the network.

After block 1 has been published a second node joins the network as both a host and a farmer, pleding 10 GB of space in a similar fashion. Both nodes will validate the proof and then compute their best solution to the block challenge. In this case, node 2 has a better solution (higher quality) than node 1 and earns the right to publish the second block.

Block 2
Coinbase Tx: Node 2 -> 100 Subspace Creidts
Pledge Tx: Node 2 -> 10 GB Space
Cost of Storage: 10 SSC / GB - month

After block 2 there are now 20 GB of space pledged to the network and 200 Subspace Credits in circulation, evenly split between the two nodes. This Cost of Storage remains the same since both space and credits increased at the same rate.

Now Node 1 wishes to store some data on Subspace. To do this it first creates a storage contract. In this case the node wants to store 1 GB of data for one month with a 2x replication factor. This will cost 20 subspace credits (10 x 1 x 1 x 2). The storage contract is a special transanction where the to address is the nexus contract. The nexus contract is a special address where all funds for client transactions are placed in escrow over the life of the contract. Funds in the nexus account may only be spent by farmers and paid out to hosts, in accordance with the rules of the nexus payment.

When the storage contract is created an ECDSA keypair is also created. The node (client) will keep the private key while the public key will be attached to the transaction, with the hash of the public key serving as the contract id. Each contract corresponds to a *client database* which represents some fraction of the entire *subspace database*. Each *client database* is divided into *shards* of 100 MB each. Each *shard* is replicated on R hosts, where R is the replication factor set in the contract. To obtain the shard ids, the contract id is iteratively hashed. Since this contract is for 1 GB it would be comprised of 10 shards. 

shard_id_1 = hash(contract_id)  
shard_id_2 - hash (shard_id_1)  
...

In this manner any node can determine the shard ids given only the contract id. Shards are assigned to nodes using weighted rendezvous hashing. Imagine a network of 100 nodes that most hold 10 shards. For each shard compute the score of node_id | shard_id using a custom hash function, with the node receiving a weight based on it's relative pledge of space in proportion to all space pledged to the network. The node with the higest score will be used to store the shard, with the next node in the ring (counter-clockwise) storing the replica, and so on for as many replicas as are required by the storage contract. As long as each node knows about all other nodes and their relative weight, it can compute the correct host/s for the shard using this approach. Once the contract is posted to the ledger the winning farmer will compute the shards and assign them to the appropriate nodes in the network.

Each node will then create an empty shard object in it's pledged sector, and add the contract id, public key, and all shard_ids to its local index.

## Transport Layer

## Network Topology



## Storage Contracts

## Put and Get Requests