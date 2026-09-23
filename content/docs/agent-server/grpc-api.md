---
title: Interfaces
linkTitle: Interfaces
description: Two ports, and what talks to them.
weight: 5
---

The image exposes two ports and nothing else. There is no web UI, no admin
endpoint and no configuration API — the agent server is driven entirely by the
agent client.

| Port | Protocol | Who connects | Expose it? |
| --- | --- | --- | --- |
| `50052` | gRPC | the agent client | **Loopback or private network only** |
| `8000` | HTTP | your monitoring | As needed |

## Port 8000 — health

One endpoint:

```sh
curl -s http://localhost:8000/health     # → ok
```

A **liveness** check, not a readiness one. It returns `ok` as soon as the
process is up, including when no calls are running — which is the normal state.
It does not tell you whether calls succeed; the log does that.

Safe to expose to a load balancer or monitoring system. It reveals nothing.

## Port 50052 — control

This is where the agent client tells the server which room to join, which agent
to run, and where to bill it. Everything about a call travels over this port.

**It must be protected.** Two reasons:

1. **There is no encryption.** The server holds no TLS certificate. The API key
   and the service-account credentials each call carries cross this port in
   clear text.
2. **It controls live calls.** A caller who reaches it can place agents into
   rooms and disconnect ones already talking.

The safe shapes are:

- Bind to `127.0.0.1` and run a TLS-terminating proxy in front — what the
  published examples do.
- Keep it on a private network that only the agent client can reach.

Publishing `0.0.0.0:50052` to the internet is not one of them.

### Authentication

Every request must present `ULAI_GRPC_API_KEY`. The client is configured with
the same value.

Leave the variable unset and the server accepts **everything**, including
requests that disconnect live agents. It logs a warning saying so at startup,
once:

```
[grpc] WARNING: ULAI_GRPC_API_KEY is unset — :50052 accepts unauthenticated
calls, including ones that disconnect live agents
```

One key, shared by every client. There is no per-client identity and no way to
revoke one caller without rotating for all of them, so treat the key as an
infrastructure secret rather than a per-team credential.

### What it actually serves

For completeness, the port serves three gRPC services — `AgentBridge` (put an
agent in a room, take it out), `AgentSession` (drive a live conversation) and
`AgentDispatch` (receive calls as they start). Operating the server requires no
knowledge of them; they matter to whoever builds the client.

Server reflection is enabled, so `grpcurl` can inspect the port for debugging —
behind the same key:

```sh
grpcurl -H 'x-api-key: <key>' -plaintext localhost:50052 list
```

## Outbound connections

The server also makes connections *out*, which your egress rules must allow:

| To | For |
| --- | --- |
| The Ulai control plane and SFU nodes | joining rooms and carrying audio |
| The AI backend (Vertex AI by default) | the conversation itself |

Both are named per call by the client, not configured here.
