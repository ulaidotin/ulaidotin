---
title: Architecture
description: The packages, and what each one is allowed to know about.
weight: 1
---

## The import graph is the design

The binary is `package main` at the repo root, with three internal packages and
two public ones underneath it:

```text
main  (gateway.go, inbound.go, outbound.go, routing.go, httpapi.go, calls.go, cplog.go)
  │
  ├── internal/telephony   SIP + RTP.     Links sipgo, pion/srtp.  NO WebRTC.
  ├── internal/sfugw       Room + WebRTC. Links pion/webrtc, Opus. NO SIP.
  ├── internal/ulaievents  Roster and lifecycle feed from the control plane.
  ├── internal/utils       Phone normalisation, env parsing, runtime tuning.
  ├── pkg/config           Every environment knob, resolved and validated once.
  └── pkg/api              The HTTP listener, with its timeouts and drain.
```

The two halves never meet. `internal/telephony` does not import pion/webrtc,
and `internal/sfugw` does not import sipgo. That is not tidiness — Go links
packages whole, and an earlier version of this gateway pulled RNNoise and a VAD
shared object into an image that filtered nothing, purely because the SIP stack
lived in a package that also held them.

`provider_adapter.go` in `main` is the seam: it translates a
`telephony.TelephonyProvider` into the narrower `sfugw.Provider` the room
gateway consumes, dropping the events a pure gateway has no use for (playback
marks).

## The three files that are one call

Everything call-shaped lives in `main`:

| File | Holds |
| --- | --- |
| `gateway.go` | The `gateway` type, call lifetime and drain, room join, the bridge |
| `inbound.go` | `OnInvite` — the UAS path |
| `outbound.go` | `POST /sip/originate` and `dialOut` — the UAC path |
| `routing.go` | Resolver lookups, reject statuses, event publishing |
| `calls.go` | The live-call table behind `GET /calls` |
| `httpapi.go` | Routes, JSON helpers, bearer parsing |
| `cplog.go` | A redacting wire log of every control-plane request |

## Per-call credentials, not per-process

There is no single control-plane client. `gateway.plane(apiKey)` builds one per
call, bound to the key that call authenticated with:

- **Outbound** uses the `Authorization: Bearer` header from
  `POST /sip/originate`. The room belongs to the project that key identifies, so
  one gateway can serve several projects without a line of configuration.
- **Inbound** uses the *empty* key. An `INVITE` carries no credential; admission
  is the resolver's decision, made from the trunk the call arrived on and the
  source IP that trunk's ACL allows. The control plane sees the call
  unauthenticated.

Building a client per call is cheap: a base URL, a key, and an `http.Client`
with no transport of its own, so every call still shares one connection pool.

## Call lifetime and the drain

Every call runs on `gateway.callCtx`, not on the context its trigger arrived
with. sipgo hands `OnInvite` a `context.Background()` that says nothing about
how long the call may run, and an HTTP request context dies the moment the
`202` is written — neither is a call's lifetime.

`enterCall` registers a call and refuses to start one once shutdown has begun;
`goCall` wraps that around a goroutine. On `SIGINT`/`SIGTERM`:

1. The HTTP listener stops accepting.
2. New calls are refused — `503` on both legs.
3. In-flight calls are given `DrainTimeout` (60 s) to finish on their own.
4. Whatever is left is cancelled, then given `CancelGrace` (5 s).
5. The routing store connection is closed **last**, so a draining call's hangup
   event still has somewhere to go.

That bookkeeping is what makes the `draining for up to 60s` log line true rather
than decorative.

## Rooms the gateway owns, and rooms it borrows

A call either lands in a room the gateway created for it, or one it was pointed
at (`X-Session-Id` on the INVITE, `session_id` in the originate body). The
difference matters exactly once, at hangup:

- A room the gateway **created** is always terminated with the call. Nobody else
  is in it.
- A room it was **pointed at** is terminated too, by default, because every
  deployment this gateway serves is 1:1 — leaving it up strands the agent in a
  live session after the caller has gone. Set
  `SIP_TERMINATE_SESSION_ON_HANGUP=false` if you genuinely share rooms.

Terminating the session, rather than just leaving the room, is what lets the
agent tell a hangup apart from a media blip on the gateway's side.
