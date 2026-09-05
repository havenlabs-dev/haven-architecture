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
| `0.0.0.0` | All interfaces — requires a reason. | `sshd`; the API |

Binding a database to `127.0.0.1` rather than `0.0.0.0` is the cheapest security
control available, and it is applied by default. Postgres containers publish to
loopback only; nothing reaches them except the service that owns them.

### A known item, stated plainly

The API currently binds `0.0.0.0:8000` rather than the tailnet interface. In
practice it is reached over the mesh, but a broad bind means the listener is
also offered to whatever physical LAN the host sits on, and it should be
narrowed to the tailnet address to match everything else.

I am listing this rather than describing the system as tighter than it is. When
I tried to verify it by connecting from another machine on a different subnet, I
got no route — which proves nothing about the local LAN, so it does not count as
evidence. An unverified control is not a control.

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
