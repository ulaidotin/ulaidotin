---
title: Getting started
description: Run it, put an agent in a room, and read the logs that say it worked.
weight: 2
---

## Prerequisites

- A room on an Ulai control plane, and its id. The agent server never creates
  one.
- A GCP project with the Vertex AI API enabled, and a service-account key for
  it. The engine holds no credentials — you send them.
- A key for the gRPC surface (`ULAI_GRPC_API_KEY`), shared with whatever will
  call it.

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

Check it came up, and read back what it thinks its configuration is:

```sh
curl -s localhost:8000/health
docker logs agent_server 2>&1 | grep -A 25 "agent server configuration"
```

The `GEMINI_*` and `GOOGLE_APPLICATION_CREDENTIALS` lines should read
`(unset) # NOT READ`. That is correct — they arrive per call.

## Put an agent in a room

```go
package main

import (
    "context"
    "log"
    "os"

    "github.com/ulaidotin/ulai-agent-sdk/ulaisdk"
)

func main() {
    ctx := context.Background()

    key, err := os.ReadFile("service-account.json")
    if err != nil {
        log.Fatal(err)
    }

    client, err := ulaisdk.Dial(ctx, "127.0.0.1:50052",
        ulaisdk.WithAPIKey(os.Getenv("ULAI_GRPC_API_KEY")))
    if err != nil {
        log.Fatal(err)
    }
    defer client.Close()

    b, err := client.JoinBridge(ctx, ulaisdk.JoinBridgeRequest{
        AgentID:            "agent_42",
        BridgeID:           os.Getenv("ROOM_ID"),
        ControlPlaneURL:    "https://stgcp.ulai.co.in",
        ControlPlaneAPIKey: os.Getenv("CP_KEY"),
        Profile: &ulaisdk.Profile{
            Prompt:   "You are a support agent. Be brief.",
            Greeting: "Hello, thanks for calling. How can I help?",
            Voice:    "Kore",

            // Placement — the engine has none of its own.
            GeminiProjectID:       os.Getenv("GCP_PROJECT"),
            GoogleCredentialsJSON: string(key),
        },
    })
    if err != nil {
        log.Fatal(err)
    }
    log.Printf("agent is in room %s as %s", b.BridgeID, b.ParticipantID)
}
```

{{% alert title="Insecure by default" color="warning" %}}
`Dial` is unencrypted unless you say `ulaisdk.Secure()`. That is fine over
loopback. It is not fine anywhere else — you are sending a service-account key.
{{% /alert %}}

## Read the logs

Server-side, a healthy join looks like this:

```
[bridge] control plane https://stgcp.ulai.co.in (api key: set)
[bridge] join: bridge=48c7ad… agent_id=d1a6b0… participant=agent_7f85…
[AIRoute] ai_project=my-project location=us-central1 creds=supplied (2347 bytes) (from the agent profile)
[bridge:48c7ad…] using the profile supplied with the request — no agent lookup, no database
[bridge:48c7ad…] [ulai] joined session=48c7ad… participant=agent_7f85…
[gemini] connected, greeting deferred (model=gemini-live-2.5-flash-native-audio voice=Kore)
[bridge:48c7ad…] greeting primed
```

Two lines are worth knowing by heart:

- **`[AIRoute] … creds=supplied (N bytes)`** — the profile carried a key. If it
  says `creds=engine default`, your client sent none and the call will be
  refused.
- **`greeting primed`** — the backend is connected and the agent is about to
  speak.

## Watch the conversation

```go
sess, _ := client.Attach(ctx, b.BridgeID, nil, ulaisdk.WithHistory())

sess.OnTranscript(func(t ulaisdk.Transcript) {
    if t.Final {
        log.Printf("%s: %s", t.Speaker, t.Text)
    }
})

result, _ := sess.Run(ctx)
log.Printf("ended: %s after %dms (%dms heard)",
    result.Reason, result.DurationMs, result.PlayedMs)
```

`WithHistory` replays what was said before you attached. Without it you start
blind, and logic that depends on what the caller already asked for gets it
wrong.

## Next

- [The agent profile](/docs/agent-server/concepts/agent-profile/) — every field.
- [Tools](/docs/agent-server/concepts/tools/) — make the agent do something.
- [Troubleshooting](/docs/agent-server/operations/troubleshooting/) — when it
  does not work.
