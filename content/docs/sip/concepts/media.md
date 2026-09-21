---
title: Media pipeline
description: Codecs, SRTP, jitter, DTMF, and the ways a call ends without a BYE.
weight: 3
---

## The audio path

The carrier speaks 8 kHz G.711. The SFU speaks 48 kHz Opus. Everything between
the two is `internal/sfugw`:

```text
uplink   caller μ-law 8k ─decode─▶ PCM 8k ─resample─▶ PCM 48k ─encode─▶ Opus ─▶ room
downlink room Opus 48k ─decode─▶ PCM 48k ─resample─▶ PCM 8k ─mix─▶ μ-law ─▶ caller
```

Both directions run on a 20 ms cadence: 160 μ-law bytes on the wire, 960
samples per Opus frame. Opus is encoded at 24 kbit/s with complexity 5 —
wideband speech without maxing out a CPU that may be carrying many concurrent
calls.

The downlink **mixes**: a room can hold several publishers, and the caller gets
all of them summed into one stream. Resampling is `go-audio-resampler` (pure
Go, SIMD-enabled); μ-law and A-law conversion is `zaf/g711`. The only native
code in the binary is libopus.

## Codec negotiation

The SDP profile is deliberately narrow — the audio pipeline is μ-law 8 kHz, so
accepting anything else would be a lie:

- **PCMU** (μ-law, PT 0) or **PCMA** (A-law, PT 8). When the carrier negotiates
  A-law, the RTP loops transcode A-law↔μ-law at the wire boundary so everything
  above stays μ-law.
- **telephone-event** (RFC 4733 DTMF) on whatever payload type the carrier
  assigns.
- No Opus on the SIP side, no video, no multiple `m=` lines.

### SRTP

SDES only, with the two common profiles: `AES_CM_128_HMAC_SHA1_80` (Twilio's
default) and `_32`. Whether it is offered is decided by the trunk's transport —
see [routing](/docs/sip/concepts/routing/#media-encryption-is-inferred-from-transport).

## Inbound RTP: reorder, dedupe, conceal

A conventional jitter buffer imposes a fixed delay on every packet. For a voice
agent that delay is charged to time-to-first-token on *every* turn, including
the overwhelming majority where the network was perfectly ordered. So the
inbound buffer is not a fixed-delay design:

| Packet arrives | What happens |
| --- | --- |
| In sequence | Emitted immediately, zero added latency |
| Out of order | Held only until the gap resolves, bounded by a 40 ms holdout |
| Duplicated | Dropped |
| Never | Concealed after the holdout, stream continues |

Concealment repeats the previous frame, attenuated, decaying to silence over a
few frames. Repeating preserves the spectral envelope so an ASR hears a brief
smear rather than the click-and-jump digital silence produces; decaying stops a
lost burst becoming an audible buzz.

The cost is paid only by calls that actually have a disordered network.

## Outbound RTP: a small playout cushion

The provider's writer and the RTP writer are two independent 20 ms tickers.
Without a cushion, ordinary scheduler jitter forces silence into the middle of
speech. The playout buffer builds 60 ms (3 frames) before starting and caps
added latency at 160 ms (8 frames), dropping the oldest beyond that.

## Symmetric RTP and the source gate

Carriers behind NAT routinely send RTP from a port they never advertised in
SDP, so the first well-formed packet **latches** the peer address and the writer
re-targets to it. Everything after that is checked against the latch.

That check matters because the RTP port range is a few hundred even ports cycled
round-robin: a call that ends while its carrier is still streaming leaves
packets in flight that land on whichever call binds that port next. Before the
gate existed, they were decoded and mixed into a live conversation as a second
voice.

A genuine media re-anchor — a B2BUA leg swap, an SBC failover — is admitted
only after the new source proves persistence (5 packets over at least 200 ms).
A re-INVITE can pre-authorise an address, but the previous peer stays valid
until the new one actually speaks: a re-INVITE is an intention, and a peer that
never follows through must not be able to mute a working call.

## Re-INVITE, hold, and session timers

sipgo's `OnInvite` fires for every INVITE, including in-dialog ones. Handing
those to the application handler treats a mid-call re-INVITE — a session-timer
refresh, a hold, a media re-anchor, an SBC failover — as a brand new call:
a second agent, a second billing row, a `180 Ringing` inside an established
dialog, a `200 OK` with a new `To` tag (a protocol violation), and an answer
advertising a new RTP port, so the carrier moves media to a socket nobody reads.
Dead air for the rest of the call.

In-dialog INVITEs are therefore handled separately, and answered as a
*re-statement* rather than a negotiation: same `To` tag, same RTP port, same
codec, session id unchanged with its version bumped, and the direction attribute
mirrored so hold is acknowledged rather than contradicted. The only thing that
may legitimately change is where the peer wants media — and that is followed.

A re-INVITE that tries to switch G.711 flavour mid-call is refused rather than
silently answered with a lie: the provider's transcode setting is fixed when it
is constructed.

Session timers (`timer`) are supported; an INVITE that `Require`s an extension
the gateway does not support is rejected rather than answered.

## DTMF

RFC 4733 `telephone-event` packets are decoded and surfaced as events. The
gateway logs them (`DTMF: 5`) and does nothing else with them — it has no IVR of
its own. An agent in the room that wants digits should consume the session
feed, not expect the gateway to act.

## Ending a call without a BYE

A hangup's `BYE` can be lost. A TLS trunk calling back a UDP-only listener never
reaches the gateway at all, and without a backstop such a call — and its room —
stays up until the process exits.

`SIP_RTP_TIMEOUT_SECONDS` (default 30) ends a call whose **inbound** audio has
stopped for that long, as if the far end had hung up. It is paused while the
call is on hold, so a legitimately silent leg is not cut off. Set `0` to
disable.

The other end-of-call signals:

| Signal | Source |
| --- | --- |
| `BYE` | The far end, normally |
| `session_terminated` | The control plane, over the session events feed — the authoritative "this call is really over" |
| Transport `Done()` | The WebRTC leg dropping, which may be a transient blip the SDK reconnects through |
| Drain cancellation | Shutdown, after the 60 s drain window |
