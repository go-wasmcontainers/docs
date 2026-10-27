---
title: "Relay security model"
linkTitle: "Security"
weight: 20
description: "Who may do what on a relay, what it refuses, and what it keeps about students."
tags: [relay, security]
---

## Principals

| Who | Credential | Where | May |
| --- | --- | --- | --- |
| Teacher, on the relay's machine | the admin socket (`0600` in a `0700` state directory) | Unix socket | everything |
| Teacher, remote | the admin token (`Authorization: Bearer`) or an OpenID Connect token carrying `--teacher-group` | `--admin-listen` only, over TLS | everything |
| Student | a session token from `Join` (join code, plus an OpenID Connect identity when the class requires one) | `--listen` | their own ports and flow decisions; read the groups |

The public listener never grants the teacher role, whatever arrives on it
(the admin token included).

## What the relay refuses

- **Identity is the relay's**: it assigns every address and port; a frame that
  names a port the session does not own is refused, a datagram from a source
  port the sender has not bound is dropped.
- **Authentication before the upgrade**: the session token is a WebSocket
  subprotocol (`bearer.<token>`), checked before the upgrade and before any
  per-session state; it never appears in a URL. An `Origin` that is neither
  the relay's own nor in `--cors-origin`, and an unknown `Host`, are refused.
- **Pre-authentication limits** per source address on joins and upgrades, a cap
  on concurrent handshakes, header timeouts and sizes.
- **Bounds**: 16 KiB frames, 64 KiB stream windows, and per session: 256
  streams, 64 listeners, 10 ports, 32 pending connects, 2 MiB/s and 2 000
  frames/s; a connection whose send queue passes 4 MiB is closed. Per class:
  50 security groups and 500 rules (Neutron's quotas).
- A frame that does not decode, data beyond the window, or a frame out of
  place closes the connection.
- **Tokens**: Ed25519, one fixed layout and one algorithm, checked against the
  relay's clock; a kicked session's tokens stop working at once.

## Browsers on another origin (CORS)

The playground runs on GitHub Pages and calls `Join` and `RefreshSession` on
the relay from the browser. That is a cross-origin request, so the browser
first sends a preflight, and the relay answers it only for the origins it is
told to trust:

```sh
wctr-relay serve --cors-origin https://go-wasmcontainers.github.io
```

- **An allowlist, empty by default.** The relay's own origin is always
  allowed. Add a page's origin once per origin (`scheme://host[:port]`, the
  flag can be repeated). `*` and wildcards are refused. Without the flag, a
  page served from elsewhere cannot join this relay.
- **Only the calls a page needs**: `Join`, `RefreshSession`, `WatchFlowMap`,
  `WatchFlows` and `ListPorts`. Other public calls answer no preflight, and
  the admin listener has no CORS at all.
- **Exactly the headers Connect needs.** The method and header lists are
  those of [connectrpc.com/cors](https://pkg.go.dev/connectrpc.com/cors),
  plus `Authorization`, `Wctr-Session`, `Wctr-Map-Ticket` and `User-Agent`.
  Firefox lists `User-Agent` in the preflight because Connect-Web sets it.
  Preflights are cached for 2 hours.
- **No credentials mode.** Tokens travel in headers and never in cookies, so
  the browser has nothing to attach on its own, and the relay never sends
  `Access-Control-Allow-Credentials`.
- **The data WebSocket checks the same list.** Browsers do not preflight
  WebSockets, so a hostile page could otherwise open the data channel with a
  token it had obtained (cross-site WebSocket hijacking). The relay therefore
  refuses an upgrade whose `Origin` is not on the list. A client that is not
  a browser sends no `Origin` and is unaffected.

The `--cors-origin` flag appeared in relay v0.3.0. It replaces `--origin`,
which covered only the WebSocket and allowed the Pages host by default.

## Secrets

The admin token, the signing keys, the CA key, an EAB key and the state file
are `0600` files of the relay's user, refused if anyone else can read them,
never on a command line, in an environment variable or in a URL. The internal
CA is name-constrained: even its key cannot sign for another site.

## What the relay keeps about students

Per session: the display name typed, a derived DNS label, and — with OpenID
Connect — a pseudonymous hash of issuer and subject (never the token, the
e-mail address or the groups, which are checked at join time and dropped).
Flow logs hold addresses and ports, not contents. Deleting a class deletes all
of it; the audit log (logins, refusals, kicks, rule changes) keeps 30 days by
default. The teacher's institution is the controller under the GDPR.

## Tested

Every point above has a test in the repository; the decoders of untrusted
input (frames, tokens, join codes, labels, rule specifications, and the relay
fed arbitrary frames) are fuzzed in CI, and `govulncheck` runs on every change.
