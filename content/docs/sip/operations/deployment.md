---
title: Deployment
description: Container, compose, and the CI pipeline.
weight: 1
---

## Run the published image

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

## Build the image

```sh
DOCKER_BUILDKIT=1 docker build --ssh default -f Dockerfile.sip -t ulai-sip .
```

`--ssh default` is required: `ulai-go-sdk` and `ulai-sip-resolver` are private
modules fetched under `GOPRIVATE` during `go mod download`. Without a forwarded
key the build fails with *Permission denied (publickey)* even when the repo
clone succeeded.

The image is a two-stage build — `golang:1.26-bookworm` with `libopus-dev`, then
`debian:bookworm-slim` with `libopus0`, `ca-certificates`, `curl` and `sipsak`.
It runs as an unprivileged user (`sipgw`, uid 10001): SIP `5060` and the RTP
range are unprivileged ports, so nothing here needs root.

`CGO_ENABLED=1` stays because of libopus, and only libopus — the resampler,
G.711 codecs, SIP, SDP and RTP are all pure Go, which is also why the image is
no longer pinned to `amd64`.

## docker compose

`docker-compose.yml` reads a `.env` file and publishes the ports explicitly.
It is convenient for a single-tenant VM where the environment is already
written out by CI — but read
[networking](/docs/sip/operations/networking/#port-publishing-versus-host-networking)
before using the published-port form in production.

```sh
docker compose build --no-cache
docker compose up -d
docker compose logs -f
```

Ports come from the `SERVICE_SIP_*` variables:

```sh
APP_NAME=ulai-sip-gateway
APP_ENV=dev
SERVICE_SIP_PORT=5060
SERVICE_SIP_RTP_PORT_LOW=10000
SERVICE_SIP_RTP_PORT_HIGH=10500
SERVICE_SIP_HTTP_PORT=8082
```

## CI

`.github/workflows/deploy.yml` deploys on every push to `prod` or `staging`, and
on manual dispatch:

1. The branch picks the environment — `prod` → production, `staging` → staging —
   and the deploy directory under `~/apps/`.
2. It SSHes to the VM, pulls the branch, and writes `.env` from repository
   secrets.
3. `docker compose build --no-cache`, `down -v`, `up -d`, then `docker system
   prune -a -f`.
4. It verifies **both** legs — `curl /health` and a `sipsak` OPTIONS ping — and
   dumps the last 50 log lines and fails the job if either is down.

Two SSH keys are involved: one to clone the repo, and one with read access to
the private Go modules pulled during the image build. Forwarding only the first
gets you a successful clone followed by a failed build.

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
