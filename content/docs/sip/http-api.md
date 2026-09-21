---
title: HTTP API
description: /health, /calls and /sip/originate.
weight: 5
---

Three endpoints, served on `SIP_HTTP_PORT` (default `8082`). Only
`/sip/originate` is authenticated.

| Method | Path | Auth |
| --- | --- | --- |
| `GET` | `/health` | none |
| `GET` | `/calls` | none |
| `POST` | `/sip/originate` | `Authorization: Bearer <ulai api key>` |

{{% alert title="These endpoints are not public" color="warning" %}}
`/health` and `/calls` are unauthenticated, and `/calls` lists live phone
numbers. Keep the HTTP port on a private network or behind a firewall; only
`/sip/originate` checks a credential.
{{% /alert %}}

## GET /health

```sh
curl -s http://148.113.58.51:8082/health
```

```json
{ "status": "ok", "service": "sip-sfu-gateway" }
```

Always `200` while the process is serving. It says nothing about the SIP leg or
the routing store — see [observability](/docs/sip/operations/observability/) for
a check that covers both.

## GET /calls

Every call currently bridged, oldest first.

```sh
curl -s http://148.113.58.51:8082/calls
```

```json
{
  "live": [
    {
      "to_number": "+919363399639",
      "from_number": "+13187184515",
      "name": "sip-module",
      "sip_call_id": "3e114ad4-a9ad-4a0b-97b9-5cf11190bdd8",
      "session_id": "527f78a3262dbe8b0d0a678a9e98ac29",
      "project_id": "fa3f23f8-a75d-44a4-84f0-6a1c828f579f",
      "direction": "outbound",
      "source": "originate",
      "started_at": "2026-09-21T10:14:52.118Z"
    }
  ]
}
```

| Field | Meaning |
| --- | --- |
| `to_number`, `from_number` | Normalised E.164 |
| `name` | Display name in the room roster |
| `sip_call_id` | The SIP `Call-ID` — the join key between logs, events and carrier CDRs |
| `session_id` | The Ulai room |
| `project_id` | Owning project; may be empty inbound if the host lookup missed |
| `direction` | `inbound` or `outbound` |
| `source` | `originate`, `resolver`, or the session header's name when a room was supplied on the INVITE |
| `started_at` | When the bridge started, RFC 3339 |

The list is sorted rather than map-ordered, so polling it does not reshuffle
rows on every request.

## POST /sip/originate

Places one outbound call. Authenticates with a **control-plane API key**, and
the room opened for the call belongs to the project that key identifies — which
is how one gateway serves several projects without per-project configuration.

```sh
curl -X POST http://148.113.58.51:8082/sip/originate \
  -H 'Authorization: Bearer ulai_live_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx' \
  -H 'Content-Type: application/json' \
  -d '{
    "to_number":        "+919363399639",
    "trunk_id":         "74958094-8c76-419f-aa42-d4eac6768ec2",
    "project_id":       "fa3f23f8-a75d-44a4-84f0-6a1c828f579f",
    "from_number":      "+13187184515",
    "name":             "sip-module",
    "max_participants": 8,
    "offer_srtp":       true,
    "require_srtp":     true,
    "dial_verbatim":    false,
    "session_id":       "527f78a3262dbe8b0d0a678a9e98ac29"
  }'
```

The smallest request that works — the gateway creates the room and infers SRTP
policy from the trunk's transport:

```sh
curl -X POST http://148.113.58.51:8082/sip/originate \
  -H "Authorization: Bearer $ULAI_API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{
    "to_number": "+919363399639",
    "trunk_id":  "74958094-8c76-419f-aa42-d4eac6768ec2",
    "project_id":"fa3f23f8-a75d-44a4-84f0-6a1c828f579f"
  }'
```

### Request body

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `to_number` | string | **yes** | Destination, E.164. Normalised before dialling. |
| `trunk_id` | string | **yes** | Outbound trunk within the project. The resolver turns it into an address, a transport and credentials — none of which are ever sent on the wire. |
| `project_id` | string | unless `ULAI_PROJECT_ID` is set | Scopes `trunk_id`. |
| `session_id` | string | no | Dial into a room that already exists. Omit it and the gateway creates one and returns its id. |
| `from_number` | string | no | Caller ID presented to the trunk. |
| `name` | string | no | Display name in the room roster. Defaults to `to_number`. |
| `agent_id` | string | no | Recorded in session metadata and visible on the control plane's discovery feed, so an orchestrator can tell which agent the room is waiting for. Unused by the gateway otherwise. |
| `max_participants` | int | no | Sizes a newly created room. Ignored when `session_id` is given. Defaults to `ULAI_MAX_PARTICIPANTS`. |
| `offer_srtp` | bool | no | Offer `RTP/SAVP` with an `a=crypto` line. Defaults to `true` on a TLS trunk, `false` otherwise. |
| `require_srtp` | bool | no | Fail the call if the answer carries no crypto. Implies `offer_srtp`. |
| `dial_verbatim` | bool | no | Send `to_number` without a leading `+`. For self-hosted or CUSTOM trunks whose dialplan matches literal digit patterns; hosted ITSPs want E.164-with-plus. |

Bodies are capped at 8 KiB. Validation reports every problem at once, so a
dialler does not get one complaint per round trip.

### 202 Accepted

```json
{
  "status": "originating",
  "to_number": "+919363399639",
  "session_id": "527f78a3262dbe8b0d0a678a9e98ac29",
  "created": false,
  "name": "sip-module",
  "trunk": "Main SBC"
}
```

`created` says whether the gateway made the room (and is therefore responsible
for tearing it down). The response returns as soon as the trunk is resolved and
the room exists — **the phone is still ringing**. Point an agent or a browser at
`session_id` while it does; the gateway itself joins the room only once the
callee answers.

### Errors

| Status | Cause |
| --- | --- |
| `400` | Malformed JSON, failed validation, or no `project_id` and no `ULAI_PROJECT_ID` |
| `401` | Missing or malformed `Authorization: Bearer` |
| `404` | No such trunk for that project, or no SIP config for the project |
| `502` | Room creation failed, or the stored trunk address is malformed |
| `503` | The gateway is shutting down |

```json
{ "error": "to_number is required\ntrunk_id is required" }
```

A `404` means the request asked for something that is not there; a `502` means
an upstream failed. Only one of those is worth retrying.

### What happens next

Nothing else comes back over HTTP. The call's progress shows up in three places:

- **The log**, prefixed with the number.
- **`GET /calls`**, once the bridge starts.
- **Telecom events** — `CALL_ANSWERED` then `CALL_HANGUP`, published to the
  platform's stream. See [routing](/docs/sip/concepts/routing/#events).

If the callee never answers, the room created for the call is terminated rather
than left running.

{{% alert title="No pacing" color="warning" %}}
The gateway dials on every `POST`. It has no origination rate limit of its own,
so a burst of `429`s from a trunk means the caller upstream has to slow down.
{{% /alert %}}
