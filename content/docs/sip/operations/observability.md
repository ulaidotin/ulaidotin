---
title: Observability
description: Logs, health checks, and the control-plane wire log.
weight: 3
---

## Reading the log

Every line of a call is prefixed with the number and the leg it arrived on, so
a busy gateway's log can be read one call at a time:

```text
[+919363399639] [sip-originate] call answered — call_id=3e114ad4…, joining room 527f78a3…
[+919363399639] [sfugw] webrtc state: connected
[+919363399639] [sfugw] DTMF: 5
[+919363399639] [sip-originate] call finished (session 527f78a3…, call_id 3e114ad4…)
```

| Prefix | Emitted by |
| --- | --- |
| `[sip-gw]` | The inbound path |
| `[sip-originate]` | The outbound path |
| `[sfugw]` | The room/WebRTC bridge |
| `[control-plane]` | The control-plane wire log |
| `[runtime]` | Startup runtime tuning |

`grep` on the number gives one call; `grep` on the `sip_call_id` joins the
gateway's log to carrier CDRs and to the published telecom events.

## Health checks

There are two legs, and `/health` only covers one:

```sh
curl -fsS http://127.0.0.1:8082/health
sipsak -s sip:healthcheck@127.0.0.1:5060
```

`health-check.sh` in the repo runs both and exits non-zero if either fails.
It is what `docker-compose.yml` and the CI verification step use. Only one
`HEALTHCHECK` can be active per image, which is why both checks live in one
script.

`/health` returns `200` whenever the process is serving. It does **not** assert
that the routing store is reachable — that is reported at startup and again on
every call that fails because of it.

## Live calls

```sh
watch -n2 'curl -s localhost:8082/calls | jq'
```

The list is sorted oldest-first, so rows do not reshuffle between polls. It is
also the number the drain reports:

```text
drain deadline reached with 3 call(s) still up — cancelling them
```

## The control-plane wire log

Every request the gateway makes to the control plane — create, join, terminate
— is logged with its headers and body:

```text
[control-plane] → POST https://stgcp.ulai.co.in/api/v1/sessions headers={Authorization: Bearer ulai…kzM (51 chars), Content-Type: application/json, X-System-Secret: ahd8…3dg (26 chars)} body={"max_participants":8,…}
[control-plane] ← POST /api/v1/sessions 201 Created (128ms)
```

It exists because "what did we actually send?" is a question best answered by
the log, not by reading the SDK — which matters most when the answer is a `403`
and the question is which header was missing.

Three rules it follows:

- **Credentials are masked**, never printed whole: first and last four
  characters plus the length. Enough to tell *which* secret went out without
  putting it in every log shipper that reads this process's output.
- **Auth headers are always reported**, present or not. For an auth failure,
  `Authorization: <not sent>` is the single most useful thing a log can say.
- **Only failure bodies are logged.** A success body carries the join ticket and
  events token, which are credentials in their own right; a failure body is the
  control plane's explanation.

## Events

`CALL_ANSWERED` and `CALL_HANGUP` are published to the platform's telecom
stream for every answered call, carrying the routing decision, the room, the
numbers and — on hangup — `duration_seconds`. They are the record to build
dashboards and alerting on; the gateway keeps no history of its own beyond the
live table. See [routing](/docs/sip/concepts/routing/#events).

## Startup lines worth alerting on

```text
routing store reachable
WARNING: routing store unreachable (...) — every call will be rejected until it recovers
```

The gateway deliberately starts either way: a store that is down now may be up
a second from now, and a SIP listener answering an honest `500` is worth more
than one that is not there at all.
