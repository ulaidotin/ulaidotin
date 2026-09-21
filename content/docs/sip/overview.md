---
title: Overview
description: What the gateway does, and where it sits in the platform.
weight: 1
---

## What it is

The Ulai SIP Gateway is a single Go binary (`sip-sfu-gateway`) that joins two
worlds:

- **SIP/RTP** — a carrier trunk, speaking SIP over UDP, TCP or TLS, carrying
  G.711 audio over RTP or SRTP.
- **Ulai SFU rooms** — WebRTC sessions on the Ulai control plane, where browsers
  and AI agents meet.

A phone call that reaches the gateway becomes an ordinary participant in a
room. Everything the gateway does is in service of that one sentence: decide
whether the call is allowed, get a room for it, answer it, and move audio in
both directions until somebody hangs up.

It handles both directions of origination:

| Direction | Trigger | The gateway is |
| --- | --- | --- |
| Inbound | A carrier sends an `INVITE` to the SIP listener | the answering party (UAS) |
| Outbound | `POST /sip/originate` with a number and a trunk id | the calling party (UAC) |

## What it is not

- **Not an agent.** The gateway never talks to a model, never decides what to
  say, and has no notion of a conversation. It publishes `CALL_ANSWERED` and
  lets the platform's dispatcher summon whatever should join the room.
- **Not a media processor.** No noise suppression, no VAD, no recording, no
  playback accounting. It transcodes and it forwards.
- **Not a rate limiter.** It dials on every `POST /sip/originate`. If a trunk
  answers `429`, the caller upstream is the one that has to slow down.
- **Not a routing database.** Numbers, trunks, ACLs and dispatcher rules live in
  the platform's routing store; the gateway only reads them.

## Where it sits

```text
   carrier / ITSP                gateway                  Ulai platform
  ┌───────────────┐        ┌──────────────────┐        ┌──────────────────┐
  │ SIP trunk     │──────▶ │ SIP listener     │        │ control plane    │
  │ (udp/tcp/tls) │ INVITE │  :5060           │──────▶ │  create / join / │
  │               │        │                  │  HTTPS │  terminate       │
  │               │◀─────  │ RTP :10000-10500 │        └──────────────────┘
  └───────────────┘  RTP   │                  │        ┌──────────────────┐
                           │ HTTP API :8082   │──────▶ │ routing store    │
                           │  /sip/originate  │  Redis │  (sip-resolver)  │
                           │  /calls /health  │        └──────────────────┘
                           │                  │        ┌──────────────────┐
                           │ WebRTC / Opus    │──────▶ │ SFU room         │
                           └──────────────────┘  SRTP  │  agent, browsers │
                                                       └──────────────────┘
```

Three external dependencies, all required:

1. **The control plane** (`ULAI_CONTROL_PLANE_URL`) creates rooms, issues join
   tickets, and feeds the roster and lifecycle events for a session.
2. **The routing store** (`SIP_ROUTING_REDIS_URL`) is read through
   [`ulai-sip-resolver`](https://github.com/ulaidotin/ulai-sip-resolver). It is
   the authority on which project owns a number, which source IPs a trunk
   accepts, which dispatcher rule matches, and how to reach an outbound trunk.
   It is also where telecom events are published.
3. **The carrier**, reachable at `SIP_PUBLIC_IP`.

## Why a separate service

The SIP stack and the room gateway are deliberately separate packages that
share no dependencies: `internal/telephony` links no WebRTC, and
`internal/sfugw` links no SIP. The only native dependency in the whole binary
is **libopus**, because there is no production-grade pure-Go Opus encoder. Every
other piece of the audio path — resampling, μ-law/A-law, RTP, SDP, SRTP — is
pure Go.

## Where to go next

- [Getting started](/docs/sip/getting-started/) — get a call through it.
- [Concepts](/docs/sip/concepts/) — how a call actually flows.
- [Configuration](/docs/sip/configuration/) — the environment it needs.
