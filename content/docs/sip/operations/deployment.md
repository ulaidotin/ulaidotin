---
title: Deployment
description: Running the image on a host.
weight: 1
---

The gateway is distributed as a container image. A deployment is one container
on a host with a routable public IP, run with host networking.

## docker run

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

- **`--network host`** — see [networking](/docs/sip/operations/networking/).
  Publishing the RTP range with `-p` spawns hundreds of userland proxies and
  rewrites the INVITE's source address, which the trunk ACL then rejects with
  `403`.
- **`SIP_PUBLIC_IP`** — the address the carrier is told to send media to. It must
  be the host's own routable address, and never `0.0.0.0`.
- **`SIP_RTP_HOST=0.0.0.0`** — what the RTP sockets *bind* to. On a cloud VM the
  public IP usually is not on any interface, so binding to it fails outright.

Pass secrets with `--env-file` rather than `-e` if the host's process list is
readable:

```sh
docker run -d --name ulai-sip --network host \
  --env-file /etc/ulai/sip.env \
  asia-south1-docker.pkg.dev/arctic-operand-415316/ulai/sip:v6
```

## docker compose

```yaml
services:
  sip-gateway:
    image: asia-south1-docker.pkg.dev/arctic-operand-415316/ulai/sip:v6
    container_name: ulai-sip
    network_mode: host
    env_file:
      - .env
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "./health-check.sh"]
      interval: 30s
      timeout: 5s
      start_period: 15s
      retries: 3
```

`network_mode: host` replaces a `ports:` block — the container binds the host's
ports directly, which is the only arrangement that keeps the carrier's source
address intact and the RTP range usable.

```sh
docker compose up -d
docker compose logs -f
```

## What the image contains

A Debian slim runtime with the gateway binary, `libopus`, `ca-certificates`,
and `curl` plus `sipsak` for the health check. It runs as an unprivileged user
(`sipgw`, uid 10001) — SIP `5060` and the RTP range are unprivileged ports, so
nothing here needs root.

Exposed ports:

| Port | Protocol | Purpose |
| --- | --- | --- |
| `8082` | TCP | HTTP API |
| `5060` | UDP, TCP | SIP signalling |
| `10000–10500` | UDP | RTP media |

## Upgrading

Pull the new tag, then replace the container. There is no state on disk — every
call is in memory and every routing decision is read fresh — so a replacement is
a restart, not a migration:

```sh
docker pull asia-south1-docker.pkg.dev/arctic-operand-415316/ulai/sip:v7
docker stop --time 70 ulai-sip
docker rm ulai-sip
docker run -d --name ulai-sip --network host --env-file /etc/ulai/sip.env \
  asia-south1-docker.pkg.dev/arctic-operand-415316/ulai/sip:v7
```

Live calls do not survive the replacement beyond the drain window below, so
prefer a quiet period — or run a second host and move the trunk across.

## Restart and shutdown

`SIGINT`/`SIGTERM` starts a drain rather than a kill:

```text
shutdown signal received — draining for up to 1m0s
```

New calls are refused (`503` on both legs), in-flight calls get up to 60
seconds to finish, and anything still up is then cancelled with a further 5
seconds of grace. The routing-store connection is closed last, so a draining
call's `CALL_HANGUP` still has somewhere to go.

Give the container a matching stop timeout so Docker does not `SIGKILL` through
the drain:

```sh
docker stop --time 70 ulai-sip
```
