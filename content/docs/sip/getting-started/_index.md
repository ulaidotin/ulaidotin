---
title: Getting started
description: Run the image and put a call through it.
weight: 2
---

The gateway ships as a container image. There is nothing to compile and nothing
to install on the host beyond Docker.

## What you need

| Requirement | Why |
| --- | --- |
| A host with a routable public IP | It is advertised to the carrier in SDP and `Contact`, and media is streamed to it |
| Docker, with host networking available | The RTP range cannot be published through a proxy — see [networking](/docs/sip/operations/networking/) |
| A control-plane URL | Rooms are created and joined there |
| A routing store URL (Redis) | Every call is resolved against it |
| A SIP trunk configured in the routing store | Inbound numbers, outbound termination, or both |
| A control-plane API key | Required for `POST /sip/originate` |

Ports that must be reachable from the carrier: **5060** (UDP, TCP or TLS) for
signalling and **10000–10500** (UDP) for media. Opening the first without the
second gives you calls that connect and then have no audio.

## Pull the image

```sh
docker pull asia-south1-docker.pkg.dev/arctic-operand-415316/ulai/sip:v6
```

If the registry is private, authenticate first:

```sh
gcloud auth configure-docker asia-south1-docker.pkg.dev
```

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

Three things in that command are load-bearing:

- **`--network host`** is required. Publishing the RTP range with `-p` spawns
  hundreds of userland proxies **and** makes every INVITE arrive from the bridge
  gateway's address, which the trunk's IP ACL then rejects with `403`.
- **`SIP_PUBLIC_IP`** must be the host's own routable address. It is what every
  SDP tells the carrier to stream media to, which is why `0.0.0.0` is rejected
  at startup.
- **`SIP_RTP_HOST=0.0.0.0`** is what the RTP sockets *bind* to. On most cloud
  VMs the public IP is attached at a NAT or load-balancer layer and is not on
  any interface inside the VM, so binding to it fails outright.

Keep secrets out of the host's process list with an env file instead:

```sh
docker run -d --name ulai-sip --network host \
  --env-file /etc/ulai/sip.env \
  asia-south1-docker.pkg.dev/arctic-operand-415316/ulai/sip:v6
```

```sh
# /etc/ulai/sip.env
ULAI_CONTROL_PLANE_URL=https://stgcp.ulai.co.in
SIP_ROUTING_REDIS_URL=redis://USER:PASSWORD@redis.ulai.co.in:6379
SIP_PUBLIC_IP=148.113.58.51
SIP_LISTEN_ADDR=0.0.0.0:5060
SIP_TRANSPORT=udp
SIP_RTP_HOST=0.0.0.0
SIP_RTP_PORT_LOW=10000
SIP_RTP_PORT_HIGH=10500
SIP_HTTP_PORT=8082
```

Only three variables are genuinely required — `ULAI_CONTROL_PLANE_URL`,
`SIP_ROUTING_REDIS_URL` and `SIP_PUBLIC_IP`. Everything else has a default.
Every knob is in the [configuration reference](/docs/sip/configuration/).

## Check it

A healthy start looks like this:

```sh
docker logs ulai-sip
```

```text
[runtime] GOMAXPROCS=4 (cgroup CPU quota=4.00, host cores=8)
[runtime] GOMEMLIMIT=3481MiB (cgroup limit=4096MiB, 85%)
[runtime] GOGC=200
routing store reachable
sip-sfu-gateway: SIP listening on 0.0.0.0:5060/udp public=148.113.58.51 rtp=10000-10500 rtp_timeout=30s
sip-sfu-gateway: HTTP listening on :8082 (control plane https://stgcp.ulai.co.in, ice_servers=1)
```

If the routing store cannot be reached the gateway still starts, and says so:

```text
WARNING: routing store unreachable (...) — every call will be rejected until it recovers
```

That is deliberate — a listener that answers with an honest `500` is worth more
than one that is not there at all.

Then the two endpoints:

```sh
curl -s localhost:8082/health
# {"status":"ok","service":"sip-sfu-gateway"}

curl -s localhost:8082/calls
# {"live":[]}
```

`/health` says nothing about the SIP leg. Check that separately — the image
ships `sipsak` for exactly this:

```sh
docker exec ulai-sip sipsak -s sip:healthcheck@127.0.0.1:5060
```

## Your first inbound call

1. Point a number at the gateway in the routing store: the SIP domain the
   carrier addresses must map to a project, the number must be assigned, the
   trunk's IP ACL must include the carrier's source address, and a dispatcher
   rule must match.
2. Dial the number.
3. Watch the log. Each line of a call is prefixed with the number, so a busy
   gateway can be read one call at a time:

```text
[+919000000000] [sip-gw] INVITE: to=+919000000000 from=+919111111111 host=sbc.example.com source=13.14.15.16 callID=3e114ad4…
[+919000000000] [sip-gw] routed: project=proj_123 trunk=Main SBC rule=support (direct) agent=support-bot
[+919000000000] [sip-gw] room sess_abc (via resolver, created=true)
[+919000000000] [sip-gw] call accepted — connecting to room sess_abc
[+919000000000] [sfugw] webrtc state: connected
```

A rejection is equally legible, and the SIP status says which check failed —
see [troubleshooting](/docs/sip/operations/troubleshooting/).

## Your first outbound call

`POST /sip/originate` needs a control-plane API key and a trunk id. The room is
created for you unless you pass `session_id`:

```sh
curl -X POST http://148.113.58.51:8082/sip/originate \
  -H "Authorization: Bearer $ULAI_API_KEY" \
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
    "dial_verbatim":    false
  }'
```

Only `to_number` and `trunk_id` are required, plus `project_id` unless
`ULAI_PROJECT_ID` is set. Everything else has a default — including the SRTP
policy, which is inferred from the trunk's transport. Add
`"session_id": "527f78a3262dbe8b0d0a678a9e98ac29"` to dial into a room that
already exists instead of creating one.

The response comes back immediately:

```json
{
  "status": "originating",
  "to_number": "+919363399639",
  "session_id": "527f78a3262dbe8b0d0a678a9e98ac29",
  "created": true,
  "name": "sip-module",
  "trunk": "Main SBC"
}
```

The `202` comes back as soon as the trunk is resolved and the room exists — the
phone is still ringing. Point an agent or a browser at `session_id` while it
does. The room is only *joined* once the callee actually answers, so nothing
greets a ringtone.

## Stopping it

`SIGTERM` starts a drain, not a kill: new calls are refused, in-flight calls get
up to 60 seconds to finish, and anything left is then cancelled with 5 seconds
of grace. Give Docker a matching timeout so it does not `SIGKILL` through it:

```sh
docker stop --time 70 ulai-sip
```

## Where to go next

- [Concepts](/docs/sip/concepts/) — what happens between the INVITE and the audio.
- [HTTP API](/docs/sip/http-api/) — the full request and response shapes.
- [Operations](/docs/sip/operations/) — networking, observability, troubleshooting.
