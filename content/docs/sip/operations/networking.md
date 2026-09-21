---
title: Networking
description: Ports, host networking, and why port publishing breaks calls.
weight: 2
---

## Ports

| Port | Protocol | Purpose |
| --- | --- | --- |
| `8082` | TCP | HTTP API — `/health`, `/calls`, `/sip/originate` |
| `5060` | UDP and/or TCP | SIP signalling (`5061` conventionally for TLS) |
| `10000–10500` | UDP | RTP media, one even port per call |

The RTP range sizes the gateway's concurrency: roughly 250 simultaneous calls at
the default range, since allocation uses even ports only by RTCP convention.
Widen `SIP_RTP_PORT_LOW`/`HIGH` to carry more.

The carrier must be able to reach `SIP_PUBLIC_IP` on the SIP port **and** the
whole RTP range. A firewall that allows 5060 but not 10000–10500 produces a call
that connects and then has no audio — the single most common misconfiguration.

## Port publishing versus host networking

Run the container with `--network host`.

Publishing the ports with `-p` breaks the gateway in two separate ways:

1. **501 userland proxies.** `-p 10000-10500:10000-10500/udp` spawns one
   `docker-proxy` process per port. That is slow to start, heavy at rest, and
   adds a hop to every media packet.
2. **The trunk ACL rejects the call.** A proxied INVITE arrives from the bridge
   gateway's address, not the carrier's. The resolver checks the source IP
   against the trunk's ACL, does not find it, and answers `403 Forbidden`.

The same reasoning is why `SIP_PUBLIC_IP` must be a routable address: it is
copied into every SDP and `Contact` the gateway sends, and it is where the
carrier will stream media.

## Bind address versus advertised address

Two different variables, routinely confused:

| Variable | Meaning |
| --- | --- |
| `SIP_RTP_HOST` (default `0.0.0.0`) | What RTP sockets **bind** to, inside the machine |
| `SIP_PUBLIC_IP` (required) | What the gateway **advertises** to the carrier |

On AWS, GCP, Azure and most VPS providers the public IP is attached at a NAT or
load-balancer layer and is not present on any interface inside the VM. Binding
to it fails with *cannot assign requested address*, so the default `0.0.0.0` is
the right answer nearly everywhere.

## NAT and ICE

The room side is WebRTC, so it needs ICE. The gateway is STUN-only by default
(`stun:stun.l.google.com:19302`), which works on a public host and fails behind
symmetric NAT. Add TURN there:

```sh
-e TURN_URLS='turn:turn.example.com:3478?transport=udp,turns:turn.example.com:5349' \
-e TURN_USERNAME=ulai \
-e TURN_PASSWORD=...
```

The startup line reports how many ICE servers were configured:

```text
sip-sfu-gateway: HTTP listening on :8082 (control plane https://stgcp.ulai.co.in, ice_servers=1)
```

`ice_servers=1` means STUN only.

## Symmetric RTP

Carriers behind NAT routinely send RTP from a port they never advertised in
SDP. The gateway latches onto the source of the first well-formed packet and
sends there, rather than trusting the SDP address — without which roughly half
of inbound calls would be one-way. See
[media](/docs/sip/concepts/media/#symmetric-rtp-and-the-source-gate).

## Transport choice

| `SIP_TRANSPORT` | Notes |
| --- | --- |
| `udp` | The default, and what most trunks use |
| `tcp` | Useful where large INVITEs fragment |
| `tls` | Requires `SIP_TLS_CERT_PATH` and `SIP_TLS_KEY_PATH`; implies offering SRTP on outbound trunks whose transport is also TLS |

A TLS trunk calling back a UDP-only listener is a real failure mode: the `BYE`
never arrives and the call would hang forever, which is what
`SIP_RTP_TIMEOUT_SECONDS` exists to catch.
