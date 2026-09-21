---
title: Configuration
description: Every environment variable, its default, and what breaks when it is wrong.
weight: 4
---

Configuration is read from the environment **once**, at startup, into a single
validated struct. Everything downstream reads that struct, never the
environment — which is also why a test can construct one directly.

Validation reports *every* problem at once, so a misconfigured deployment does
not have to be fixed one restart at a time.

## Required

| Variable | Description |
| --- | --- |
| `ULAI_CONTROL_PLANE_URL` | Base URL of the Ulai control plane, e.g. `https://stgcp.ulai.co.in`. Rooms are created, joined and terminated here. |
| `SIP_ROUTING_REDIS_URL` | The routing store every call is resolved against. `REDIS_URL` is accepted as a fallback. |
| `SIP_PUBLIC_IP` | The address advertised in SDP and `Contact`. Must be routable by the carrier. `0.0.0.0` is rejected — it is not a listen address, it is what the carrier is told to stream media to. |

## SIP signalling

| Variable | Default | Description |
| --- | --- | --- |
| `SIP_LISTEN_ADDR` | `0.0.0.0:5060` | What sipgo binds to. `host:port`; the host may be `0.0.0.0`. |
| `SIP_TRANSPORT` | `udp` | `udp`, `tcp` or `tls`. |
| `SIP_TLS_CERT_PATH` | — | Required when `SIP_TRANSPORT=tls`. |
| `SIP_TLS_KEY_PATH` | — | Required when `SIP_TRANSPORT=tls`. |
| `SIP_SESSION_HEADER` | `X-Session-Id` | The INVITE header that may point a call at an existing room. Must not be blank. |

## RTP media

| Variable | Default | Description |
| --- | --- | --- |
| `SIP_RTP_HOST` | `0.0.0.0` | What RTP sockets **bind** to, as distinct from `SIP_PUBLIC_IP`, which is what is **advertised**. On most cloud VMs the public IP is attached at a NAT or load-balancer layer and is not on any interface inside the VM, so binding to it fails with *cannot assign requested address*. The default is right nearly everywhere. |
| `SIP_RTP_PORT_LOW` | `10000` | Bottom of the RTP allocation range. |
| `SIP_RTP_PORT_HIGH` | `10500` | Top of the range. Must be above the low end, and both within 1–65535. Even ports only, by RTCP convention. |
| `SIP_RTP_TIMEOUT_SECONDS` | `30` | End a call whose inbound audio has stopped for this long, as if the far end had hung up — the backstop for a `BYE` that never arrives. Paused while the call is on hold. `0` disables it. |

## HTTP API

| Variable | Default | Description |
| --- | --- | --- |
| `SIP_HTTP_PORT` | `8082` | Port for `/health`, `/calls` and `/sip/originate`. Chosen clear of the worker (8080) and the Tata gateway (8081). |

## Rooms

| Variable | Default | Description |
| --- | --- | --- |
| `ULAI_PROJECT_ID` | — | Default project for `POST /sip/originate` when the body omits `project_id`. The single-tenant deployment's answer to sending it every time. |
| `ULAI_MAX_PARTICIPANTS` | `8` | Size of a room the gateway creates. Overridden per request by `max_participants`. Must be positive. |
| `SIP_TERMINATE_SESSION_ON_HANGUP` | `true` | End the whole session — every participant, every transport — when the SIP leg drops, rather than merely leaving the room. Set `false` only when a room reached via `X-Session-Id` or `session_id` is genuinely shared. A room the gateway created is always terminated regardless. |

## ICE

The gateway is STUN-only by default (`stun:stun.l.google.com:19302`), which is
fine on a public host and will fail behind symmetric NAT.

| Variable | Default | Description |
| --- | --- | --- |
| `TURN_URLS` | — | Comma-separated TURN URLs. Appended to the STUN default. |
| `TURN_USERNAME` | — | TURN credential. |
| `TURN_PASSWORD` | — | TURN credential. |

## Go runtime

At startup the process aligns the Go runtime with its cgroup limits — two
defaults that are correct on a bare host become latency or OOM bugs inside a
container. An explicit environment variable always wins.

| Variable | Behaviour when unset |
| --- | --- |
| `GOMAXPROCS` | Pinned to `ceil(cgroup CPU quota)` when that is below the host core count. Left alone otherwise. |
| `GOMEMLIMIT` | Set to 85% of the cgroup memory limit, or of total system RAM when there is none. The headroom covers goroutine stacks and the off-heap libopus and resampler allocations `GOMEMLIMIT` cannot see. |
| `GOGC` | Raised to `200`, trading memory for fewer GC cycles and less latency jitter, bounded by the limit above. |

Each decision is logged at startup:

```text
[runtime] GOMAXPROCS=4 (cgroup CPU quota=4.00, host cores=8)
[runtime] GOMEMLIMIT=3481MiB (cgroup limit=4096MiB, 85%)
[runtime] GOGC=200
```

## Built-in timeouts

Not configurable — they are constants, listed here because they explain the
behaviour you will observe.

| Constant | Value | Bounds |
| --- | --- | --- |
| `ResolveTimeout` | 5 s | One routing-store lookup |
| `CreateTimeout` | 15 s | Creating a room |
| `JoinTimeout` | 15 s | Joining a room |
| `DialTimeout` | 15 s | The WebRTC signalling dial |
| `OriginateTimeout` | 60 s | An outbound INVITE, through ring |
| `TerminateTimeout` | 5 s | Terminating a room, or hanging up a leg |
| `DrainTimeout` | 60 s | The shutdown drain |
| `CancelGrace` | 5 s | Extra time after the drain deadline cancels calls |
| `MaxRequestBytes` | 8 KiB | An originate body — it is a control API, not an upload |
| publish timeout | 2 s | One event publish, best-effort |
| control-plane HTTP timeout | 10 s | Any single control-plane request |

## Example

```sh
# Required
ULAI_CONTROL_PLANE_URL=https://stgcp.ulai.co.in
SIP_ROUTING_REDIS_URL=redis://user:password@redis.example.com:6379/0
SIP_PUBLIC_IP=203.0.113.10

# SIP + RTP
SIP_LISTEN_ADDR=0.0.0.0:5060
SIP_TRANSPORT=udp
SIP_RTP_PORT_LOW=10000
SIP_RTP_PORT_HIGH=10500
SIP_RTP_TIMEOUT_SECONDS=30

# HTTP
SIP_HTTP_PORT=8082

# Rooms
SIP_TERMINATE_SESSION_ON_HANGUP=true
```
