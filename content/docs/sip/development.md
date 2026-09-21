---
title: Development
description: Repo layout, tests, and how to change it.
weight: 7
---

## Layout

```text
.
├── main.go                  process wiring: config, runtime, HTTP + SIP listeners, drain
├── gateway.go               the gateway type, call lifetime, room join, the bridge
├── inbound.go               OnInvite — the UAS path
├── outbound.go              POST /sip/originate and dialOut — the UAC path
├── routing.go               resolver lookups, reject statuses, event publishing
├── calls.go                 the live-call table behind GET /calls
├── httpapi.go               routes, JSON helpers, bearer parsing
├── cplog.go                 redacting wire log of control-plane requests
├── provider_adapter.go      telephony.TelephonyProvider ➜ sfugw.Provider
├── internal/
│   ├── telephony/           SIP, SDP, RTP, SRTP, jitter, DTMF, re-INVITE
│   ├── sfugw/               WebRTC room gateway and the codec bridge
│   ├── ulaievents/          roster and lifecycle feed from the control plane
│   └── utils/               phone normalisation, env parsing, runtime tuning
├── pkg/
│   ├── config/              every environment knob, resolved and validated once
│   └── api/                 the HTTP listener, with its timeouts and drain
├── Dockerfile.sip
├── docker-compose.yml
└── health-check.sh
```

## Tests

```sh
go test ./...
go test -race ./...
go test -run TestOriginate -v .
```

The call paths are testable without a control plane or a Redis because of two
deliberate seams:

- `gateway.plane` is a **field**, not a method, so a test can stand a fake in
  front of the control plane.
- `gateway.dial` is a `dialer` func, defaulting to `SIPRuntime.Originate`, so a
  test can answer a call without ringing a real trunk.
- `router` is an interface over the resolver, so routing decisions can be
  stubbed.

`pkg/config` reads the environment exactly once into a struct; everything
downstream reads the struct, so a test constructs a `Config` directly rather
than setting environment variables.

## Conventions worth keeping

- **Errors are reported all at once.** Config validation and request validation
  both use `errors.Join`, because one complaint per round trip is miserable to
  integrate against.
- **Comments explain *why*.** The files in this repo carry unusually long
  header comments, and they document decisions that cost someone a production
  incident — the re-INVITE handling, the RTP source gate, the jitter buffer's
  refusal to add fixed latency. Read them before changing the code they sit on.
- **Secrets never reach the log whole.** `cplog.go` masks credentials, and the
  SIP trace helpers redact `Authorization`, `WWW-Authenticate` and `a=crypto`
  lines. Anything new that logs a message must go through them.
- **The two halves stay apart.** `internal/telephony` must not import
  pion/webrtc; `internal/sfugw` must not import sipgo. Go links packages whole,
  and this split is what keeps the binary — and the image — small.

## Branches and deployment

| Branch | Environment |
| --- | --- |
| `main` | Default development branch |
| `staging` | Deployed to the staging VM on push |
| `prod` | Deployed to production on push |

`.github/workflows/deploy.yml` SSHes to the VM, pulls the branch, writes `.env`
from repository secrets, rebuilds the image, restarts compose, and verifies both
the HTTP and SIP legs before declaring success.

## Adding a telephony provider

`telephony.TelephonyProvider` is the contract a carrier leg implements;
`sfugw.Provider` is the subset a room gateway consumes. A new provider
implements the first, and `provider_adapter.go` shows how the two are bridged —
translating events and dropping the ones a pure audio gateway has no use for.

The audio contract is fixed at μ-law 8 kHz mono, 20 ms frames, in both
directions. Everything above the wire boundary assumes it.
