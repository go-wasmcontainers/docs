---
title: "Relay"
weight: 20
description: "wctr-relay, the classroom relay and firewall: install it, start a class, open flows with security groups."
tags: [relay, networking]
---

`wctr-relay` connects the WASI containers of a class's browsers (or engines)
through one relay that is also a **firewall**, with security groups modelled
on OpenStack Neutron. It runs on the teacher's laptop for a classroom LAN, or
on a server for remote students with OpenID Connect sign-in. Release
**v0.2.0** of [go-wasmcontainers/relay](https://github.com/go-wasmcontainers/relay)
covers milestones M-relay-1, M-relay-2 and M-relay-2b (the
[live flow map]({{< relref "flowmap" >}})) of the
[design]({{< relref "design/classroom-network" >}}).

{{< callout type="info" >}}
**What works today**: the relay, its firewall, its control API, the
`wctr-relay` command and the live flow map, tested natively and, for the
flow map, in Chromium and Firefox. The engine does not use the relay yet (its
network driver is being written), so containers in the playground cannot
reach each other through it yet.
{{< /callout >}}

## Install

Download the binary for your system from the
[releases](https://github.com/go-wasmcontainers/relay/releases) (it includes
the flow map), or build it:

```sh
go install github.com/go-wasmcontainers/relay/cmd/wctr-relay@v0.2.0   # without the flow map page
```

Pure Go, `CGO_ENABLED=0`; Linux, macOS and Windows.

## Start a class (teacher)

```sh
wctr-relay init --name relay.local --ip 192.168.1.10   # state directory, admin token, internal CA
wctr-relay serve --listen :8443 &
wctr-relay class create "Networking 101"               # prints the class id and the join code
```

Students join with the code. Each gets a subnet (`10.42.<n>.0/24`), each
container an address and the name `<container>.<student>.class`. A new class
starts **isolated**: no ingress, egress to the teacher's subnet only. Opening
flows is the lab:

```sh
wctr-relay sg create web --class <class>
wctr-relay rule add web --class <class> --ingress --proto tcp --port 80
wctr-relay rule add default --class <class> --egress --proto tcp --port 80
wctr-relay port set alice/web --class <class> --sg default,web
wctr-relay flows --class <class> --follow        # every allow and deny, with the rule that decided
wctr-relay explain --class <class> bob/curl alice/web tcp/80
```

Deleting a rule resets, at once, the flows only that rule allowed.

## Certificates

Students' browsers must trust the relay's certificate. In order of preference:

1. a publicly trusted certificate for a name the school owns:
   `--tls acme --acme-domain class.example-school.fr` (ACME through
   [go-authn/servercert](https://github.com/go-authn/servercert)), or
   `--tls files --cert … --key …`;
2. the relay's internal CA (the default, `--tls internal`): install the root
   printed by `wctr-relay ca export` on the class's machines; it is
   name-constrained to the relay's names and addresses;
3. behind Caddy or another TLS proxy: `--tls off --listen 127.0.0.1:8080`
   (refused on any other address).

## Pages

- [Command reference]({{< relref "cli" >}})
- [The live flow map]({{< relref "flowmap" >}})
- [Security model]({{< relref "security" >}})
