# Networking

## Shape

There is no public ingress to the application. Every client — phone, laptop,
watch relay — reaches the host over a WireGuard mesh (Tailscale), which gives
each device a stable address on a private overlay and handles NAT traversal
without any port forwarding on the home router.

```
iPhone ─┐
laptop ─┼─ WireGuard mesh (100.x.x.x/32 per node) ─→ haven :8000 / :1616
watch  ─┘                                              (via phone relay)
```

Three nodes are enrolled: the server, a laptop, and a phone. Access is by device
identity, not by password or by being on the right LAN.

## Interface-binding discipline

Listening sockets are bound deliberately, and the binding *is* the access
control layer:

| Bind address | Meaning | Examples |
|---|---|---|
| `127.0.0.1` | Host-only. Never reachable off-box. | Private metasearch, vehicle command proxy, both Postgres instances, tunnel daemon |
| `<tailnet IP>` | Reachable from enrolled devices only. | Web dashboard, metrics, automation UI, vehicle telemetry UI |
| `0.0.0.0` | All interfaces — requires a reason, and the API's is below. | `sshd`; the API |

Binding a database to `127.0.0.1` rather than `0.0.0.0` is the cheapest security
control available, and it is applied by default. Postgres containers publish to
loopback only; nothing reaches them except the service that owns them.

### The API binds broadly on purpose, and is guarded in-process

The API listens on `0.0.0.0:8000`, which by itself would offer it to the
physical LAN. It is not narrowed, because binding is all-or-one — `uvicorn`
takes a single `--host` — and eleven internal callers reach the API over
`127.0.0.1`. Rebinding to the tailnet address would break every one of them.

The textbook fix is an `nftables` rule scoped to the `tailscale0` interface.
That is unavailable here: there is no passwordless sudo on this host and the
unit files live under `/etc/systemd/system`.

So the perimeter is enforced **in-process**, as the outermost middleware, ahead
of any handler or logging work. A request is served only if its source is
loopback, `100.64.0.0/10` (the CGNAT block Tailscale allocates from), or
Tailscale's IPv6 ULA prefix. Everything else gets a 403 and a logged warning.
Loopback stays reachable for internal callers; the LAN does not.

Two details that matter more than the rule itself:

- **It fails closed.** A missing or unparseable client address is refused, not
  waved through. The one thing a perimeter must never do is guess.
- **`/healthz` and `/ping` stay open**, so a blocked client can still tell
  "the server is down" from "you are not on the tailnet." A perimeter that
  makes those two indistinguishable costs you an hour of misdiagnosis.

An `HAVEN_ALLOW_ALL_IPS=1` escape hatch exists for the case where the guard
misjudges a legitimate client. It is off, and the test suite asserts that it is
off — an escape hatch nobody checks is just a hole with better manners.

This guard shipped on 2026-07-29 and had **no test until September**, which
is its own lesson: the control existed and nothing proved it could refuse. It
now has 28, including LAN and public addresses being denied, the CGNAT block
boundaries, malformed input failing closed, and — the one that matters most —
an assertion that the middleware is actually *registered*, not merely defined.
A guard nothing installs is the exact failure mode described in
[observability](observability.md).

## Egress

Outbound is where the real exposure lives, and it is treated as a privacy
question rather than a security one:

- LLM providers, banking aggregation, health APIs, and search all receive
  outbound requests.
- Which provider handles which class of data is a deliberate decision. Search
  and transcription were moved to local or self-hosted implementations
  specifically so that queries stopped leaving the network.
- One outbound tunnel exists for a single automation webhook that requires a
  public callback URL. It is scoped to that one service and terminates in a
  container, not on the host.

## Diagnosis note: a dead tunnel is not a dead machine

During a multi-worker job the host stopped answering and looked, from outside,
like it had died. It had not. Home Wi-Fi had dropped, which took the mesh link
with it. The server had 141 days of uptime and was sitting at 42 °C the whole
time.

The trap is that the mesh control plane can still report a node as `Online`
after the data path is gone. The fields that do not lie are `LastHandshake` and,
once you are back in, `uptime`. **Confirm the host is actually down before
naming a cause for why it went down.**
