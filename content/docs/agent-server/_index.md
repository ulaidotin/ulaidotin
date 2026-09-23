---
title: Agent Server
linkTitle: Agent Server
description: The voice engine. Agents in, conversations out.
weight: 2
---

`ulai-agent-server` is a single Go binary (`orchestrator`) that runs a voice
conversation. It joins an Ulai SFU room as a participant, connects an AI
backend, and owns everything that has to be right to the millisecond — the
audio chain, turn detection, barge-in, playback accounting, the silence ladder,
the end-call contract.

It holds no database, no agent store and no credentials. Everything about *how
a call runs* arrives with the request that starts it.

Source: [github.com/ulaidotin/ulai-agent-server](https://github.com/ulaidotin/ulai-agent-server)

## Run it

```sh
docker run -d --name agent_server --restart unless-stopped \
  -p 127.0.0.1:50052:50052 \
  -p 8000:8000 \
  -e APP_GRPC_LISTEN_PORT=50052 \
  -e APP_HTTP_PORT=8000 \
  -e ULAI_GRPC_API_KEY='<64-char key>' \
  -e PROMPT_LOG=on \
  asia-south1-docker.pkg.dev/arctic-operand-415316/ulai/agent_server:v1.02
```

That is the whole configuration. No GCP project, no service-account key, no
model, no vendor — those travel with each call. See
[configuration](/docs/agent-server/configuration/).

## Put an agent in a room

Nothing here is HTTP. The control surface is gRPC, and the supported way to
speak it is the Go SDK:

```go
client, _ := ulaisdk.Dial(ctx, "agent-server:50052",
    ulaisdk.WithAPIKey(key), ulaisdk.Secure())

b, _ := client.JoinBridge(ctx, ulaisdk.JoinBridgeRequest{
    AgentID:            "agent_42",
    BridgeID:           roomID,
    ControlPlaneURL:    "https://stgcp.ulai.co.in",
    ControlPlaneAPIKey: "ulai_live_…",
    Profile: &ulaisdk.Profile{
        Prompt:                "You are a support agent for …",
        Greeting:              "Hello, thanks for calling.",
        Voice:                 "Kore",
        GeminiProjectID:       "my-gcp-project",
        GoogleCredentialsJSON: string(keyJSON),
    },
})
```

The room must already exist — the agent server never creates one.

## In this section

- **[Overview](/docs/agent-server/overview/)** — what it does, what it
  deliberately does not, and where it sits.
- **[Getting started](/docs/agent-server/getting-started/)** — run it, put an
  agent in a room, and read the logs that say it worked.
- **[Concepts](/docs/agent-server/concepts/)** — the agent profile, tools, and
  the shape of a call.
- **[Configuration](/docs/agent-server/configuration/)** — every environment
  variable, and the much longer list of things that are *not* environment.
- **[gRPC API](/docs/agent-server/grpc-api/)** — the three services and what
  each one is for.
- **[Operations](/docs/agent-server/operations/)** — deployment, observability
  and troubleshooting.
