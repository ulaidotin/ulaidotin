---
title: Troubleshooting
description: Symptom first, cause second.
weight: 4
---

## The gateway will not start

```text
sip-sfu-gateway: ULAI_CONTROL_PLANE_URL is required (e.g. https://stgcp.ulai.co.in)
SIP_PUBLIC_IP must be a routable address, not 0.0.0.0 — it is advertised to the carrier
```

Configuration is validated once, and every problem is reported together — fix
them all, then restart. `0.0.0.0` in `SIP_PUBLIC_IP` is rejected on purpose: it
is not a listen address, it is what the carrier is told to stream media to.

Other startup failures:

| Message | Cause |
| --- | --- |
| `SIP_TLS_CERT_PATH and SIP_TLS_KEY_PATH are required when SIP_TRANSPORT=tls` | TLS transport with no certificate |
| `SIP_RTP_PORT_LOW/HIGH (…) must be a valid ascending port range` | Inverted or out-of-range ports |
| `routing service: …` | The Redis URL is malformed — this one *does* fail startup |
| `SIP runtime init: …` | The SIP port is already bound, or the TLS material is unreadable |

## Calls are rejected

The SIP status tells you which check failed:

| Status | Meaning | Where to look |
| --- | --- | --- |
| `403 Forbidden` | Source IP not in the trunk's ACL | The carrier's signalling address, and whether you are behind a Docker port proxy — see [networking](/docs/sip/operations/networking/#port-publishing-versus-host-networking) |
| `404 Not Found` | Unknown SIP domain, unassigned number, or no project config | The routing store: is the domain mapped, is the number assigned |
| `480 Temporarily Unavailable` | Routed, but no dispatcher rule matched | The project's dispatcher rules |
| `500 Server Internal Error` | The routing lookup itself failed | Redis connectivity — `WARNING: routing store unreachable` will be in the log |
| `503 Session unavailable` | The control plane would not create a room | The `[control-plane]` wire log |
| `503 Shutting down` | Drain in progress | Deploy timing |

Every rejection is logged with the reason immediately before it:

```text
[+919363399639] [sip-gw] rejecting 403: source IP not allowed by the trunk ACL: …
```

## The call connects but there is no audio

Almost always the media path, not the signalling path:

1. **The RTP range is not open.** 5060 allowed, 10000–10500 blocked, is the
   classic. Check the firewall and the security group.
2. **`SIP_PUBLIC_IP` is wrong.** The carrier is streaming to whatever the SDP
   said. Check it against the host's actual routable address.
3. **Port publishing instead of host networking.** Media arrives at the proxy
   rather than the process.
4. **One-way only?** That is usually NAT on the carrier's side. The gateway
   already latches onto the observed source address; if it is still one-way,
   the outbound direction is being dropped upstream.

## Outbound calls fail

| Response | Cause |
| --- | --- |
| `401` | No `Authorization: Bearer` header, or a malformed one |
| `400 project_id is required (or set ULAI_PROJECT_ID)` | Neither the body nor the environment names a project |
| `404` | That `trunk_id` does not exist in that project |
| `502` | Room creation failed, or the stored trunk address is malformed |

If the `202` comes back but the phone never rings, the failure is in the
background half and only the log has it:

```text
[+919363399639] [sip-originate] Originate failed: …
[+919363399639] [sip-originate] RATE LIMITED by trunk … (SIP 429, Retry-After 30s) — this gateway does not pace originations; the caller must slow down
```

The room created for that call is terminated rather than left running.

### `488 Secure media required`

A carrier with secure trunking enabled answered a plain `RTP/AVP` offer. The
gateway infers SRTP from the trunk's transport — TLS trunks offer it, UDP and
TCP do not — so either fix the stored transport, or force it per request with
`"offer_srtp": true`. See
[routing](/docs/sip/concepts/routing/#media-encryption-is-inferred-from-transport).

### `404` from the trunk on a self-hosted SBC

A dialplan matching literal digit patterns will not match a leading `+`. Send
`"dial_verbatim": true`. Hosted ITSPs want the opposite — E.164 with the plus.

## A call never ends

If the far end's `BYE` is lost — a TLS trunk calling back a UDP-only listener
never reaches us at all — the call and its room would stay up until the process
exits. `SIP_RTP_TIMEOUT_SECONDS` (default 30) is the backstop: a call whose
inbound audio stops for that long is ended as if the far end had hung up. It is
paused while the call is on hold. Setting it to `0` disables the backstop
entirely, which is rarely what you want.

## The agent talks to a ringtone

It should not — the gateway joins the room only *after* the callee answers,
precisely so a participant appearing in the roster is a reliable cue. If it
happens, something else joined the room early; the gateway's own join is logged
as `call answered — … joining room …`.

## The agent keeps talking after the caller hangs up

The session is not being terminated with the call. A room the gateway created
is always torn down; a room it was pointed at (`X-Session-Id`, or `session_id`
on originate) is torn down too *unless*
`SIP_TERMINATE_SESSION_ON_HANGUP=false`. Check that variable first.

## Duplicate agents on one call

A mid-call re-INVITE handled as a new call produces exactly this — a second
agent, a second model session, a second billing row. In-dialog INVITEs are
handled separately for that reason. If you see it, capture the SIP flow and
check whether the second invocation followed a re-INVITE from the carrier's SBC.
