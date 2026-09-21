---
title: SIP Gateway
linkTitle: SIP Gateway
description: SIP trunks in, Ulai SFU voice rooms out.
weight: 1
---

`ulai-sip-module` is a single Go binary (`sip-sfu-gateway`) that puts a phone
call into an Ulai SFU room. A carrier's `INVITE` is routed, admitted and
answered; an API call dials out through a trunk. Either way the caller ends up
as an ordinary WebRTC participant talking to whoever else is in the room.

Source: [github.com/ulaidotin/ulai-sip-module](https://github.com/ulaidotin/ulai-sip-module)

## Run it

```sh
docker run -d --name ulai-sip \
  -e ULAI_CONTROL_PLANE_URL=https://stgcp.ulai.co.in \
  -e SIP_ROUTING_REDIS_URL='redis://USER:PASSWORD@redis.ulai.co.in:6379' \
  -e SIP_PUBLIC_IP=148.113.58.51 \
  -e SIP_LISTEN_ADDR=0.0.0.0:5060 \
  -e SIP_TRANSPORT=udp \
  -e SIP_RTP_HOST=0.0.0.0 \
  -e SIP_RTP_PORT_LOW=10000 \
  -e SIP_RTP_PORT_HIGH=10500 \
  -e SIP_HTTP_PORT=8082 \
  --network host \
  asia-south1-docker.pkg.dev/arctic-operand-415316/ulai/sip:v6
```

`--network host` is not optional — see
[networking](/docs/sip/operations/networking/).

## Dial out

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

## In this section

- **[Overview](/docs/sip/overview/)** — what it does, what it does not do, and
  where it sits in the platform.
- **[Getting started](/docs/sip/getting-started/)** — prerequisites, build,
  configuration, and your first inbound and outbound call.
- **[Concepts](/docs/sip/concepts/)** — architecture, call flows, the routing
  plane, and the media pipeline.
- **[Configuration](/docs/sip/configuration/)** — every environment variable and
  built-in timeout.
- **[HTTP API](/docs/sip/http-api/)** — `/health`, `/calls` and
  `/sip/originate`.
- **[Operations](/docs/sip/operations/)** — deployment, networking,
  observability and troubleshooting.
- **[Development](/docs/sip/development/)** — repo layout, tests, and how to
  change it.
