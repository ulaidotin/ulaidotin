---
title: Agent Server
linkTitle: Agent Server
description: The voice engine, shipped as a container image.
weight: 2
---

`ulai-agent-server` is the voice engine. It joins an Ulai SFU room, connects an
AI backend, and runs the conversation — the audio chain, turn detection,
barge-in, the silence ladder and the end-call contract.

It ships as a **container image**. There is nothing to build and nothing to
compile.

It holds no database, no agent store and no credentials. Everything about *how
a call runs* — the prompt, the voice, the GCP project, the service-account key
— arrives from whatever drives it, per call. That is why the configuration
below is so short.

## Run it

```sh
docker run -d --name agent_server --restart unless-stopped \
  -p 127.0.0.1:50052:50052 \
  -p 8000:8000 \
  -e APP_GRPC_LISTEN_PORT=50052 \
  -e APP_HTTP_PORT=8000 \
  -e ULAI_GRPC_API_KEY='<64-character key>' \
  asia-south1-docker.pkg.dev/arctic-operand-415316/ulai/agent_server:v1.02
```

Check it:

```sh
curl -s http://localhost:8000/health     # → ok
```

That is the whole installation. See
[deploy it](/docs/agent-server/getting-started/) for the full walk-through,
including the part that actually makes calls happen.

## What you still need

The agent server does not place calls by itself. It waits on port `50052` for
an **agent client** to tell it which room to join and which agent to run.

```
  agent client  ──gRPC :50052──▶  agent server  ──▶  Ulai SFU room
  (drives calls)                   (this image)       (the conversation)
```

Both sides must share the same key: whatever you set as `ULAI_GRPC_API_KEY`
here, the client must present. Nothing happens until a client connects.

## In this section

- **[Overview](/docs/agent-server/overview/)** — what it does, what it
  deliberately does not, and where it sits.
- **[Deploy it](/docs/agent-server/getting-started/)** — pull, run, verify, and
  connect a client.
- **[Configuration](/docs/agent-server/configuration/)** — every environment
  variable, and the longer list of things that are *not* environment.
- **[What a call carries](/docs/agent-server/concepts/)** — why the environment
  is short, and what arrives per call instead.
- **[Interfaces](/docs/agent-server/grpc-api/)** — the two ports and what talks
  to them.
- **[Operations](/docs/agent-server/operations/)** — deployment, observability
  and troubleshooting.
