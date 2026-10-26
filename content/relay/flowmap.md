---
title: "The live flow map"
linkTitle: "Flow map"
weight: 15
description: "A page the relay serves that draws the class's flows as they happen: who talks to whom, how much, and what the firewall refused."
tags: [relay, flowmap]
---

Since relay **v0.2.0**.

```sh
wctr-relay flowmap --class <class> --url <the relay's https address>
```

prints a link; open it on the projector. Each student is a box holding their
containers, coloured by security group. Each flow is a line between two
containers whose width follows its throughput. A **refused flow is a dashed
red line**, and the event log beside the map says which rule was missing
("no ingress rule of … allows it"). Click a container for its address and
groups, or a flow for its counters and the rules that allowed it; the arrow
keys move from container to container. The filter box dims everything that
does not match a student, a container or a group.

## Who can see what

| Viewer | Credential | Sees |
| --- | --- | --- |
| The teacher | the link from `wctr-relay flowmap`: a read-only ticket in the URL fragment (never sent to a server, removed from the address bar when the page loads), valid 12 hours | the whole class |
| A student | their own session token (`#class=…&session=…`) | only the flows that touch their own containers, and the containers at the other end |

The ticket allows watching that class's map and nothing else: it cannot
change a rule, list the groups, or join the class.

## How it works

The relay aggregates its flow log per edge — source container, destination
container, protocol and port — and streams one update per tick (4 per
second) over `FlowLogService.WatchFlowMap`, a server stream of the Connect
protocol. A slow page gets the newest map, never a backlog. An update holds at
most 400 containers, 2 000 flows and 200 events.

The page is a Go WebAssembly application built on
[go-widgets](https://github.com/go-widgets/toolkit)' `GraphView` (a
node-link graph over a view model, added to go-widgets for this page and
reusable elsewhere) and `LogView`. The relay serves it itself, under a strict
Content-Security-Policy: `default-src 'none'; script-src 'self'
'wasm-unsafe-eval'; connect-src 'self'` — no inline script, nothing loaded
from anywhere else, so it works on a classroom LAN with no internet.

## Getting a binary that has it

The page's WebAssembly is about 5 MB, so it is not in the repository: the
binaries attached to each
[release](https://github.com/go-wasmcontainers/relay/releases) carry it. From
source:

```sh
go run ./internal/buildflowmap
go install -tags flowmap ./cmd/wctr-relay
```

A plain `go install …@latest` serves the page, which then says the map is
missing from that build.

## Not yet

Pause and replay of the last minutes; the relay keeps the last 1 000 events,
the page shows the last 200.
