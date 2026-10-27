---
title: "wctr-relay commands"
linkTitle: "Commands"
weight: 10
description: "Every wctr-relay command and flag, as of v0.1.0."
tags: [relay, cli]
---

`wctr-relay` is one binary: `serve` runs the relay; every other command is a
client of its control API. Locally they use the state directory's admin socket
(`0600`: its permissions are the credential); for a remote relay, `--server URL --token-file
FILE [--ca-file ROOT.pem]`, where URL is the relay's `--admin-listen` address
over https (the token is never a flag value). `--state-dir` (default `<user config dir>/wctr-relay`) applies to
all, and so does `--admin-socket PATH` (since v0.2.1): the admin socket's
path, `admin.sock` in the state directory by default. A Unix socket path holds
at most 103 bytes on macOS and 107 on Linux; a longer one is refused with that
reason, so a deep state directory needs a short `--admin-socket`, given to
`serve` and to the other commands alike.

| Command | What |
| --- | --- |
| `init [--name N] [--ip A]` | create the state directory (`0700`): admin token, session-token signing keys, and the internal CA for those names and addresses |
| `serve` | run the relay |
| `class create NAME` | a class; flags `--cidr` (default `10.42.0.0/16`), `--session-bits` (24), `--domain` (`class`), `--template isolated\|neutron`, `--allow sub:…\|email:…\|group:…`, `--require-identity` |
| `class list` | classes and their join codes |
| `class sessions CLASS` | sessions: label, name, subnet, connected |
| `class kick CLASS SESSION` | end a session: its connection, ports and tokens |
| `class rotate-code CLASS` | a new join code; the old one stops admitting |
| `class delete CLASS` | delete the class and what the relay knows of its students |
| `sg list --class C` | groups and their rules |
| `sg create NAME --class C [--stateless] [--description D]` | an empty group (it allows nothing) |
| `sg delete GROUP --class C` | a group no port uses |
| `rule add GROUP --class C --ingress\|--egress [--proto tcp\|udp\|icmp\|any] [--port 80\|8000-8080] [--remote-ip PREFIX\|--remote-group G]` | allow a flow |
| `rule delete RULE --class C` | remove a rule; the flows only it allowed are reset |
| `port list --class C [--student S]` | ports: name, address, groups |
| `port set PORT --class C [--sg a,b] [--port-security on\|off]` | a port's groups (replaces the list) |
| `flows --class C [--follow] [--deny-only] [--student S] [--backlog N]` | the flow log |
| `explain --class C SRC DST PROTO/PORT` | what the firewall would decide, and which rules |
| `ca export`, `ca fingerprint`, `ca rotate [--name] [--ip]` | the internal CA |
| `teacher-session --class C --out FILE` | a session token (written `0600`) for the teacher's own containers |
| `flowmap --class C --url URL` (the relay's https address) | the link to the class's [live flow map]({{< relref "flowmap" >}}), with a read-only 12-hour ticket in its fragment (v0.2.0) |
| `version` | the version |

A port is named by its id, its DNS name (`web.alice.class`) or
`student/container` (`alice/web`).

## `serve` flags

| Flag | Default | What |
| --- | --- | --- |
| `--listen` | `:8443` | the address students reach (data plane `/v1/data`, join, `/v1/info`) |
| `--tls` | `internal` | `internal`, `files`, `acme`, or `off` (loopback only) |
| `--cert`, `--key` | | PEM files for `--tls files`, re-read when they change |
| `--acme-domain`, `--acme-email`, `--acme-directory`, `--acme-cache` | | ACME for `--tls acme` (default CA: Let's Encrypt) |
| `--eab-kid`, `--eab-hmac-file` | | External Account Binding (the HMAC key in a `0600` file) |
| `--acme-http` | | also answer `http-01` on this address (e.g. `:80`) |
| `--cors-origin` | none | a browser origin (`scheme://host[:port]`) allowed besides the relay's own, e.g. the playground's (on GitHub Pages); repeatable, never `*`. See [Browsers on another origin](../security/#browsers-on-another-origin-cors) |
| `--host` | | Host names served; any other is refused |
| `--oidc-issuer`, `--oidc-audience`, `--oidc-type` | | OpenID Connect token verification |
| `--teacher-group` | | groups-claim value granting the teacher role on `--admin-listen` |
| `--admin-listen` | | also serve the admin API on this address (TLS) |
| `--session-ttl` | `12h` | session token lifetime |
| `--audit-retention` | `720h` | how long the audit log keeps events |
| `--relay-id` | host name | this relay's id |
