---
title: Deployment
description: The image, the ports, and the order things must be deployed in.
weight: 1
---

## The image

It is a **heavy image and has to be**: it owns the audio. RNNoise is compiled
from vendored C, TEN VAD is a prebuilt `.so`, and libopus, libsoxr and
libspeexdsp are linked for the transport and the DSP chain.

The runtime stage is Debian slim and carries only the binary plus those shared
libraries. It owns no business data — no database, no agent store, no
credentials.

## Ports

| Port | What | Expose it? |
| --- | --- | --- |
| `50052` | gRPC (`agentsession.v1`) | **Loopback or private network only.** |
| `8000` | `/health` | As needed. |
| `6060` | pprof | Binds container-loopback; publishing it does nothing. |

{{% alert title="There is no TLS" color="warning" %}}
The gRPC server has no transport credentials of its own. The API key and any
service-account key a client sends cross the wire in clear.

Publish `50052` to `127.0.0.1` and terminate TLS in front of it. The SDK's
`Secure()` expects exactly that.
{{% /alert %}}

## Health

`GET /health` returns `ok`. It is a liveness check, not a readiness one — the
process is healthy with zero calls running, which is the normal state.

## Deployment order

The engine holds no project and no credentials, so **the client that sends them
must be deployed first**. Deploying the server ahead of the client means every
call is refused for want of a key.

The order:

1. Deploy the client with its placement configured.
2. Confirm `[AIRoute] … creds=supplied (N bytes)` on a call.
3. Deploy the server.

Reversing it produces a working-looking server and a fleet of refused calls.

## Sizing

One process handles many concurrent calls; the limit is CPU, and the audio
chain is what spends it. Each call runs denoise, foreground isolation and
neural VAD every 20 ms.

`GOMEMLIMIT` is derived from the cgroup limit when unset — the boot log says
what it picked:

```
[runtime] GOMEMLIMIT=13589MiB (/proc/meminfo limit=15987MiB, 85%)
```

## Rolling restarts

`Stop` drains in-flight RPCs and then escalates to a hard stop after a timeout.
One long-lived session stream must not hold a deploy open indefinitely — a
session stream ends when its call does, which can be minutes away.

Calls in flight during a restart end. There is no session migration.
