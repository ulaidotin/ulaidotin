---
title: Routing plane
description: How a call is admitted, how a trunk is resolved, and what gets published.
weight: 2
---

Everything the gateway knows about numbers, trunks and agents comes from
[`ulai-sip-resolver`](https://github.com/ulaidotin/ulai-sip-resolver) over the
platform's Redis. The gateway holds no routing table of its own.

```text
inbound  : (host, dialled number, source IP)  →  project, trunk ACL, dispatcher rule
outbound : (project, trunk id)                →  address, transport, digest auth
```

Neither leg carries trunk details on the wire. Callers used to send host, port,
transport and credentials on every origination, which put carrier passwords in
every dialler's logs and meant a trunk migration had to be rolled out to each of
them.

## Inbound admission

`Resolve(host, dialledNumber, sourceIP)` is the whole admission decision. It
returns the trunk the call came in on and the dispatcher rule that won, or an
error that maps to a specific SIP status:

| Resolver error | Status | Means |
| --- | --- | --- |
| `ErrHostNotFound` | `404` | No project owns that SIP domain |
| `ErrNumberNotAssigned` | `404` | The domain is known, the number is not |
| `ErrConfigNotFound` | `404` | The project has no SIP configuration |
| `ErrIPNotAllowed` | `403` | The INVITE came from an address the trunk's ACL does not list |
| `ErrNoMatchingRoute` | `480` | Admitted, but no dispatcher rule matched the number |
| anything else | `500` | The lookup itself failed |

Each one is a different operational problem and deserves to be distinguishable
from the carrier's side.

The **project id** is a second, separate lookup (`ulai:sip_host:<host>`), because
the resolver's `Result` does not carry it. That lookup is best-effort: if it
fails the call still proceeds, logged as `continuing without a project id`, and
the published events lose a field.

## Session metadata

A room created for an inbound call carries the routing decision in its metadata,
which rides the control plane's discovery feed. An orchestrator that has never
heard of the call can read it and know which agent the room is waiting for:

| Key | Source |
| --- | --- |
| `project_id` | The host lookup |
| `trunk_id`, `trunk_name` | The resolved trunk |
| `rule_id`, `rule_name`, `rule_type` | The dispatcher rule |
| `agent_name` | The rule, when it names one |
| `dispatch_metadata` | The rule's free-form metadata, JSON-encoded |
| `name`, `phone_number`, `dialled`, `sip_call_id` | The call |
| `direction` | `inbound` |
| `source` | `sip-sfu-gateway` |

Empty fields are **left out** rather than written blank. Several of them
routinely are — rules are stored keyed by id with no id inside the value — and a
dispatcher can act on an absent `rule_id` where it cannot tell a blank one from
a real empty answer.

An outbound room's metadata is the same idea with the outbound fields:
`phone_number`, `caller_id`, `name`, `agent_id`, `project_id`, `trunk_id`,
`trunk_name`, `direction: outbound`, `source`.

## Outbound trunks

`GetOutboundTrunk(projectID, trunkID)` returns a stored trunk, which the gateway
reduces to dial parameters:

- **Address** — accepted as `host`, `host:port`, or either with a `sip:`/`sips:`
  scheme, userinfo or URI parameters attached. Port `0` means unspecified, and
  the dialler fills in `5061` for TLS or `5060` otherwise.
- **Transport** — `udp`, `tcp`, `tls`, or empty (treated as UDP). Anything else
  is an error rather than a silent fallback: dialling a TLS-only trunk over UDP
  fails as a timeout minutes later, which is a miserable way to learn about a
  typo.
- **Credentials** — digest username and password, empty when the trunk
  authenticates by IP ACL.

### Media encryption is inferred from transport

The stored trunk carries a transport but no media-encryption field, so SRTP
policy is derived:

| Trunk transport | Offer SRTP | Require SRTP |
| --- | --- | --- |
| `tls` | yes | no |
| `udp`, `tcp` | no | no |

This is not cosmetic. A carrier with secure trunking enabled — Twilio's is, on a
TLS trunk — answers an `RTP/AVP` offer with `488 Secure media required` and the
call never rings.

Offering without requiring is the safe half: a carrier that wants SRTP finds
crypto in the offer; one that does not echoes no `a=crypto` and the call falls
back to plain RTP. Override per request with `offer_srtp` and `require_srtp` —
including `offer_srtp: false` to force cleartext media on a TLS trunk.
`require_srtp` implies `offer_srtp`; asking for the contradictory pair
(`offer_srtp: false, require_srtp: true`) is a `400`.

## Events

Two event types are published to the platform's telecom stream, through the same
resolver that authorised the call:

| Type | When | Notable metadata |
| --- | --- | --- |
| `CALL_ANSWERED` | Immediately after the call is answered, before the room is joined | routing fields, `session_id`, `sip_call_id`, `direction`, `to_number`, `from_number` |
| `CALL_HANGUP` | When the bridge ends | the same, plus `duration_seconds` |

Publishing is bounded at 2 seconds and best-effort: an event is never allowed to
hold up a call, and a miss is logged and nothing more. The hangup publish is
detached from the call's context on purpose — that context is being cancelled at
precisely the moment the event matters most.

## Startup probe

go-redis connects lazily, so without a probe the first sign of a dead routing
store would be a carrier receiving a `500`. At startup the gateway looks up a
host no project can own and reports what happened:

```text
routing store reachable
WARNING: routing store unreachable (...) — every call will be rejected until it recovers
```

It does **not** fail startup. A malformed URL already did that; a store that is
down now may be up a second from now.
