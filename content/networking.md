---
title: "Networking"
weight: 22
description: "Sockets for guests, built-in DNS, networks with aliases and several networks per container, and a class network through the relay. Live in the playground since v0.3.0."
tags: [networking]
---

Containers have a network (engine v0.10.0, extended in v0.14.0). **Networking is
live in the [playground]({{< relref "playground" >}}) since playground v0.3.0**
(engine v0.10.0 there), and joining a teacher's class from the page works.

```console
$ wctr network create lab
$ wctr run -d --name web --network lab netprobe serve 80
$ wctr run --rm --network lab netprobe get http://web/class
```

The second container reaches `web` by its name, through the network's DNS.

## Sockets for guests

A guest opens sockets with the WasmEdge socket extension of
`wasi_snapshot_preview1` (its V2 form, as stealthrocket/wasi-go serves it) and
preview 1's `sock_*` calls. A stock Go program that uses
`github.com/stealthrocket/net` v0.2.1 runs unchanged: HTTP and UDP between
containers, by container name.

## DNS by container name

Every network has a DNS server inside the engine, on UDP port 53, answering on
`127.0.0.11`, `127.0.0.1` and `::1`. It answers A and AAAA for the names on the
network and NXDOMAIN for anything else; **nothing is forwarded outside**. Guests
built with `-tags getaddrinfo` resolve through `sock_getaddrinfo` instead.

## Networks

- The default network is `bridge` (`10.88.0.0/16`); the other is `none`, which
  gives no sockets.
- `wctr network create|ls|inspect|rm|connect|disconnect|prune`, and `wctr run
  --network NAME`; the container's address is shown by `inspect`. The API has
  `/networks`.
- **Since v0.14.0**: `wctr run --network-alias NAME` gives a container more names
  on a user-defined network (replicas may share one); `wctr network connect
  [--alias NAME] NETWORK CONTAINER` attaches a running container at once, so a
  container can be on **several networks**; `disconnect [-f]` detaches it; and
  `prune [-f] [--filter label=K[=V]]` removes unused networks. Lists can be
  filtered by label, and `wctr events` has network events (`create`, `connect`,
  `disconnect`, `destroy`, `prune`).

## A class network

`wctrd --relay URL --relay-join-code-file FILE [--relay-ca CA] [--relay-network
class]` joins a class of the [relay]({{< relref "relay" >}}) and adds a network
on which containers get the class's addresses and names, under the teacher's
firewall. The join code is read from a file, which is refused if other users can
read it; the session token is kept with mode 0600 and refreshed. Since engine
v0.11.0 a host can also add and remove such a network at run time, which is what
the playground's "Join class" does.

## The WASI host

`wctrd --wasi engine` is the default, because sockets need it. `wctrd --wasi
wazero` gives no network.

## In the browser

In the playground, nothing leaves the page: there is no route to the Internet or
to the machine. Measured by `net bench` (200 sequential GETs, then one 8 MiB GET,
between two containers on the bridge) on an Apple M4 Max, under a load of 16 to
19, so **indicative only**:

| | latency p50 / p90 | throughput |
|---|---|---|
| Chromium (the page) | 1.1 / 1.4 ms | 106–112 MB/s |
| native: the same engine on wazero | 0.33 / 0.39 ms | 1.2 GB/s |
| native: Go `net/http` over loopback TCP (upper bound) | 0.05 / 0.07 ms | 3.8 GB/s |

The browser is about 3 times slower in latency and 11 times in throughput than the
same engine natively, because every socket read or write is a WASI call that
crosses from the container's worker to the engine's. Larger reads per call are an
open item.

## Limitations

- **No published ports (`-p`)** yet. Natively, `-p` is coming in engine v0.22.0; in
  the browser it is planned (the service worker would route a path to the
  container's port) and not implemented.
- A container on several networks binds to its primary address or to all of them
  (`0.0.0.0`), not to a secondary address alone.
- Aliases work on in-engine networks only; no IPv6 on in-engine networks; no
  `AF_UNIX`.
- The DNS shadows a container's own UDP server on port 53 at those resolver
  addresses, and an unconnected UDP `sendto` to the resolver is not answered.
