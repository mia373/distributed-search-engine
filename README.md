# PennSearch: Distributed Search Engine

> **Note:** Due to academic integrity policies, the source code is not published. This README describes the project for portfolio purposes.

## Project Overview

PennSearch is a distributed search engine built inside the ns-3 network simulator. The project models a peer-to-peer system where simulated nodes join a Chord ring, publish keyword-to-document metadata into a distributed inverted index, and answer multi-keyword search queries by routing work through the overlay.

The system combines three layers:

1. A simulated IP network built with ns-3 topologies and point-to-point links.
2. Custom routing protocols used by the simulator to deliver packets across changing network graphs.
3. A Chord-based application layer that stores and queries search metadata across participating nodes.

The goal was not to build a web search UI. The interesting part is the distributed systems behavior: routing, membership, stabilization, key ownership, lookup forwarding, index placement, query execution, and correctness under joins and leaves.

## Why This Project Matters

This project is a compact version of several real distributed systems problems:

- How do nodes discover each other and maintain a consistent overlay?
- How can a key be mapped to the correct storage node without centralized coordination?
- How should data move when responsibility changes after a node joins or leaves?
- How do search queries execute when each keyword may be stored on a different machine?
- How can correctness be verified when events are asynchronous and the network topology is simulated?

It gave me hands-on experience with protocol implementation, event-driven simulation, packet/message design, distributed indexing, and debugging systems where behavior emerges over simulated time rather than a simple call stack.

## Tech Stack

- C++
- ns-3 network simulator
- Waf build system
- OpenSSL hashing utilities
- Docker / Docker Compose for reproducible development
- Custom simulation scripts and topology files

## Core Features

### Chord Distributed Hash Table

The project implements a Chord-style peer-to-peer overlay. Each participating node receives a hashed identifier, and the identifier space is organized as a logical ring.

Supported behavior includes:

- Node join through an existing bootstrap node
- Ring initialization from a single node
- Successor and predecessor maintenance
- Periodic stabilization
- Finger table maintenance for faster lookup routing
- Predecessor health checking
- Key lookup by identifier
- Ring state inspection for debugging and grading
- Graceful node leave behavior
- Callback integration with the search layer

The Chord layer is responsible for answering the question: "Which node owns this key?"

### Distributed Keyword Publishing

Documents are represented through metadata files containing document IDs and associated keywords. When a node publishes a metadata file, each keyword is hashed into the Chord keyspace.

The system then:

- Hashes each keyword deterministically
- Uses Chord lookup to locate the node responsible for that keyword
- Sends the keyword/document pair to the responsible node
- Stores the mapping in a distributed inverted index
- Handles multiple documents sharing the same keyword
- Supports data redistribution when ownership changes

Conceptually, the distributed index looks like this:

```text
keyword hash -> Chord lookup -> responsible node -> set of matching documents
```

This allows the index to be partitioned across many nodes instead of stored centrally.

### Multi-Keyword Search

The search layer supports queries containing one or more keywords. For a multi-keyword query, the system must locate the inverted list for each keyword and compute the intersection of the matching document sets.

The query flow is roughly:

```text
query originator
  -> lookup first keyword owner
  -> retrieve candidate document list
  -> lookup next keyword owner
  -> intersect results
  -> repeat until all terms are processed
  -> return final document set to originator
```

This design avoids requiring every node to know the entire index. Each node only owns the keyword lists assigned to its Chord key range.

### Dynamic Membership

The system supports nodes joining and leaving the Chord ring during a simulation. This is one of the most important parts of the project because key ownership changes as the ring changes.

The implementation handles:

- Joining an existing overlay through a known bootstrap node
- Updating successor and predecessor pointers
- Maintaining finger tables over time
- Detecting stale predecessor state
- Moving relevant inverted index entries to newly responsible nodes
- Preserving search correctness after topology-level and overlay-level changes

### Custom Message Protocols

The project defines custom packet headers and message types for both the overlay and the search application.

Chord-level messages include:

- Ping request / response
- Stabilization request / response
- Notify request
- Ring state traversal
- Find-successor request / response
- Leave notification

Search-level messages include:

- Ping request / response
- Publish request
- Search request
- Search response

Each message type has explicit serialization and deserialization behavior so it can be transmitted through ns-3 sockets as simulated network traffic.

### Routing Protocol Work

In addition to PennSearch, the repository includes custom routing protocol work from the broader course project.

Implemented or integrated modules include:

- Distance Vector routing
- Link State routing
- Routing message definitions
- Neighbor and routing table state
- Ping support for route validation
- Scenario-driven route testing
- Comparison against ns-3 routing behavior

These routing components support the lower-level packet delivery behavior used by the simulator.

## System Architecture

```text
+-------------------------------------------------------------+
| Simulation Driver                                           |
| - Reads topology files                                      |
| - Reads scenario scripts                                    |
| - Schedules events over simulated time                      |
+--------------------------+----------------------------------+
                           |
+--------------------------v----------------------------------+
| ns-3 Network Layer                                          |
| - Nodes, links, interfaces, addresses                       |
| - Point-to-point channels                                   |
| - Custom or built-in routing                                |
+--------------------------+----------------------------------+
                           |
+--------------------------v----------------------------------+
| Routing Layer                                               |
| - Distance Vector                                           |
| - Link State                                                |
| - ns-3 baseline routing                                     |
+--------------------------+----------------------------------+
                           |
+--------------------------v----------------------------------+
| Chord Overlay                                               |
| - Ring membership                                           |
| - Successor/predecessor state                               |
| - Finger tables                                             |
| - Key lookup                                                |
+--------------------------+----------------------------------+
                           |
+--------------------------v----------------------------------+
| PennSearch Application                                      |
| - Keyword publishing                                        |
| - Distributed inverted index                                |
| - Multi-keyword search                                      |
| - Result aggregation                                        |
+-------------------------------------------------------------+
```

## Repository Layout Used During Development

The private project repository was organized around the ns-3 source tree. The most important project-specific areas were:

```text
contrib/upenn-cis553/
  common/                 Shared application, routing, logging, and test utilities
  dv-routing-protocol/    Distance Vector routing implementation
  ls-routing-protocol/    Link State routing implementation
  penn-search/            Chord overlay and PennSearch application
  keys/                   Sample keyword/document metadata files
  test-app/               Simple simulator test application

scratch/
  simulator-main.cc       Simulation entry point
  scenarios/              Timed simulation commands
  topologies/             Network topology definitions
  results/                Expected outputs for local routing checks

Dockerfile
docker-compose.yml        Reproducible Linux development environment
```

The public version of this project does not include the graded implementation files.

## Example Simulation Scenario

A typical PennSearch scenario performs actions like:

```text
enable verbose Chord/search logging
wait for network startup
node 0 creates a Chord ring
node 1 joins through node 0
node 2 joins through node 1
node 0 publishes a metadata file
node 1 publishes another metadata file
node 0 searches for multiple keywords
node 1 leaves the ring
node 2 runs a search after membership changes
```

This kind of scenario verifies both the happy path and the dynamic behavior of the overlay.

## Example Data Model

Metadata files map document IDs to keywords:

```text
doc0: T1 T2 T3 T4
doc1: T2 T3 T4 T5
doc2: T93 T3 T4 T5 T6
doc3: T7 T8 T9 T10
doc4: T1 T2 T4 T5
```

After publishing, the system stores inverted lists by keyword ownership:

```text
T1 -> {doc0, doc4}
T2 -> {doc0, doc1, doc4}
T3 -> {doc0, doc1, doc2}
```

A query for `T1 T2` should return the intersection:

```text
{doc0, doc4}
```

In the actual system, each keyword may be owned by a different Chord node, so the query must be routed through the overlay to collect and intersect partial results.

## Testing Strategy

Testing was scenario-driven and focused on observable distributed behavior rather than isolated unit output alone.

Key test categories:

- Ring formation from one node to multiple nodes
- Successor and predecessor correctness after joins
- Lookup routing correctness across the Chord keyspace
- Finger table maintenance over simulated time
- Keyword publishing to responsible nodes
- Multi-keyword search result correctness
- Search behavior after node joins
- Search behavior after node leaves
- Routing correctness over 10-node, 29-node, and 40-node topologies
- Logging consistency for grading and debugging

The most useful debugging signals were structured logs for:

- Lookup issue, forwarding, and result events
- Ring state traversal
- Publish and store events
- Inverted list transfer
- Search request and search result events

## Engineering Challenges

### Keeping Chord and Search Responsibilities Separate

The Chord layer should know how to find the owner of a key, but it should not know the meaning of a keyword or document. The search layer should know about inverted indexes, but it should not implement overlay routing.

I designed a callback interface so Chord could report lookup results to PennSearch without knowing why a lookup was requested: 

- Chord reports lookup success to PennSearch.
- PennSearch decides whether the lookup was needed for publish, search, or redistribution.
- Chord reports membership changes that require index movement.

This separation made the system easier to debug and reason about.

### Redistributing the Inverted Index on Membership Changes

When a node joins, some keyword entries that belonged to its successor now fall in the new node's key range. When a node leaves, its stored keyword lists must reach its successor before it stops responding.

I implemented both directions:

- On a join, the successor identifies entries in the transferred key range and sends them to the new node.
- On a leave, the departing node pushes its entire inverted index to its successor before shutting down.
- Key range checks had to account for wraparound at the 32-bit identifier boundary to avoid silently dropping or duplicating entries during transitions.

### Tracking In-Flight Search State

A multi-keyword search does not complete in one step. The originator resolves keywords one at a time, each requiring a separate Chord lookup on a potentially different node, with intersection happening between results. Each step is asynchronous — a callback fires when the lookup returns.

I built state objects that tracked the pending keyword list, the accumulated document intersection, and the original requester's address. Each callback advanced the search: issue the next lookup, intersect when the result arrives, and return the final document set once all terms are processed.

### Debugging Behavior That Only Appears at Runtime

A bug in this system typically appeared as "the wrong documents came back" rather than a crash, with the root cause spread across multiple hops and simulated timestamps.

I added structured lookup logs to make the failure path visible:

- Where a lookup was issued and by which node
- Which nodes forwarded it
- Which node claimed ownership of the key
- What the inverted list at that node contained
- How the final document intersection was produced

## What I Learned

- How Chord maps keys to nodes and maintains ring consistency
- How finger tables improve lookup efficiency over successor-only routing
- How distributed inverted indexes can support keyword search
- How to write and debug custom ns-3 applications
- How to design serialized protocol messages in C++
- How to reason about eventual consistency in a simulated network
- How to build scenario-based tests for distributed systems
- How important observability is when debugging asynchronous protocols
