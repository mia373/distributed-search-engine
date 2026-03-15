# Distributed Search Engine

> **Note:** Due to academic integrity policies, the source code is not published. This README describes the project for portfolio purposes.

This project was built on top of the [NS3](https://www.nsnam.org/) network simulator.

## Repo structure

```bash
.
├── Makefile
├── README.md
├── VERSION
├── bindings
├── contrib
│   ├── upenn-cis553
│   │   ├── common
│   │   │   ├── penn-application.cc
│   │   │   ├── penn-application.h
│   │   │   ├── penn-log.cc
│   │   │   ├── penn-log.h
│   │   │   ├── penn-routing-protocol.cc
│   │   │   ├── penn-routing-protocol.h
│   │   │   ├── ping-request.cc
│   │   │   ├── ping-request.h
│   │   │   ├── test-result.cc
│   │   │   └── test-result.h
│   │   ├── dv-routing-protocol
│   │   │   ├── dv-message.cc
│   │   │   ├── dv-message.h
│   │   │   ├── dv-routing-helper.cc
│   │   │   ├── dv-routing-helper.h
│   │   │   ├── dv-routing-protocol.cc
│   │   │   └── dv-routing-protocol.h
│   │   ├── keys
│   │   │   ├── metadata0.keys
│   │   │   ├── metadata1.keys
│   │   │   ├── metadata2.keys
│   │   │   ├── metadata3.keys
│   │   │   ├── metadata4.keys
│   │   │   └── metadata5.keys
│   │   ├── ls-routing-protocol
│   │   │   ├── ls-message.cc
│   │   │   ├── ls-message.h
│   │   │   ├── ls-routing-helper.cc
│   │   │   ├── ls-routing-helper.h
│   │   │   ├── ls-routing-protocol.cc
│   │   │   └── ls-routing-protocol.h
│   │   ├── penn-search
│   │   │   ├── penn-chord-message.cc
│   │   │   ├── penn-chord-message.h
│   │   │   ├── penn-chord.cc
│   │   │   ├── penn-chord.h
|   |   |   ├── penn-key-helper.h
│   │   │   ├── penn-search-helper.cc
│   │   │   ├── penn-search-helper.h
│   │   │   ├── penn-search-message.cc
│   │   │   ├── penn-search-message.h
│   │   │   ├── penn-search.cc
│   │   │   └── penn-search.h
│   │   ├── test-app
│   │   │   ├── test-app-helper.cc
│   │   │   ├── test-app-helper.h
│   │   │   ├── test-app-message.cc
│   │   │   ├── test-app-message.h
│   │   │   ├── test-app.cc
│   │   │   └── test-app.h
│   │   └── wscript
│   └── wscript
├── doc
├── scratch
│   ├── results
│   │   ├── 10-dv.output
│   │   ├── 10-ls.output
│   │   ├── 29-dv.output
│   │   ├── 29-ls.output
│   │   ├── 40-dv.output
│   │   └── 40-ls.output
│   ├── scenarios
│   │   ├── 10-dv.sce
│   │   ├── 10-ls.sce
│   │   ├── 29-dv.sce
│   │   ├── 29-ls.sce
│   │   ├── 40-dv.sce
│   │   ├── 40-ls.sce
│   │   ├── m2.sce
│   │   ├── m2i.sce
│   │   ├── penn-chord.sce
│   │   ├── penn-search.sce
│   │   └── test.sce
│   ├── simulator-main.cc
│   └── topologies
│       ├── 10.topo
│       ├── 29.topo
│       ├── 40.topo
│       ├── m2.topo
│       └── small.topo
├── src
├── test.py
├── utils
├── utils.py
├── waf
├── waf-tools
├── waf.bat
├── wscript
└── wutils.py
```

## Project Overview

This project implements three core distributed systems components within the NS3 network simulator:

1. **Distance Vector (DV) Routing Protocol** — implemented in `contrib/upenn-cis553/dv-routing-protocol/`. Nodes periodically exchange distance vectors with neighbors and converge on shortest paths using the Bellman-Ford algorithm.
2. **Link State (LS) Routing Protocol** — implemented in `contrib/upenn-cis553/ls-routing-protocol/`. Nodes flood link-state advertisements across the network and compute shortest paths locally using Dijkstra's algorithm.
3. **Chord DHT + PennSearch** — implemented in `contrib/upenn-cis553/penn-search/`. A Chord-based distributed hash table that supports consistent hashing across nodes, on top of which a peer-to-peer keyword search engine (PennSearch) is built.

The simulator entry point is `scratch/simulator-main.cc`. Network topologies and test scenarios are defined under `scratch/topologies/` and `scratch/scenarios/` respectively. Simulation outputs are stored in `scratch/results/`. The `src` directory contains the NS3 source code (unmodified).

## Compiling and running the simulator

### Compiling

1. Run `./waf configure` on first use. If the `waf` executable lacks execution permissions, fix that with `chmod u+x waf`.
2. Compile the simulator by running `./waf`. This may take a few minutes depending on your machine.

### Running

The simulator is run via the [WAF](https://waf.io/) build system (distinct from NS3 itself — NS3 uses WAF as its build system). The general command is:

`./waf --run "simulator-main --routing=<NS3/LS/DV> --scenario=./scratch/scenarios/<SCENARIO_FILE_NAME>.sce --inet-topo=./scratch/topologies/<TOPOLOGY_FILE_NAME>.topo --project=<1/2>"`

For example, to run the LS-40 simulation using the LS routing implementation:

`./waf --run "simulator-main --routing=LS --scenario=./scratch/scenarios/40-ls.sce --inet-topo=./scratch/topologies/40.topo --project=1"`

To run the Chord/PennSearch simulation with the "m2" scenario using built-in NS3 routing:

`./waf --run "simulator-main --routing=NS3 --scenario=./scratch/scenarios/m2.sce --inet-topo=./scratch/topologies/m2.topo --project=2"`

## Troubleshooting

If at any point you are getting compiling errors, try cleaning the build cache of the NS3 simulator by running `./waf distclean`. This will mean that `waf` will need to recompile all source code the next time it runs.

Tips on using the provided hashing functions in `penn-key-helper.h`:

- Hash a node using its `ns3::Ipv4` address
- Hash a lookup term using a `std::string`
- Only use `PennKeyHelper::KeyToHexString()` for printing
