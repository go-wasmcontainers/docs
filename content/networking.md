---
title: "Networking"
weight: 22
description: "Sockets for guests, built-in DNS and networks in engine v0.10.0, and a class network through the relay. Not in the playground yet."
tags: [networking]
---

Engine v0.10.0 gives containers a network. **Networking in the playground is
not live yet**: it is the next milestone. Everything below is the native engine.

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
- `wctr network create|ls|inspect|rm`, `wctr run --network NAME`, and the
  container's address is shown by `inspect`. The API has `/networks`.

## A class network

`wctrd --relay URL --relay-join-code-file FILE [--relay-ca CA] [--relay-network
class]` joins a class of the [relay]({{< relref "relay" >}}) and adds a network
on which containers get the class's addresses and names, under the teacher's
firewall. The join code is read from a file, which is refused if other users can
read it; the session token is kept with mode 0600 and refreshed.

## The WASI host

`wctrd --wasi engine` is now the default, because sockets need it.
`wctrd --wasi wazero` gives no network.

## Limitations

- No published ports (`-p`) and no `network connect`.
- No IPv6 on in-engine networks.
- The DNS shadows a container's own UDP server on port 53 at those resolver
  addresses.
- Throughput has not been measured yet.
- Networking in the playground is not live yet (next milestone).
