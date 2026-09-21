---
title: Getting started
description: Build it, configure it, and put a call through it.
weight: 2
---

## Quickest path: the published image

If you only want a running gateway, skip the build entirely:

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

`SIP_PUBLIC_IP` must be the host's own routable address — it is what the carrier
is told to stream media to. `--network host` is required; publishing the RTP
range with `-p` spawns hundreds of userland proxies **and** makes every INVITE
arrive from the bridge gateway's address, which the trunk ACL then rejects with
`403`. See [networking](/docs/sip/operations/networking/).

Then jump to [check it](#check-it). The rest of this page builds from source.

## Prerequisites

| Requirement | Why |
| --- | --- |
| Go 1.26+ with cgo enabled | `libopus` is linked natively |
| `libopus` development headers | Opus encode/decode for the room side |
| A reachable public IP | Advertised to the carrier in SDP and `Contact` |
| An Ulai control plane URL | Rooms are created and joined there |
| A routing store (Redis) | Every call is resolved against it |
| A SIP trunk | Inbound numbers, outbound termination, or both |

On Debian/Ubuntu:

```sh
sudo apt-get install -y gcc pkg-config libopus-dev libopusfile-dev
```

On macOS:

```sh
brew install opus pkg-config
```

The gateway depends on two private modules — `ulai-go-sdk` and
`ulai-sip-resolver` — so `go mod download` needs SSH access to
`github.com/ulaidotin`:

```sh
export GOPRIVATE=github.com/ulaidotin/*
git config --global url."ssh://git@github.com/".insteadOf "https://github.com/"
```

## Build

```sh
git clone git@github.com:ulaidotin/ulai-sip-module.git
cd ulai-sip-module
CGO_ENABLED=1 go build -o sip-gateway .
```

Or build the container, which bakes in libopus and the health check:

```sh
DOCKER_BUILDKIT=1 docker build --ssh default -f Dockerfile.sip -t ulai-sip .
```

`--ssh default` is required: the build fetches the two private modules.

## Configure

Copy `.env.example` and fill in the three required values:

```sh
ULAI_CONTROL_PLANE_URL=https://stgcp.ulai.co.in
SIP_ROUTING_REDIS_URL=redis://user:password@redis.example.com:6379/0
SIP_PUBLIC_IP=203.0.113.10

SIP_LISTEN_ADDR=0.0.0.0:5060
SIP_TRANSPORT=udp
SIP_RTP_PORT_LOW=10000
SIP_RTP_PORT_HIGH=10500
SIP_HTTP_PORT=8082
```

Every knob, with its default, is in the [configuration
reference](/docs/sip/configuration/).

{{% alert title="SIP_PUBLIC_IP is not a listen address" color="warning" %}}
It is what every SDP the gateway sends tells the carrier to stream media to.
`0.0.0.0` is rejected at startup for exactly that reason. Use `SIP_RTP_HOST` if
you need to control what the RTP sockets *bind* to — on most cloud VMs the
public IP is not attached to any interface, so the default `0.0.0.0` is correct.
{{% /alert %}}

## Run

```sh
./sip-gateway
```

A healthy start looks like this:

```text
[runtime] GOMAXPROCS=4 (cgroup CPU quota=4.00, host cores=8)
[runtime] GOMEMLIMIT=3481MiB (cgroup limit=4096MiB, 85%)
[runtime] GOGC=200
routing store reachable
sip-sfu-gateway: SIP listening on 0.0.0.0:5060/udp public=203.0.113.10 rtp=10000-10500 rtp_timeout=30s
sip-sfu-gateway: HTTP listening on :8082 (control plane https://stgcp.ulai.co.in, ice_servers=1)
```

If the routing store cannot be reached the gateway still starts, and says so:

```text
WARNING: routing store unreachable (...) — every call will be rejected until it recovers
```

That is deliberate — a listener that answers with an honest `500` is worth more
than one that is not there at all.

## Check it

```sh
curl -s localhost:8082/health
# {"status":"ok","service":"sip-sfu-gateway"}

curl -s localhost:8082/calls
# {"live":[]}
```

And the SIP leg, which `/health` says nothing about:

```sh
sipsak -s sip:healthcheck@127.0.0.1:5060
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

## Where to go next

- [Concepts](/docs/sip/concepts/) — what happens between the INVITE and the audio.
- [HTTP API](/docs/sip/http-api/) — the full request and response shapes.
- [Operations](/docs/sip/operations/) — how to deploy it without breaking media.
