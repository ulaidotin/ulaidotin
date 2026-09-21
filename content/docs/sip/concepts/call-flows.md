---
title: Call flows
description: The ordered steps of an inbound and an outbound call — and why the orders differ.
weight: 1
---

Both legs end in the same place: a `bridgedCall` handed to `sfugw.Run`, which
moves audio until one side stops. Everything before that differs.

## Inbound

The gateway is the answering party. `telephony.SIPRuntime` invokes
`gateway.OnInvite` for every out-of-dialog `INVITE`.

```text
 carrier            gateway                 resolver        control plane
   │  INVITE ────────▶ │                        │                 │
   │                   │ 1. Resolve ───────────▶│                 │
   │                   │    (host, number, IP)  │                 │
   │  ◀── 4xx/5xx ──── │ ◀── reject ────────────│                 │
   │                   │                        │                 │
   │                   │ 2. CreateSession ──────┼────────────────▶│
   │  ◀── 200 OK ───── │ 3. Accept                                │
   │  ──── ACK ──────▶ │                                          │
   │                   │ 4. PublishEvent CALL_ANSWERED ─▶ resolver│
   │                   │ 5. JoinSession ────────┼────────────────▶│
   │  ◀═══ RTP ══════▶ │ ◀════════ bridge ══════╪═════ WebRTC ═══▶│
```

1. **Route.** `(host, dialled number, source IP)` → project, trunk ACL,
   dispatcher rule. A rejection here is a *specific* SIP status the carrier can
   act on, not a blanket `503`. See [routing](/docs/sip/concepts/routing/).
2. **Room.** An `X-Session-Id` header on the INVITE wins if present; otherwise
   a session is created, carrying the dispatcher rule in its metadata so the
   platform can see what the room is for. The header only chooses the room — it
   cannot overrule admission.
3. **Answer.** `200 OK` + `ACK`, which binds RTP.
4. **Tell.** `CALL_ANSWERED` is published *before* the WebRTC handshake, not
   after. That event is what summons the agent, so the dispatcher gets to work
   while the gateway is still joining.
5. **Join and bridge.** A seat is taken in the room, then audio flows.

The room is joined only **after** the call is answered. On this path "accepted"
*is* the pickup, so no seat is taken until there is a live call to put in it —
a join ticket is only valid for about 30 seconds, and a participant appearing in
the room is the agent's cue to start talking.

### Rejections

| Condition | SIP status |
| --- | --- |
| Empty `To` user | `400 Bad Request` |
| Gateway is shutting down | `503 Shutting down` |
| Unknown domain, unassigned number, missing config | `404 Not Found` |
| Source IP not in the trunk's ACL | `403 Forbidden` |
| No dispatcher rule matched | `480 Temporarily Unavailable` |
| Routing lookup failed | `500 Server Internal Error` |
| Session could not be created | `503 Session unavailable` |

The project-id lookup is deliberately best-effort: a call that routed cleanly is
not dropped because that second lookup missed. The dispatcher loses a field, the
caller keeps their call.

## Outbound

The gateway is the calling party. `POST /sip/originate` splits into a
synchronous half the caller can act on and an asynchronous half it cannot.

```text
 caller              gateway                 resolver        control plane
   │ POST ──────────▶ │                         │                 │
   │                  │ auth: Bearer <ulai key> │                 │
   │                  │ 1. GetOutboundTrunk ───▶│                 │
   │ ◀── 404/502 ──── │ ◀── not found ──────────│                 │
   │                  │ 2. CreateSession ───────┼────────────────▶│
   │ ◀── 202 ──────── │    (unless session_id given)              │
   │   {session_id}   │                                           │
   ╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌ │ ╌╌ background ╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌ │
   │                  │ 3. INVITE ──▶ trunk, block through ring    │
   │                  │ 4. PublishEvent CALL_ANSWERED ─▶ resolver  │
   │                  │ 5. JoinSession ─────────┼────────────────▶│
   │                  │ ◀════════ bridge ═══════╪═════ WebRTC ═══▶│
```

1. **Trunk first.** An unknown `trunk_id` must not leave an orphan room behind,
   so the trunk is resolved before anything is created.
2. **Room.** Created unless the request carried a `session_id`. Its metadata
   carries the number, the trunk and the `agent_id`, so an orchestrator watching
   the control plane's discovery feed can be ready before the phone is answered.
3. **Dial.** `Originate` blocks through ring, up to `OriginateTimeout` (60 s).
   The dial context stays alive for the whole call — sipgo builds the client
   transaction on it, so cancelling it at answer would pull the dialog out from
   under the call just connected.
4. **Tell**, then **5. join and bridge**, exactly as inbound.

Ringing takes up to 30 seconds and has nothing useful to say until it is over,
which is why it is not on the request path. Trunk lookup and room creation are
fast, can fail in ways the caller can act on, and produce a result the caller
needs — so they are.

### If the callee never answers

Busy, rejected, no answer, unreachable trunk — the room created for the call is
terminated, because anything already waiting in it deserves to be told rather
than left listening to silence. A `429` from the trunk is logged specially: it
is the one failure that is the platform's own fault, and it is fixed by dialling
slower, not by retrying.

## What both legs do at the end

`sfugw.Run` returns when the SIP leg drops, the room ends, or the context is
cancelled. Then:

- `CALL_HANGUP` is published, with `duration_seconds`, pairing the
  `CALL_ANSWERED` already sent. It is published on a context detached from the
  call's own, because the call's context is usually being torn down at exactly
  that moment.
- The call is removed from `GET /calls`.
- The session is terminated, unless the room was borrowed and
  `SIP_TERMINATE_SESSION_ON_HANGUP=false`.
