# Subspace System Architecture Document

A high level overview of the design and components of the Subspace protocol.

- [Core Modules](#core-modules)
  * [Crypto](#crypto)
  * [Profile](#profile)
  * [Database](#database)
  * [Relay](#relay)
  * [Local Hash Table](#local-hash-table)
  * [Distributed Hash Table](#distributed-hash-table)
  * [Remote Procedure Calls](#remote-procedure-calls)
  * [Ledger](#ledger)
- [API Modules](#API-Modules)
  * [Host](#host)
  * [Client](#client)
  * [Farmer](#farmer)
  * [Gateway](#gateway)
- [Core Implementations](#core-implementations)
  * [Server](#server)
  * [Browser](#browser)
  * [Mobile](#mobile)
  * [Desktop](#desktop)
  * [BitBot](#bitbot)
- [Subspace Implementations](#subspace-implementations)
  * [Subspace Explorer](subspace-explorer-spa)
  * [Subspace Console](subspace-console-web-app)
  * [Subspace Full Node](subspace-full-node-mobile--desktop)
  * [Subspace Full Node (headless)](subspace-full-node-server)

## Core Modules

### Crypto

Library for a variety of cryptographic helper methods.

Native Crypto
* Generate and verify hashes
* Generate random numbers (for symmetric passwords)

OpenPGP protocol
* Generate ECDSA key pairs (profiles & records)
* Sign and verify messages
* Asymmetric encryption and decryption of symmetric keys
* Generate and verify join and leave proofs (signed timestamps)
* Need to performance test on mobile

AES-JS
* symmetric encryption and decryption of record contents
* May be able to use OpenPGP or native crypto modules instead

Needed
* Simple VDF (maybe balloon hash) for creating unique replicas of records on each host
* Simple proof of space (maybe hard to pebble graphs) for pledging space
* Hierarchical Deterministic Wallets for key generation from seed phrases

### Profile

Simple module that uses crypto module to create a profile (including key-pair), expose as an object, and persist to local storage. Each node has a persistent identity on the network which is the SHA256 hash of it's ECDSA public key.

### Database

* Encodes the records in a schema following BEP-44 (partially implemented)
* Uses crypto to generate symmetric keys and encrypt/decrypt records
* Persists records to disk based on env (put/get)
  * Node: node-persist
  * React-Native: async storage
  * Browser: local-forage (wrapper around local storage, indexed db, and web db)
* Uses crypto to generate VDFs (not yet) before persisting
* Synchronizes state across hosts (not yet) plan to use CRDTs from Y.js
* May explore using GraphQL to include a schema, type system, and query language

### Relay

Creates a structured overlay network between peers. Heavily based on peer-relay library. All DHT and RPC messages are passed through this overlay network / transport layer.

* Nodes with public IP address may bootstrap the network and act as a gateway
* All other nodes must know the web socket URL of at least one other gateway node
* Gateway nodes implement ICE (both STUN and TURN) and connect other nodes directly via WebRTC
* Nodes organize their known peers into a routing table using k-buckets based on XOR distance between node IDs (following Kademlia)
* Allows messages to be passed between any two peers on the network while knowing on the nodeID (forms the basis of RPC messages)
* Exposes a UDP like socket interface for the DHT (as a hack)

Modifications

* Node IDs are passed in on join, instead of generated at random
* Node IDs are extended from 20 bytes to 32 bytes in length
* WRTC connection event had to be extended to detect active WebRTC connects / disconnects

Ideas

* WebRTC connections are resource heavy. The number of connections should be based on the implementation, increasing from mobile -> browser -> desktop -> server
* Network topology and link distribution seems non-uniform, need a better way to structure the network  

### Local Hash Table

* Local Hash Table (LHT) is an in memory ES6 map that is persisted to disk between sessions.
* It keeps a record of all active hosts (not nodes) on the network by recording their join proofs into the map object
* Primary purpose is to track uptime of nodes globally
* Secondary purpose is to provide a lookup directory for puts
* Entries are added with a valid join proof
* Entries are removed with a valid leave or failure proof
* Includes a Random Peer Sampling (RPS) method to select peers for record puts()
* On joining the network a node will receive its LHT from the gateway and validate each join independently
* Valid changes are added to the membership delta and batched for gossip to neighbors at an interval
* Updates are eventually consistent based on the size of the network

Currently memDelta and LHT are two separate modules that need to be combined. Also RPS is in relay and needs to be extracted into LHT.

### Distributed Hash Table

BitTorrent DHT module from WebTorrent. This is a sloppy Kademlia DHT that has been modified and includes an opinionated implementation of BEP-44. This provides a global index of all records in SSDB. Given a record id (hash of public key) it will return the id/s of the node/s holding the key as a series of async events.

Modifications
* K-Buckets store Node IDs, not IP/Port since this is already handled by Relay Module

Methods

* AddNode(): called when a new relay network connection is added to the network, adding them to the DHT as well
* Announce(): called when a host puts a new record
* Lookup(): called by any host/client searching for a key, will return the nodeId that is holding the key

Ideas
* There is a PR from Mafintosh to include unAnnounce() which would need to be called if a host dropped a key
* Need to seperate the notion of long lived announces from hosts and short term announces from clients
* Need a way to multiannounce multiple nodes for a single key
* Need a way for any nodes to unannounce keys for any host that has failed
* Need a way to query the DHT for all keys for a given node (in case of failures or to validate keys)

### Remote Procedure Calls  

Originally called messenger. A series for requests and responses messages passed between subspace nodes.

Join(): Connects to the network through a gateway node with a signed message verifying the public key. On verification the gateway will add the new node to it's LHT and membership Delta (to be sent at update interval).

Leave(): For graceful shutdowns, this sends a signed leave message from the node leaving the network. On verification the node will be dropped from the LHT (or clock stopped) and add to them memDelta.

Fail(): Not implemented. Still researching a low-overhead algorithm. Looking at Parsec consensus from MaidSafe (based on HashGraph). The ideas is allow all of the neighbors of a node to come to consensus on whether a node has failed or not, if one node detects a failure. This is the classic Byzantine Generals problem. Starting point would be for each node to store a partial graph of the network that included at least all the nodes that it's direct neighbors are connected to, then route messages to them on failure to decide if the node has failed or just been obstructed by the network.

Update(): Gossips the latest membership delta (memDelta) including all valid joins, leaves, and failures to all neighbors at a regular interval (currently 5 seconds)

Get(): Implements the full get workflow (as described in the white paper) between two nodes (one of which must be a host).

Put(): Implements the full put workflow (as described in the white paper) between two nodes (one of which must a host)

Balance(): Not implemented. Would call a host from the LHT using RPS, compare size of shard, and swap records accordingly, in such a way that hosts cannot lie about what records they are holding.

GetKey(): Retrieves a specific key by id from a host (as opposed to querying the DHT). Implemented for the subspace explorer web app.

GetKeys(): Retrieves an array of all keys stored on a given host. Implemented for the subspace explorer web app.

Currently uses an awkward case syntax, has redundant ack messages, and somewhat spaghetti code with events. Needs to be refactored to be cleaner and more logical.


### Ledger

The blockchain or distributed ledger that storage payments are based on. Not at all implemented yet. Roughly modeled on the bitcoin blockchain and may be able to take whole chunks from bcoin javascript implementation. Will also be a re-implementation of either Chia or SpaceMint consensus, based on which one is finished first or works best.

First question is to whether to store blocks as immutable records on SSDB or have a separate storage/network module that is only used by hosts, but that seems redundant. If stored as immutable records on SSDB then a way to pay the cost of storage need to be devised that does not break the system, such as having a portion of the coinbase transaction set aside for the cost of immutable storage.

Otherwise the data structures for transactions and blocks should follow the Bitcoin model closely, including the Script smart contract language for implementing storage contracts. Proofs of space for farming should be the same as proofs of space for hosting, or re-designed to meet requirements for both. Client Data Contracts will be Pay to Public Script Hash, which are paid to the nexus contract. Nexus contract may be hardcoded into the blockchain or exist as a special contract that is inserted at genesis. Finally a means of reconciling storage payments over time and determining the cost of storage needs to implemented in the farmer logic.

Key Problems Summary
* Implement transactions, blocks, and contracts as SSDB immutable data structures
* Develop proof of space for seeding (ideally compact)
* Derive consensus algorithm based on quality function (based on Chia/SpaceMint)
* Implement client data contracts and nexus contracts in Script


## API Modules

The full reference implementation would include all four modules, i.e subspace.host.seed() or subspace.client.get().
Light clients might only use a single module, i.e. subspace.connect(), subspace.put() in the browser.

### Host

Pledges some amount of local storage to the network and is rewarded with subspace credits based on uptime and space pledged.

Connect(): Joins the subspace network as a relay node, with a join proof, gets the LHT
Seed(): Generates a proof of space of some amount of local disk space that is gossiped to farmers, listens for puts and gets after it is validated and added to the ledger.
Invoice(): Submits a request for nexus payment once their interval cycle has elapsed.
Disconnect(): Gracefully leaves the subspace network, with a leave proof, announcing departure to all peer holders who have similar keys.


### Client

Able to get and put records from the network.

Connect(): Joins the subspace network as a light client
Disconnect(): Leaves the subspace network
Put(): Add a record to SSDB with a valid storage contract and announces ephemerally on the DHT
Get(): Retrieves a record and announces ephemerally on the DHT

### Farmer

Able to validate transactions and propose new blocks for the chain.

Seed(): Seeds free disk space with a set of solutions to the block challenge. Non-interactive, does not have to be submitted for validaton.
Connect(): Connects to the network as a farmer, downloads the blockchain, listens for new transactions that will be stored in the memory pool
Disconnect(): Leaves the network gracefully.

### Gateway

Serves as an entry point for new nodes on the network. Must have a public ip address. Intended for headless operation may be have a management console.

Connect(): Joins the network simply to be a node in the peer relay network, gossiping the LHT and other messages between peers.
Disconnect()

## Core Implementations

All are currently in javascript. Goal is to move to Typescript soon.

### Server

The core protocol currently written for Node JS. Uses Node WebRTC for as the transport adapter and Node Persist as the storage adapter. Currently run as a daemon from the console. Goal is to package with Zeit Packager or Docker to simplify deployments.

### Browser

The core protocol bundled using browserify. Used native window.webrtc for the transport layer and localforage as the storage adapter. Currently includes a simple UI to see neighboring nodes, get records by id, and put records while visualizing the json data.

### Mobile

The core protocol with several modification in React Native. Uses RN-WebRTC as the transport layer, RN Async Storage as the store layer, Node-Libs React Native to provide similar node style interface, along with half a dozen other libraries to fill out all of the node libs.

### IoT  

The core protocol running in Rasbian (Debian Ubuntu) on a Raspberry Pi for the BitBot. Everything except for node-webrtc will port over flawlessly. Unable to get node-webrtc to build for ArmV7 architecture. Have been able to use electron-webrtc using old version of electron. May also be able to run x86 VM on Pi in run standard server protocol here.

### Desktop

Currently running core protocol as a background process for OSX and Ubuntu. Have not tested on Windows. Next step is to wrap with a simple electron app, first as a tray icon and later with a full UI similar to the mobile wallet experience.

## Subspace Implementations

### Subspace Explorer (SPA)

A hosted web app similar to blockchain.info or ether scan but for the subspace network. Later may include the ability to create wallets, submit transactions, and store records on SSDB with a valid key or wallet.

### Subspace Console (Web App)

A secure hosted app meant primarily for developers who are building apps on the subspace network or reserving storage. Similar to the AWS or Google Cloud console in appearance and functionality.

### Subspace Full Node (Mobile & Desktop)

The authoritative app/s for sharing space on the subspace network from a mobile device or desktop computer. Mobile is currently in beta with in both the app and play stores through FastLane.

### Subspace Full Node (Server)

A headless version of subspace meant to be run by technical hosts/farmers running on dedicated hardware in the cloud. Goal is to make it as simple as possible for them to deploy on existing cloud service providers.
