---
title: "Classroom network"
weight: 10
description: "How students' browsers reach each other's containers through a relay — on the teacher's laptop or on the internet — that is also a firewall, with rules modelled on OpenStack Neutron security groups, a gRPC control API, OpenID Connect sign-in and a high-availability plan."
tags: [design, networking, security]
---

{{< callout type="info" >}}
**Status: M-relay-1 and M-relay-2 shipped in relay v0.1.0** (see
[Relay]({{< relref "relay" >}})); the rest is planned. The implementation lives
in [go-wasmcontainers/relay](https://github.com/go-wasmcontainers/relay). This
page decides what is built and records why, so the code can be reviewed against
it. Milestones are named `M-relay-1` and up.
{{< /callout >}}

## The problem

A teacher has a laptop on the classroom LAN, possibly with no internet access.
Ten to forty students each open the playground in their own browser (Chrome,
Firefox or Safari, sometimes on a school-managed machine). Each browser runs
WASI containers in Web Workers. Inside one browser, containers already share a
virtual network. The lab now needs **student A's container to reach student
B's container**, for example a web server on one side and `curl` on the other,
and the teacher needs to **decide and show which flows are allowed**, the way a
cloud's security groups do.

The answer on this page: every browser opens **one secure WebSocket** to a
**relay** that runs on the teacher's laptop. The relay brokers socket-level
(L4) flows between containers, and it is also a **firewall**: every flow is
checked against **security groups** whose model is OpenStack Neutron's. The
teacher manages them through a **gRPC control API**. The brief called it the
"c-api"; this page reads that as **control API** (the control plane, as
opposed to the data plane that carries the containers' bytes).

## Decisions

| # | Question | Decision |
| --- | --- | --- |
| 1 | Browser to relay transport | **WebSocket over TLS (`wss`)**, one per browser, multiplexing every container's flows. WebTransport is the second transport, added later. WebRTC is not used. |
| 2 | Where the page comes from | **Both paths are first-class**: the relay serves the playground itself over HTTPS on the LAN, and the page on GitHub Pages connects to the relay. Whether a school can install a CA is not known in advance, so the join flow **detects** which works (see [Join-flow detection](#join-flow-detection)). |
| 2b | Where the TLS certificate comes from | In this order: **(1) a publicly trusted certificate from ACME** through [go-authn/servercert](https://github.com/go-authn/servercert) (a school-owned name such as `class.<school-domain>` pointing at the relay; DNS-01 for a LAN-only relay is to be added to servercert); **(2) the relay's internal CA**, whose root a school installs; **(3) WebTransport certificate hashes** from the GitHub Pages page. Caddy in front of the relay is the documented alternative. |
| 3 | Network model | **L4 flows**: the relay brokers `connect`/`listen`/datagrams between container endpoints, each with a virtual IP and a DNS name. No packet-level (L3) network stack in the engine. |
| 4 | Firewall model | Neutron's **networks, ports, security groups and rules**, allow-only, stateful by default, enforced at both ends of every flow, with flow logs. |
| 5 | Control API | **gRPC services**, served with Connect so the same handler answers gRPC, gRPC-Web and Connect; the teacher's CLI is a **separate cobra binary, `wctr-relay`**. |
| 6 | Identity | The relay assigns every address; nothing a browser sends about its own identity is trusted. Teacher: an admin token. Students: a class code exchanged for a short-lived session token. |
| 7 | Internet mode | The same relay can run on the internet for remote or hybrid students, with **OpenID Connect** sign-in verified by [go-authn/oidc](https://github.com/go-authn/oidc), logins through [go-authn/rp](https://github.com/go-authn/rp), and [go-authn/bridge](https://github.com/go-authn/bridge) as the identity provider for SAML-only universities. The LAN mode keeps working with a class code and no identity provider. |
| 8 | High availability | The **control plane** (classes, groups, rules, addresses, signing keys, audit) is replicated with Raft across relays; the **data plane** (live flows, connection tracking) is not. One relay holds a class at a time, under a lease with an increasing **epoch**; clients follow an ordered list of relays and refuse an older epoch. |
| 9 | Build or adopt | **Hybrid**: our relay stays the classroom data plane and firewall (relay-side enforcement and flow logs are the lesson); go-authn supplies identity and certificates; the client failover follows Tailscale's DERP pattern. See [Alternatives considered](#alternatives-considered). |
| 10 | Seeing the network | The relay serves a **live flow map**: a go-widgets page (Go `js/wasm` on a canvas) fed by an aggregated, coalesced server stream over the Connect protocol; denied flows flash red with the refusing decision; teacher view of the class, per-student view of one's own flows. See [The live flow map](#the-live-flow-map). |

Answers to the questions this page first left open (decided 2026-10-26):

| Question | Answer |
| --- | --- |
| Default class network | `10.42.0.0/16`, configurable per class with `wctr-relay class create --cidr` |
| How a new class starts | **isolated**: no ingress, egress to the teacher's subnet only. Opening flows is part of the lab. Neutron's default group is a template the teacher can choose. |
| Can schools install a classroom CA? | **Unknown per school**, so both TLS paths stay first-class and the join flow detects which works |
| IPv6 | later; the model carries `ethertype`, the first milestones allocate IPv4 only |
| The teacher's containers | they join as a session (subnet 0, label `teacher`), so the teacher can publish shared services |

The docs repository has no ADR convention, so these tables are the decision record.

## Topology

```mermaid
flowchart LR
  subgraph A["Student A's browser"]
    A1["container web<br/>10.42.1.2"]
    A2["container db<br/>10.42.1.3"]
    AE["engine<br/>(relay network driver)"]
    A1 --- AE
    A2 --- AE
  end
  subgraph B["Student B's browser"]
    B1["container curl<br/>10.42.2.2"]
    BE["engine<br/>(relay network driver)"]
    B1 --- BE
  end
  subgraph T["Teacher's laptop"]
    R["wctr-relay<br/>relay + firewall + DNS"]
    API["control API<br/>(gRPC / gRPC-Web / Connect)"]
    CLI["wctr-relay CLI<br/>teacher dashboard"]
    R --- API
    CLI --> API
  end
  AE <-- "wss: one connection,<br/>multiplexed flows" --> R
  BE <-- "wss" --> R
```

Traffic between two containers of the **same** browser does not leave it; the
engine checks it against the same rules (see [Where rules are
enforced](#where-rules-are-enforced)). Traffic between **different** browsers
always crosses the relay: a star, so the firewall sees every flow.

## 1. Transport between the browser and the relay

Three candidates were compared.

| | WebSocket (`wss`) | WebTransport | WebRTC data channels |
| --- | --- | --- | --- |
| Support | every browser the labs target | Chrome 97, Firefox 114, Safari 26.4 (iOS 26.4) per MDN's compatibility data, and `serverCertificateHashes` from Chrome 100, Firefox 125, Safari 26.4; MDN marks it Baseline 2026 ("since March 2026") | Baseline widely available since January 2020 (MDN) |
| In a Web Worker | yes | yes (MDN: the constructor is available in workers) | not established by MDN (an `RTCDataChannel` is transferable, which is not the same) |
| Certificate without a public CA | must be trusted by the browser | `serverCertificateHashes`: no CA needed, but strict constraints (below) | DTLS fingerprints from signalling |
| Multiplexing | ours, in the frames | native streams and datagrams (QUIC), no head-of-line blocking between streams | native channels |
| Firewall sees the traffic | yes, it terminates at the relay | yes | only if forced through the relay, which removes the reason to use it |

**`serverCertificateHashes`** lets a page connect to a server whose certificate
no CA signed, by giving the SHA-256 of the certificate. The W3C WebTransport
Candidate Recommendation (30 July 2026) and MDN set the constraints:

- the certificate **must be X.509v3**, its total validity period **must not
  exceed two weeks**, and the current time must be inside it;
- the accepted key algorithms are implementation-defined but **must include
  ECDSA P-256 (secp256r1)** and **must not include RSA**;
- the hash is **`sha-256`**, over the DER encoding of the leaf certificate;
- the option only works on a dedicated connection (`allowPooling: false`).

The relay can meet these: it generates an ECDSA P-256 certificate valid for at
most 13 days (a margin under 14) when a class starts, and hands its hash to
students in the join link. A class that lasts longer gets a new certificate
and a new link.

**WebRTC is rejected.** Its value is a direct peer-to-peer path; the firewall
must see every flow, so the topology would be a star through the relay anyway,
and WebRTC would only add ICE, DTLS and SCTP to a connection the relay
terminates. It is also not established that it can be used from the Web Worker
that runs the engine.

**Decision.** WebSocket is the first transport because it works in every
target browser, in a worker, and through the simplest server code. It carries
everything over one TCP connection, so one lost Wi-Fi frame stalls every flow
of that browser (head-of-line blocking); WebTransport removes that and is the
second transport (M-relay-4), selected per browser by feature detection, with
the same framing on its streams.

## 2. Reaching a LAN host from a web page

A page that talks to the teacher's laptop runs into four browser rules.

1. **Mixed content.** An `https` page cannot open `ws://`; MDN classes any
   non-upgradable insecure request as blockable, and the Local Network Access
   draft states that WebSocket has no `RequestInit`, so `ws://` URLs cannot
   use its mixed-content exemption either. The relay must speak **`wss`**, so it
   needs a certificate the browser accepts.
2. **Local Network Access** (formerly Private Network Access). The WICG draft
   (7 August 2026) classifies addresses as *loopback*, *local* (RFC 1918,
   link-local, `fc00::/7`) and *public*; a request from a document to a **less
   public** address space needs the `local-network` permission, only from a
   secure context. The WebSocket handshake is a fetch, so it is covered;
   WebTransport inherits the rule through fetch. Chrome's announcement says the
   permission **prompt launches in Chrome 142**, and in June 2025 it still
   listed WebSocket, WebTransport and WebRTC as not yet gated. The design
   assumes they are gated: a page on GitHub Pages (public) that opens a socket
   to `192.168.x.y` (local) triggers the prompt, and a school's enterprise
   policy may pre-deny it. A page served **from the relay itself** is in the
   local address space, so its connections back to the relay are not "less
   public" and need no permission.
3. **Cross-origin isolation.** The executor uses `SharedArrayBuffer`, which MDN
   allows only in a **secure context** that is **cross-origin isolated**. On
   GitHub Pages, which cannot set response headers, the playground gets this
   from coi-serviceworker; a relay serving the page sets
   `Cross-Origin-Opener-Policy` and `Cross-Origin-Embedder-Policy` itself.
4. **Certificate trust.** A WebSocket has no certificate-error interstitial
   to click through: either the browser already trusts the certificate, or the
   connection fails.

### Two deployments

| | (a) Page on GitHub Pages, relay on the LAN | (b) Relay serves the page over HTTPS |
| --- | --- | --- |
| Works without internet | no (the page and its assets come from GitHub Pages) | **yes** |
| Local Network Access prompt | **yes**, on every student browser; may be blocked by policy | **no** (same origin, same address space) |
| Cross-origin isolation | coi-serviceworker | response headers, no service worker |
| Certificate | WebTransport hashes (no trust needed), or a trusted certificate for `wss` | a certificate every student browser trusts |
| Version skew | page and relay released separately | the relay embeds one playground release: no skew |

**Both are first-class.** Whether a school can install a CA on its machines is
not known in advance, so neither deployment is a fallback for the other.

### Where the certificate comes from, in order of preference

1. **A publicly trusted certificate from ACME.** A school-owned name (say
   `class.example-school.fr`) resolves to the relay's address — a private LAN
   address is fine — and a public CA issues its certificate: no student
   installs anything. Caddy's automatic-HTTPS documentation states the key
   point for a LAN-only relay: with the DNS challenge, "the server requesting a
   certificate does not need to be externally accessible"; it needs credentials
   for the DNS provider. The DNS-01 challenge (RFC 8555 §8.4) proves control of
   the name with a TXT record at `_acme-challenge.<name>` whose value is the
   base64url SHA-256 of the key authorization. The relay takes its certificate
   from [go-authn/servercert](https://github.com/go-authn/servercert), the
   fleet's shared source of TLS server certificates: files reloaded on change,
   or ACME through autocert with External Account Binding. servercert answers
   `tls-alpn-01` and `http-01` only ("No `dns-01`"), which covers a relay on
   the internet; it also covers a LAN-only relay whose organisation has a
   **GÉANT TCS / HARICA** account with pre-validated domains, which issues "with
   no ACME challenge at all". For every other LAN-only relay, **DNS-01 is added
   to servercert** (shared code, not a copy in the relay). An IP-address
   certificate is not an option for a LAN relay: Let's Encrypt issues them
   (generally available since 15 January 2026, short-lived profile only, 160
   hours), but servercert refuses IP names, and whether a public CA would
   validate a private LAN address was not established here; a name is the
   path.
2. **The relay's internal CA**, when no name or no internet at issuance is
   available. `wctr-relay init` creates a root (ECDSA P-256), **name-constrained**
   to the classroom's names and addresses, which a school installs on managed
   machines by the same policy channel that installs any CA. mkcert's README
   states the risk of any such CA: its key "gives complete power to intercept
   secure requests" from a machine that trusts it — hence the name
   constraints, a `0600` key that is never exported, and `wctr-relay ca rotate`.
   Caddy's internal issuer is the same idea (a local CA it tries to add to the
   system trust store, which "isn't guaranteed to work" without privileges),
   but it lives in Caddy's PKI app, not in CertMagic (which has no internal
   issuer), so embedding it would mean embedding Caddy.
3. **WebTransport certificate hashes** from the GitHub Pages page (deployment
   (a)): no trust needed, under the constraints of
   [§1](#1-transport-between-the-browser-and-the-relay).

**Caddy as a front** is the documented alternative for a teacher who already
runs it: `wctr-relay serve --tls off --listen 127.0.0.1:8080` (the relay
refuses `--tls off` on any non-loopback address) behind Caddy's
`reverse_proxy`, with Caddy's own ACME or `tls internal` and `caddy trust`.
The relay still checks `Origin` and `Host` itself. If on-demand TLS is used it
"must be both enabled and restricted" (Caddy's words), with an `ask` endpoint.

**Secrets**: a DNS provider's API token, an EAB HMAC key and the CA key are
read from **files** (`0600`, owned by the relay's user), never from a command
line, an environment variable or a URL.

### Join-flow detection

The join link (printed and shown as a QR code by `wctr-relay class create`)
carries, in its URL **fragment** (never sent to a server): the relay's LAN URL,
its GitHub Pages fallback, the join code, and the WebTransport certificate hash
when there is one. The page:

1. fetches `https://<relay>/v1/info` (a small public JSON document: version,
   relay id, TLS mode) with a 3 s timeout; if it answers, the certificate is
   trusted (public ACME or an installed internal CA) and the browser is sent to
   the playground **served by the relay** (deployment (b));
2. otherwise, if the page is on GitHub Pages and the browser has WebTransport
   with `serverCertificateHashes`, it connects with the hash (deployment (a));
3. otherwise it tells the student exactly what is missing (install the CA,
   accept the Local Network Access prompt, or use another browser), rather than
   failing silently.

Whether a page loaded after a certificate warning is still a secure context
in each browser is **not established** by the references opened here; M-relay-3
measures it in the three browsers before any path relies on a click-through.

## 3. Network model: L4 flows, not L3 packets

The containers' sockets use the WasmEdge socket extension in the module
`wasi_snapshot_preview1`, as `github.com/stealthrocket/net` imports it:
`sock_open`, `sock_bind`, `sock_listen`, `sock_connect`, `sock_recv_from`,
`sock_send_to`, `sock_getaddrinfo`, `sock_shutdown`, plus `sock_getsockopt`,
`sock_setsockopt`, `sock_getlocaladdr` and `sock_getpeeraddr`. Its socket
types are **`SOCK_STREAM` and `SOCK_DGRAM` only**; there is no raw socket. The
engine therefore sees *a container connecting to 10.42.2.2:80*, never a packet.

The alternative is what container2wasm does: an L3 network stack based on
gVisor's netstack (`c2w-net`, built on gvisor-tap-vsock) that carries Ethernet
frames, forwarded over WebSocket in its `delegate=ws://...` mode.

| | L4 flows (chosen) | L3 packets (gVisor netstack) |
| --- | --- | --- |
| Fits the socket ABI | directly: one host call is one flow operation | needs a TCP/IP stack between the ABI and the wire |
| Firewall | sees connect, listen, port, peer: what a rule names | must reassemble flows from packets to say the same |
| Cost | one frame per write | TCP inside the container's stack, carried over the relay's TCP: TCP over TCP, retransmission on two levels, per-packet framing |
| Size in the browser | none beyond the driver | a whole network stack compiled into the engine |
| UDP | datagram frames | native |
| ICMP / ping | not expressible by the ABI (see below) | possible, but gvisor-tap-vsock says ICMP is not forwarded outside its network |

**Decision: L4.** Addressing is still IP-shaped, because the lab teaches it:

- one **network** per class, default `10.42.0.0/16`; student *n* gets
  `10.42.n.0/24`, the teacher `10.42.0.0/24`; the relay allocates every
  address (as Neutron allocates `fixed_ips`), the browser never chooses one;
- DNS: `<container>.<student>.class` resolves to the container's address,
  through `sock_getaddrinfo`, answered by the relay; a container's bare name
  still resolves inside its own browser;
- **TCP**: `listen` registers the address and port at the relay; `connect` is a
  request to the relay, which checks the rules and opens a stream to the
  listener's browser, or refuses (the caller gets `ECONNREFUSED`, or a timeout
  if the teacher chose "drop" for teaching);
- **UDP**: each `sock_send_to` is one datagram frame with a source and
  destination, at most 8 KiB; the relay checks it and forwards or drops it.

**Ping.** The ABI cannot send ICMP, so ping is taught through a small extension
the engine provides later (M-relay-5): an import module `wctr_net` with one
function, `icmp_echo(addr, seq, payload)`, which the driver turns into an echo
frame. The relay answers it on behalf of the destination port if its rules
allow ICMP echo ingress (`protocol: icmp`, `port_range_min: 8`, type 8). A
`ping` image built against it gives the expected pedagogy: allowed, the
student sees replies; removed, the pings time out, and the flow log shows the
drop.

## 4. Firewall model, mapped from Neutron

The Neutron API reference defines a **security group rule** by `direction`
(ingress or egress), `ethertype` (IPv4 or IPv6), `protocol` (`tcp`, `udp`,
`icmp`, ..., or `any`), `port_range_min` and `port_range_max` (for ICMP: the
ICMP type and code), and one of `remote_ip_prefix`, `remote_group_id` or
`remote_address_group_id`. A **security group** has a `stateful` flag: a
stateless group "bypasses connection tracking", so both directions of a flow
need rules. Nova's guide states the overall policy: "Security groups deny
everything by default, so rules indicate what is allowed", and adding a rule
is additive. A **port** carries `security_groups`, `fixed_ips`,
`port_security_enabled` ("security group rules and anti-spoofing rules are
applied") and `allowed_address_pairs`. Neutron creates, for each project's
`default` group, four rules (all IPv4/IPv6 egress, and ingress from ports in the
same group), and for other new groups only the two egress rules; a template of
those rules is configurable (`used_in_default_sg`, `used_in_non_default_sg`,
`remote_group_id: PARENT`). Quotas count `security_group` and
`security_group_rule` per project.

### Our resources

| Resource | Fields | Neutron counterpart |
| --- | --- | --- |
| `Class` | id, name, join code, state, quotas | project |
| `Network` | id, class, `cidr`, `dns_domain` (`class`), per-student prefix length | network + subnet |
| `Session` | id, student display name, label (`alice`), prefix, state, last seen | (none: a user) |
| `Port` | id, network, session, container name, `fixed_ip`, `security_groups`, `port_security_enabled` | port |
| `SecurityGroup` | id, class, name, description, `stateful` | security group |
| `SecurityGroupRule` | id, group, `direction`, `ethertype`, `protocol`, `port_range_min`, `port_range_max`, `remote_ip_prefix` or `remote_group_id`, description, `log` | security group rule |
| `DefaultRule` | the same fields, plus `used_in_default_sg`, `used_in_non_default_sg` | default security group rule |

A port is created when a container attaches to the class network and is
deleted with it. Only the teacher can set `port_security_enabled: false`.
There is no `allowed_address_pairs`: a container has exactly one address,
which the relay assigned.

### Policy and evaluation

- **Default deny.** A flow is allowed only if a rule allows it.
- **Both ends.** A flow from port S to port D is allowed iff **S's egress rules**
  allow it **and D's ingress rules** allow it, as Neutron filters at each port.
- **Order-free.** Rules only allow, groups on a port are a union, so there is no
  order to get wrong: the result is the same whatever order rules were added
  in. The decision names the rule that allowed it, or "no rule" if none did.
- **Default groups.** Each class gets a `default` group with Neutron's four
  rules (egress allowed, ingress from the same group), and the teacher can
  switch the class template to **isolated** (egress only to the teacher's
  subnet, no ingress), which is the better start for a lesson on opening ports.
- **`remote_group_id`** matches the current addresses of every port in that
  group, so a rule "ingress tcp 5432 from group `web`" follows containers as
  students start and stop them.

### Statefulness

Statefulness is **connection tracking of L4 flows**, which the relay has for
free because it brokers every flow:

- **TCP (stateful group):** the rules are checked once, at `connect`; the stream
  then carries both directions until either side closes.
- **UDP (stateful group):** an allowed datagram creates a conntrack entry keyed
  by the 5-tuple; replies match it; it expires after 30 s without traffic.
- **Stateless group:** for TCP, the reverse direction (D's egress, S's ingress)
  must be allowed too, or the connect is refused; for UDP, every datagram is
  checked both ways. This is Neutron's "two rules for a TCP flow", visible to
  students.
- **Rule changes:** after any change, the relay re-evaluates open flows and
  **resets** the ones no longer allowed, logging them as `revoked`. A firewall
  that silently keeps old connections confuses a lesson.

### Where rules are enforced

Between browsers, at the **relay**: it is authoritative. Inside one browser,
the **engine** applies the same rules, pushed by the relay with the session, so
that a lesson works the same with two containers of one student. That copy is
advisory: a student who modifies their own page only affects their own
containers, which they control anyway.

### Flow logs

Every decision can be logged: time, class, source and destination session,
port, address and port, protocol, verdict (`allow`, `deny`, `revoked`), the
matching rule or `no rule`, and at close the bytes each way and duration.
Rules carry a `log` flag; denies are always logged. The teacher streams logs
live (CLI, dashboard); a student sees the decisions about their own ports,
which is the point of the exercise ("why can't I reach Bob's server?").

### The live flow map

The relay serves a page that draws the class's network **as it happens**:
nodes are sessions (the students and the teacher) holding their containers,
coloured by security group; edges are live flows, their width following
throughput; **denied** flows flash red with the rule decision that refused
them ("no ingress rule of bob/web allows tcp/80"), because seeing the refusal
is the lesson.

- **Data.** The page calls a server-streaming RPC,
  `FlowLogService.WatchFlowMap`, over the Connect protocol (the reason
  connect-go was chosen: "the other RPC types also support HTTP/1.1", server
  streaming included, so a browser needs no proxy). The relay **aggregates**:
  every tick (default 4 Hz) it sends one update holding the per-edge counters
  (src port → dst port, protocol and port: bytes each way, flows opened, open,
  denied) and the discrete events since the last tick (opened, closed, denied,
  revoked, with the rule ids). A slow page gets **coalesced** updates — the
  newest state replaces the unsent one — never an unbounded queue. A view is
  capped (400 nodes, 2 000 edges; past that the update says what was left out).
- **View.** A **stable, incremental layout**, not a force simulation (no full
  relayout per tick): sessions are boxes on
  a grid ordered by subnet (teacher first), containers inside them, so a node
  does not move when traffic changes and 40 students × several containers stay
  readable on a projector. Click a node or an edge for its details (groups,
  the rules that matched, conntrack state, counters). Filters by student,
  group and port. Light and dark themes. Pause and replay of the last minutes
  are **deferred** (the ring of the last 1 000 events is already kept).
- **Technique (decided 2026-10-26): the fleet's own toolkit.** The page is a
  Go `js/wasm` application built on
  [go-widgets/toolkit](https://github.com/go-widgets/toolkit) and run in a
  `<canvas>` by [go-widgets/webcanvas](https://github.com/go-widgets/webcanvas),
  which blits "an RGBA framebuffer into the canvas and routes DOM pointer and
  keyboard events back into the scene". The widgets a flow map needs and the
  toolkit lacks — a node-link **graph view** with groups, animated edges and
  incremental layout, a legend, a filter bar, a detail panel, a sparkline —
  are added **to go-widgets**, generic and reusable, under its conventions
  (MVVM, Theme, keyboard navigation, its coverage gate), each in its own pull
  request and minor release; the relay imports them. Everything is served by
  the relay (same origin, the `.wasm` included), under a strict
  Content-Security-Policy: `default-src 'self'; script-src 'self'
  'wasm-unsafe-eval'` (the narrowest allowance WebAssembly compilation needs),
  no inline script, no third-party request — the relay runs without internet.
  The join QR code is drawn from [go-gfx/qr](https://github.com/go-gfx/qr)'s
  module matrix. Because a canvas has no DOM to assert on, the page's tests
  read the **view model** (MVVM's testable layer), which the page exposes to
  the test harness.
- **Access.** The teacher's view of the whole class needs the admin role (the
  admin token on loopback or the Unix socket, or the OpenID Connect teacher
  role in internet mode). A student may open **their own** view: only the
  flows that touch their own containers, with the other end named but no flow
  between two other students — the privacy between students the class needs.
- **Tests.** Playwright (Chromium and Firefox) drives real flows through an
  in-process relay and asserts what the page's view model says (edges present,
  a denied flash with its rule, counters moving), not screenshots; the update
  latency from a flow to the screen, the frame time at about 200 nodes and
  1 000 edges on one core, and the relay's CPU at 40 sessions are measured.

### Quotas

Per class, as Neutron's: `security_group` (default 50), `security_group_rule`
(default 500), and `port` (default 40 × 10). Per session, for safety rather
than pedagogy: see the [security audit](#6-security-audit).

## 5. Control API

### Services

The control API is gRPC, described in one `.proto`; the relay serves it with
connect-go, whose handlers support gRPC, gRPC-Web and the Connect protocol and
mount on a plain `net/http` server. The Connect protocol uses no HTTP trailers
and supports server streaming over HTTP/1.1 (bidirectional streaming needs
HTTP/2), so a **browser dashboard** can call the same API, including the live
flow log, without a proxy. Nothing in the API needs bidirectional streaming.

```protobuf
syntax = "proto3";
package wasmcontainers.relay.v1;

service ClassService {
  rpc CreateClass(CreateClassRequest) returns (Class);       // teacher
  rpc GetClass(GetClassRequest) returns (Class);
  rpc RotateJoinCode(RotateJoinCodeRequest) returns (Class); // teacher
  rpc Join(JoinRequest) returns (JoinResponse);              // student: code -> session token
  rpc ListSessions(ListSessionsRequest) returns (ListSessionsResponse);
  rpc KickSession(KickSessionRequest) returns (Empty);       // teacher
}

service NetworkService {
  rpc GetNetwork(GetNetworkRequest) returns (Network);
  rpc ListPorts(ListPortsRequest) returns (ListPortsResponse); // student: own ports only
  rpc UpdatePort(UpdatePortRequest) returns (Port);            // security_groups, port_security_enabled
}

service SecurityGroupService {
  rpc ListSecurityGroups(ListSecurityGroupsRequest) returns (ListSecurityGroupsResponse);
  rpc CreateSecurityGroup(CreateSecurityGroupRequest) returns (SecurityGroup);
  rpc UpdateSecurityGroup(UpdateSecurityGroupRequest) returns (SecurityGroup);
  rpc DeleteSecurityGroup(DeleteSecurityGroupRequest) returns (Empty);
  rpc CreateRule(CreateRuleRequest) returns (SecurityGroupRule);
  rpc DeleteRule(DeleteRuleRequest) returns (Empty);
  rpc ListDefaultRules(ListDefaultRulesRequest) returns (ListDefaultRulesResponse);
  // Dry run: which rule, if any, would allow this flow, and at which end.
  rpc ExplainFlow(ExplainFlowRequest) returns (FlowDecision);
}

service FlowLogService {
  rpc WatchFlows(WatchFlowsRequest) returns (stream FlowEvent); // server streaming
}

message SecurityGroupRule {
  string id = 1;
  string security_group_id = 2;
  Direction direction = 3;            // INGRESS, EGRESS
  EtherType ethertype = 4;            // IPV4, IPV6
  Protocol protocol = 5;              // ANY, TCP, UDP, ICMP
  optional uint32 port_range_min = 6; // ICMP: type
  optional uint32 port_range_max = 7; // ICMP: code
  oneof remote {
    string remote_ip_prefix = 8;
    string remote_group_id = 9;
  }
  string description = 10;
  bool log = 11;
}

message FlowEvent {
  google.protobuf.Timestamp time = 1;
  string src_session = 2;  string src_port_id = 3;  string src = 4; // "10.42.1.2:51234"
  string dst_session = 5;  string dst_port_id = 6;  string dst = 7;
  Protocol protocol = 8;
  Verdict verdict = 9;     // ALLOW, DENY, REVOKED, CLOSED
  string rule_id = 10;     // empty: no rule matched
  uint64 bytes_out = 11;  uint64 bytes_in = 12;
}
```

The **data plane** is not gRPC: it is the binary framing on the WebSocket
(below), because it carries every byte the containers exchange and its cost
per frame matters.

### Authentication and authorization

| Principal | Credential | Can |
| --- | --- | --- |
| Teacher (local) | the Unix socket, mode `0600`, or the admin token over loopback | everything |
| Teacher (dashboard on another machine) | the admin token, as `Authorization: Bearer`, over TLS; opt-in with `--admin-listen` | everything |
| Student | a session token from `Join` | own ports, own flow decisions, read the groups that apply to them |

- The **admin token** is 256 random bits generated by `wctr-relay init`, stored
  `0600` under `os.UserConfigDir()`, compared in constant time, never printed
  except by an explicit `wctr-relay token show`, never in a URL.
- The **class code** (10 characters, from an alphabet without look-alikes) is
  shown on the teacher's screen and in a QR code. `Join` exchanges it with a
  display name for a **session token**: signed by the relay (Ed25519), bound to
  the class and the session id, valid 12 hours, revoked by `KickSession` or by
  rotating the code.
- **mTLS is not used for students**: provisioning client certificates into 40
  browsers is the problem the class code solves. A teacher may require mTLS on
  `--admin-listen`.
- Authorization is checked in the handler from the token's claims, never from a
  field of the request (a student cannot pass another session's id).

### The teacher's command line

A **separate binary, `wctr-relay`**, in its own repository
([go-wasmcontainers/relay](https://github.com/go-wasmcontainers/relay)), built with cobra like the rest of
the fleet. It is the relay daemon and its client, as `wctrd` and `wctr` are for
the engine, but in one binary because a teacher installs one thing. It stays out
of `wctr`, whose surface is compatible with the Docker Engine API and is what
students learn.

```text
wctr-relay init                          # classroom CA, admin token, state directory
wctr-relay serve [--listen :8443] [--admin-listen 127.0.0.1:8444]
wctr-relay class create "Networking 101" # prints the join link and QR code
wctr-relay class sessions | kick <student>
wctr-relay sg list | create <name> [--stateless] | delete <name>
wctr-relay rule add <sg> --ingress --proto tcp --port 80 --remote-group web
wctr-relay rule del <rule-id>
wctr-relay port list [--student alice] | set <port> --sg web,db
wctr-relay flows [--follow] [--deny-only] [--student alice]
wctr-relay explain alice/web bob/curl tcp/80
wctr-relay ca export | ca rotate
```

## Data plane framing

One WebSocket per browser, binary messages, one frame per message:

```text
 0      1      2              6              10
 +------+------+--------------+--------------+----------------+
 | type | flags| stream id    | length       | payload ...    |
 | u8   | u8   | u32 BE       | u32 BE       | <= 16 KiB      |
 +------+------+--------------+--------------+----------------+
```

Frame types (version 1, numbered in the protocol): `HELLO`, `WELCOME`,
`ATTACH`, `ATTACHED`, `DETACH`, `LISTEN`, `LISTENING`, `UNLISTEN`, `CONNECT`,
`ACCEPT`, `REFUSE`, `DATA`, `WINDOW`, `CLOSE`, `RESET`, `DATAGRAM`, `RESOLVE`,
`RESOLVED`, `RESULT`, `REAUTH`, `GOAWAY`; `ECHO` comes with ICMP in M-relay-5.
The client allocates odd stream ids and the relay even ones. The session token
is checked **before** the WebSocket upgrade: a browser cannot set an
`Authorization` header on a WebSocket, so the token travels as a second
subprotocol (`Sec-WebSocket-Protocol: wctr-relay.v1, bearer.<token>`) — a
header, never a URL. `REAUTH` hands the relay a fresh token on the open
connection. Control payloads are protobuf messages from the same `.proto`;
`DATA` payloads are raw bytes. Flow control is credit-based per stream (64 KiB initial window,
`WINDOW` frames return credit) and per connection (1 MiB), so one fast sender
cannot fill the relay's memory for a slow reader. The relay rewrites every
source address: a frame names a stream or a port the session owns, never an
address the browser chose.

```mermaid
sequenceDiagram
  participant A as alice/curl (engine A)
  participant R as wctr-relay
  participant B as bob/web (engine B)
  B->>R: LISTEN port bob/web tcp/80
  R-->>B: OK (10.42.2.2:80)
  A->>R: RESOLVE web.bob.class
  R-->>A: 10.42.2.2
  A->>R: CONNECT stream 7, port alice/curl, 10.42.2.2:80
  Note over R: egress(alice/curl) and ingress(bob/web)?<br/>conntrack + flow log
  R->>B: CONNECT stream 3 from 10.42.1.2:51234
  B-->>R: ACCEPT stream 3
  R-->>A: ACCEPT stream 7
  A->>R: DATA stream 7
  R->>B: DATA stream 3
```

## Internet mode: OpenID Connect

The same star can run on a server on the internet, for remote or hybrid
classes. It must then not be open to everyone, so it signs people in with
**OpenID Connect**, using the fleet's own libraries:

| Piece | Library | What it does here |
| --- | --- | --- |
| verifying tokens | [go-authn/oidc](https://github.com/go-authn/oidc) | the relay verifies the token that comes with `Join`, `RefreshSession` and the teacher's calls: issuer compared exactly, audience required, expiry required, an allowlist of signature algorithms (no `none`, no HMAC), key-set refetches bounded. With `Type: "at+jwt"` it refuses an ID token presented as an access token (RFC 9068 §4, which the library cites). |
| logging people in | [go-authn/rp](https://github.com/go-authn/rp) | the browser login (authorization code with PKCE S256, state and nonce) and the teacher's `wctr-relay login`, a **loopback login** (RFC 8252) that listens on 127.0.0.1 only and opens the system browser |
| universities | [go-authn/bridge](https://github.com/go-authn/bridge) | an OpenID provider in front of a SAML federation: people "log in at their own university, through RENATER's Fédération Éducation-Recherche or eduGAIN". **The recommended identity provider for academia**, since many university IdPs speak SAML only. |

**Authorization model.**

- **Issuer and audience pinned** in the relay's configuration (`--oidc-issuer`,
  `--oidc-audience`); one issuer per relay.
- **Class membership**: an identity alone admits nobody. A student still joins
  a class with its **join code**; the class may also list who may join —
  `sub:<subject>`, `email:<address>` (only an address the issuer marked
  verified: go-authn/oidc's `Username` falls back to `email` only when
  `email_verified` is true, for the same reason) or `group:<group>` — and may
  **require** an identity (`--require-identity`), which a relay on the internet
  sets on every class.
- **Teacher role**: a configured group claim (`--teacher-group`) gives the
  admin role, but only on the **admin listener**, never on the public one.
- **One identity, one session per class**: joining again returns the same
  session (same addresses), and the per-session quotas are therefore
  **per identity**, not per IP address — a class behind one NAT is not one
  student.

**Long connections and short tokens.** A data connection lasts a lesson; an
access token lasts minutes. The relay's own session token is what the data
connection holds, bound to the session; before it expires the page calls
`RefreshSession` (with a fresh OIDC token, from the refresh grant go-authn/rp
handles) and sends the new session token in a `REAUTH` frame. No flow is
dropped. A connection whose token expires without `REAUTH` gets `GOAWAY` and
closes. **Logout and revocation**: the teacher's `KickSession`, or deleting the
class, ends the session and its tokens at once (the relay checks that the
session exists on every authentication, not only the signature).

**Internet hardening.**

- **Authenticate before upgrading**: the session token is checked before the
  WebSocket upgrade, and before any per-session state is allocated; an
  unauthenticated request costs a header parse and a signature check.
- **Per-IP limits before authentication**: token buckets on `Join` and on
  upgrade attempts per source address, and a cap on concurrent unauthenticated
  handshakes. `ReadHeaderTimeout` (5 s), `MaxHeaderBytes` (16 KiB) and idle
  timeouts against slowloris.
- **Max sessions per identity** (one per class) and per class (its subnets).
- Frame and flow bounds as on the LAN ([security audit](#6-security-audit)).
- **No admin API on the public listener**: the admin API stays on the Unix
  socket and loopback, or on a separate `--admin-listen` that requires the
  teacher role (or the admin token).
- **Audit log** (JSON lines, `0600`): logins, refusals, kicks, class and rule
  changes, with pseudonymous identities.

**Privacy (GDPR).** Students are data subjects, often minors. The relay stores,
per session: a display name the student typed, a derived DNS label, and — with
OpenID Connect — a **pseudonymous identifier** (a hash of issuer and subject),
never the token, the e-mail address or the groups (they are checked at the
moment of joining and dropped). Flow logs hold addresses and ports, not
contents. Retention: everything about a class is deleted with the class
(`wctr-relay class delete`); the audit log is pruned after 30 days by default
(`--audit-retention`). The teacher's institution is the controller and should
say so in the course's information notice.

## 6. Security audit

The relay is a network service on a LAN full of untrusted clients, parsing
their frames. Every row is a requirement for the implementation and a test.

| Threat | Mitigation |
| --- | --- |
| A student impersonates another (identity, address) | Session token bound to the session; ports and addresses assigned by the relay; source addresses in frames ignored and rewritten; a port id in a frame must belong to the session (Neutron's anti-spoofing, without `allowed_address_pairs`). |
| A student claims another's DNS name | Names are `<container>.<student-label>.class`, the label fixed by the relay at `Join`; label collisions get a suffix. |
| Display names injected into the dashboard | Display names are data: length-limited (40), control characters removed, escaped by the dashboard; the label is derived, ASCII. |
| Brute-forcing the class code | 10 characters from 32 symbols (50 bits); `Join` rate-limited per source IP (5 per minute) and per class; the code expires with the class or on rotation. |
| Flooding and resource exhaustion | Per session: 256 open streams, 64 listeners, 32 pending connects, 10 ports, token buckets on frames per second (2 000) and bytes per second (2 MiB/s, burst 8 MiB), all configurable by the teacher. Per connection: 1 MiB of buffered data. Global: a cap on sessions (`port` quota) and on unauthenticated connections (32, 10 s to authenticate). |
| Oversized or malformed frames | WebSocket read limit of 16 KiB + header per message; a frame whose `length` disagrees with the message is a protocol error that closes the connection; every decoder fuzzed (`go test -fuzz`, corpus committed). |
| Slow readers (memory held) | Credit-based windows; a stream with no credit for 60 s is reset; WebSocket ping every 15 s, close after 45 s without a pong. |
| A malicious public page attacks the relay through a student's browser | Every API call and the WebSocket need a token in a header or in the first frame, never a cookie, so no ambient authority exists to ride (no CSRF). The WebSocket upgrade checks `Origin` against the relay's own origin and `https://go-wasmcontainers.github.io`. The `Host` header must be one of the relay's names or addresses (DNS rebinding). Local Network Access prompts are a second barrier, not the first. |
| The admin API reachable from the LAN | Off by default: the admin side listens on the Unix socket and loopback only; `--admin-listen` is explicit and can require mTLS. |
| Secrets | Admin token, CA key and token-signing key: `0600` files under `os.UserConfigDir()`, never in a URL, a command line, an environment variable or a log; logs redact anything token-shaped. Join links carry the code and the certificate hash in the URL fragment. The CA is name-constrained. |
| A compromised relay | It sees all classroom traffic by design. It runs as the teacher's user, needs no privilege (port 8443), and holds no credential beyond its own. |
| Supply chain | `govulncheck` in CI; dependencies limited to connect-go, protobuf, a WebSocket library and, later, quic-go/webtransport-go (which implements draft-16 of WebTransport over HTTP/3, not an RFC: pinned and re-checked when the draft moves). |

### Chaos cases

| Fault | Expected behaviour |
| --- | --- |
| A student reloads the page | The page reconnects with its session token; within a **120 s grace period** the relay keeps the session's ports and addresses; open streams are reset (the container sees a connection reset, as after a network cut); listeners are re-registered by the engine. |
| The teacher's laptop sleeps | Every WebSocket drops. On wake, browsers reconnect with exponential backoff and jitter (1 s to 30 s), so 40 browsers do not reconnect in the same millisecond; sessions older than the grace period are re-joined silently with the token. |
| The relay restarts or crashes | Classes, sessions, groups, rules, ports and address allocations are persisted (written to a temporary file, then renamed) after every change; tokens stay valid because the signing key is on disk. Flows do not survive: they are reset and logged as such. |
| Clock skew between machines | Tokens are checked against the relay's clock only; the browser's clock is never trusted. |
| Wi-Fi loss | Detected by the ping/pong timeout, handled as a reload. |
| Mixed versions (page and relay) | The first frame carries a protocol version; the relay refuses an unknown major version with a message the page shows. In deployment (b), skew cannot happen. |
| Two teachers' relays on one LAN | Each class has its own code, certificate and CA; a join link names one relay. |

The chaos campaign (M-relay-5) injects these faults into a test classroom of
real browsers and records what breaks.

## 7. Performance

**Budget.** The relay must not be what limits a lab. The assumptions to
measure, not results:

- 40 students on one access point share its airtime; a lab's traffic is small
  (HTTP requests, a database query, a chat), a few hundred kB/s per student,
  so the aggregate stays far below what one Go process forwards;
- container-to-container round trip ≈ two Wi-Fi round trips (A to relay, relay
  to B) plus the relay's processing, which must stay **under 1 ms** at the
  99th percentile;
- framing cost: our 10-byte header, the WebSocket header (2 to 10 bytes, plus a
  4-byte masking key from client to server, RFC 6455) and the TLS record: under
  0.3 % for a full 16 KiB frame, significant only for very small writes, which
  the engine coalesces within one event-loop turn;
- head-of-line blocking on the single TCP connection is the main latency risk
  on a lossy Wi-Fi; WebTransport's independent streams are the remedy.

**What to measure, against what.**

| Measure | Reference | Target |
| --- | --- | --- |
| relay forwarding throughput, one core, 16 KiB frames | a direct TCP connection between the same two processes on loopback | at least 50 % of it |
| relay frames per second, one core, 64-byte frames | the same | reported, then optimised |
| added latency per hop (p50, p99) at 40 sessions × 10 streams | direct loopback TCP | p99 < 1 ms |
| memory per idle session | none | < 64 KiB |
| rule evaluation per connect, 500 rules | linear scan | constant per flow after the first lookup, indexed by protocol and port |
| end to end in a real classroom (40 browsers) | the same lab with all containers in one browser | reported |

Measurements follow the fleet's rule: interleaved rounds, single core, never on
a loaded machine. Masking and unmasking WebSocket payloads is a byte loop on
the hot path: it goes to a shared SIMD kernel (go-simd) if it shows in the
profile, not into the relay.

## High availability

**The problem.** A class runs on relays in a datacenter; if that relay stops
answering, the class must move to another one without losing its setup.

**What state exists.**

| Plane | State | On failover |
| --- | --- | --- |
| control | classes, join codes, members and their identities' hashes, security groups and rules, port and address/DNS assignments, session-token signing keys, audit log | **replicated**: survives |
| data | live streams, UDP connection tracking, listeners | **not replicated**: flows are reset (containers see `ECONNRESET`, which is teachable); listeners are re-registered by the engines when they reconnect; addresses, names and rules are the same |

**Replication: Raft, not a CRDT.** The control plane is a replicated state
machine (the Raft paper, §2: servers "compute identical copies of the same
state" and are "fully functional (available) as long as any majority of the servers
are operational"; "they do not depend on timing to ensure the consistency of the
logs"). Security rules need that linearizability. A CRDT (go-crdt exists in the
fleet) merges concurrent updates instead of ordering them: an add-wins set
would bring back an allow-rule the teacher deleted on another replica during a
partition. **A deleted allow-rule must never reappear by merge**, so CRDTs are
the wrong tool here. The library candidate is etcd's `raft` (the core
algorithm only: "the application supplies its own network transport and
persistent storage"; Apache-2.0; used by etcd, CockroachDB and TiDB). The
relay already reaches its state through a `Store` interface (`FileStore` on a
single relay), which the replicated store implements.

**One relay per class: an epoch lease.** The star needs a class to live on
**one** relay at a time. The replicated store holds, per class, a lease
`(relay_id, epoch, expiry)`; a relay takes or renews it through the log, and
each new holder increments the **epoch** (Raft's terms are the model: "terms
act as a logical clock", and a server rejects a request with a stale term).
The epoch is in `Welcome` and in `Class`; a client **refuses a relay that shows
an older epoch** than one it has seen (fencing), so a partitioned old holder
cannot keep half a class. A relay that cannot renew its lease stops serving
the class before the lease expires, with a margin for clock drift.

**Quorum.** A majority quorum of 2 datacenters does not survive the loss of
either (2 of 2 needed). Three sites are needed: two datacenters and a small
**witness** (a third voter that serves no class). The trade-off is written
down rather than hidden: without the witness, a two-site deployment is
active/passive with manual failover.

**Client-side failover, after DERP.** Tailscale's DERP documentation describes
the pattern: each client gets a map of relays, picks one by latency, "selects
the next closest region" when its region is unreachable, and saves the list so
it works while the coordination server is down. Ours: the join link carries a
**signed class descriptor** listing the relays in order (not DNS-only failover:
resolvers cache past a TTL). A client health-checks, reconnects with
exponential backoff and jitter, and resumes with its **session token, valid on
any relay** of the cluster (signing keys are in the replicated store and
rotated through it; OpenID Connect tokens stay valid because issuer and
audience are the cluster's). Each relay has its own certificate from
servercert.

**Phase 2: federation.** Relay-to-relay links would let one class span relays
(geographic proximity, partial failure), the firewall still enforced at both
ends. Not before the single-holder design is shipped and measured.

**Chaos plan** (M-relay-HA): kill the lease holder in the middle of a lab;
partition one datacenter; skew the clocks of the lease holder; run mixed relay
versions during a rolling upgrade; make every student reconnect at once
(thundering herd: jitter, and admission control that answers "retry after"
rather than accepting more than it can serve).

## Alternatives considered

Before writing the relay, existing overlay networks were evaluated (October
2026) against what a lab needs: running in a browser tab with nothing
installed, inside the engine's Go `js/wasm`, a firewall the teacher controls,
flow logs, a LAN mode without internet, and high availability.

| | Browser client | Firewall and where | HA | Licence |
| --- | --- | --- | --- | --- |
| **Tailscale** building blocks: DERP, `tsnet`, `cmd/tsconnect` | yes: `cmd/tsconnect/wasm` runs wireguard-go and gVisor's netstack in the page, DERP over WebSocket (`derphttp/websocket.go`, js build); the npm package's `main.wasm` is 26.4 MB (4.73 MB brotli); its JS API exposes `login`, `ssh` and `fetch` ([`@tailscale/connect` files](https://app.unpkg.com/@tailscale/connect)); it ships logs to Tailscale's log service by default | ACLs, deny by default, directional, enforced **on the receiving device** | DERP regions with client failover; the control server is Tailscale's | BSD-3-Clause |
| **headscale** (open control server) | uses Tailscale's clients | Tailscale ACLs; "OIDC groups cannot be used in ACLs"; flow logs only an open issue | single instance; SQLite recommended, PostgreSQL "in maintenance mode"; scope: homelabs, "not enterprise software" | BSD-3-Clause |
| **NetBird** | a browser client exists (Go to WASM, wireguard-go, WebSocket relay), as a dashboard feature for SSH and RDP, logged in with the admin's credentials | groups and rules | no HA statement found | management, signal and relay **AGPLv3** |
| **Nebula** | none (UDP only) | groups in certificates | lighthouses | MIT |
| **ZeroTier** | none found | a rules engine | — | MPL-2.0 plus non-free parts |

**What decided it.**

- With end-to-end WireGuard, **the relay sees nothing**: good for privacy, but
  there is no relay-side enforcement and no relay-side flow log. Filtering at
  the receiver is sound against untrusted students (a student can only weaken
  their own node), but egress rules cannot be enforced on a node the student
  controls, and logs kept by the endpoints can be forged by the student whose
  flows they describe. The relay-side log is the lesson.
- Neutron's model only partly translates to Tailscale ACLs: ingress rules map
  to tags and ports; egress rules and stateless groups do not.
- headscale has no HA, so adopting it would not remove the Raft work.
- A 26 MB WireGuard + netstack per tab, against a socket-level relay that adds
  no network stack to the engine (performance of both is unmeasured).

**Decision: hybrid (option C).** Our relay stays the classroom data plane and
firewall; identity and certificates come from go-authn; client failover
follows DERP's ordered-relay pattern. A later option, not planned: `tsnet` in
the teacher's relay to reach native machines on an existing tailnet.

## 8. Roadmap and what the engine must expose

### The engine's `NetworkDriver`

The engine maps the socket host functions onto a driver; the in-browser network
it has today becomes the `local` driver, and the relay is a second one:

```go
// Package netdriver, in go-wasmcontainers/engine (planned).
type NetworkDriver interface {
	// Attach gives a container an endpoint on a network; the driver
	// assigns its address and DNS name.
	Attach(ctx context.Context, c ContainerRef, network string) (Endpoint, error)
	Close() error
}

type Endpoint interface {
	Addr() netip.Addr
	Listen(ctx context.Context, network string, port uint16) (net.Listener, error)       // "tcp"
	Dial(ctx context.Context, network string, addr netip.AddrPort) (net.Conn, error)      // "tcp", "udp"
	ListenPacket(ctx context.Context, network string, port uint16) (net.PacketConn, error) // "udp"
	Resolve(ctx context.Context, name string) ([]netip.Addr, error)
	Detach() error
}
```

The engine owns this interface; the relay's client library implements it, so
native engines (wazero) and browser engines use the same driver code, and the
relay's tests need no browser.

### Milestones

| Milestone | Content | Tests |
| --- | --- | --- |
| **M-relay-1** | `NetworkDriver` in the engine (by the engine's owner); the relay as a Go library: framing, flow broker, firewall, DNS, conntrack | **in-process relay + two native engines** in one `go test`: TCP across, UDP, DNS, deny, revoke, stateless; fuzzed decoders; rule evaluation tables |
| **M-relay-2** | `wctr-relay` binary: WebSocket transport, control API (connect-go), cobra CLI, persistence, flow logs, quotas; TLS from servercert (files or ACME) or the internal CA; OpenID Connect verification (go-authn/oidc) and the internet hardening | native engines over loopback `wss`; restart with state; rate limits; a test identity provider; chaos on the native path |
| **M-relay-2b** | the live flow map: `WatchFlowMap` (aggregated, coalesced); the graph, legend, filter, detail and sparkline widgets added to go-widgets (their own releases); the page served by the relay; teacher and per-student views | Playwright (Chromium, Firefox) on real flows through an in-process relay; update latency and CPU at 40 sessions |
| **M-relay-3** | browser driver in the Web Worker; deployment (b): the relay serves the playground; join-flow detection | Chrome, Firefox and WebKit driven by Playwright against a local relay; secure-context check after a certificate warning |
| **M-relay-4** | deployment (a) from GitHub Pages; WebTransport with certificate hashes; teacher dashboard over gRPC-Web/Connect; `wctr-relay login` (go-authn/rp loopback); DNS-01 in servercert | the Local Network Access prompt in each browser; transport fallback |
| **M-relay-5** | `wctr_net.icmp_echo` and a `ping` image; chaos campaign; performance in a real classroom | the chaos table above; the performance table above |
| **M-relay-HA** | Raft-replicated `Store` (etcd `raft`), per-class epoch leases, signed class descriptor with ordered relays, client failover | the high-availability chaos plan |

## Open questions

1. Which DNS providers must servercert's DNS-01 support first (schools'
   registrars; RFC 2136 for a school's own DNS server)?
2. The witness site for a two-datacenter deployment: who runs it?

## References

Opened for this design (October 2026):

- W3C, [WebTransport](https://www.w3.org/TR/webtransport/), Candidate Recommendation Snapshot, 30 July 2026: custom certificate requirements and certificate-hash verification
- MDN, [`WebTransport()` constructor](https://developer.mozilla.org/en-US/docs/Web/API/WebTransport/WebTransport): `serverCertificateHashes`, secure context, availability in workers
- MDN browser-compat-data, [`api/WebTransport.json`](https://github.com/mdn/browser-compat-data/blob/main/api/WebTransport.json): browser versions of the constructor and of `serverCertificateHashes`
- MDN, [`RTCDataChannel`](https://developer.mozilla.org/en-US/docs/Web/API/RTCDataChannel): availability
- MDN, [Mixed content](https://developer.mozilla.org/en-US/docs/Web/Security/Mixed_content)
- MDN, [`SharedArrayBuffer`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer): secure context and cross-origin isolation
- WICG, [Local Network Access](https://wicg.github.io/local-network-access/), Draft Community Group Report, 7 August 2026
- Chrome for Developers, [New permission prompt for Local Network Access](https://developer.chrome.com/blog/local-network-access)
- IETF, [RFC 6455, The WebSocket Protocol](https://www.rfc-editor.org/rfc/rfc6455): framing and masking
- OpenStack, [Networking API v2.0 reference](https://docs.openstack.org/api-ref/network/v2/index.html): ports, `port_security_enabled`, `allowed_address_pairs`
- OpenStack neutron-lib API reference sources: [security groups](https://github.com/openstack/neutron-lib/blob/master/api-ref/source/v2/security-groups.inc) (stateful groups), [security group rules](https://github.com/openstack/neutron-lib/blob/master/api-ref/source/v2/security-group-rules.inc), [parameters](https://github.com/openstack/neutron-lib/blob/master/api-ref/source/v2/parameters.yaml) (rule fields, quotas)
- OpenStack Neutron spec, [Configurable default security group rules](https://specs.openstack.org/openstack/neutron-specs/specs/2023.2/configurable-default-sg-rules.html)
- OpenStack Nova, [Security groups](https://docs.openstack.org/nova/latest/user/security-groups.html): default deny, additive rules
- [stealthrocket/net](https://github.com/stealthrocket/net), and its [WasmEdge socket imports](https://github.com/stealthrocket/net/blob/main/wasip1/syscall_wasmedge_wasip1.go)
- [container2wasm](https://github.com/ktock/container2wasm): `c2w-net` and browser networking
- [gvisor-tap-vsock](https://github.com/containers/gvisor-tap-vsock): a netstack-based network for VMs
- [Connect protocol](https://connectrpc.com/docs/protocol/) and [connect-go](https://github.com/connectrpc/connect-go)
- [quic-go/webtransport-go](https://github.com/quic-go/webtransport-go)
- [mkcert](https://github.com/FiloSottile/mkcert)
- [go-widgets/toolkit](https://github.com/go-widgets/toolkit), [go-widgets/webcanvas](https://github.com/go-widgets/webcanvas) and [go-gfx/qr](https://github.com/go-gfx/qr) (the fleet's rendering pieces, for the flow map)
- Caddy, [Automatic HTTPS](https://caddyserver.com/docs/automatic-https) (DNS challenge, internal CA, IP certificates, on-demand TLS), [`acme_server`](https://caddyserver.com/docs/caddyfile/directives/acme_server) and [`caddy trust`](https://caddyserver.com/docs/command-line)
- [CertMagic](https://github.com/caddyserver/certmagic): the library behind Caddy's automatic HTTPS (DNS-01 through libdns; no internal issuer)
- IETF, [RFC 8555, ACME](https://www.rfc-editor.org/rfc/rfc8555), §8.4 (DNS-01)
- Let's Encrypt, [Short-lived and IP address certificates generally available](https://letsencrypt.org/2026/01/15/6day-and-ip-general-availability/) (15 January 2026)
- go-authn: [servercert](https://github.com/go-authn/servercert), [oidc](https://github.com/go-authn/oidc), [rp](https://github.com/go-authn/rp), [bridge](https://github.com/go-authn/bridge)
- D. Ongaro, J. Ousterhout, [In Search of an Understandable Consensus Algorithm (Raft, extended version)](https://raft.github.io/raft.pdf), and [etcd-io/raft](https://github.com/etcd-io/raft)
- Tailscale: [DERP servers](https://tailscale.com/kb/1232/derp-servers), [ACLs](https://tailscale.com/kb/1018/acls), [`tsnet`](https://pkg.go.dev/tailscale.com/tsnet), [`cmd/tsconnect`](https://github.com/tailscale/tailscale/tree/main/cmd/tsconnect) and its [`wasm_js.go`](https://github.com/tailscale/tailscale/blob/main/cmd/tsconnect/wasm/wasm_js.go), [`derphttp/websocket.go`](https://github.com/tailscale/tailscale/blob/main/derp/derphttp/websocket.go)
- [headscale](https://github.com/juanfont/headscale): [FAQ](https://headscale.net/stable/about/faq/) and [features](https://headscale.net/stable/about/features/)
- NetBird: [repository](https://github.com/netbirdio/netbird) (licences) and [browser client architecture](https://docs.netbird.io/manage/peers/browser-client/architecture)
- [Nebula](https://github.com/slackhq/nebula), [ZeroTierOne](https://github.com/zerotier/ZeroTierOne)
