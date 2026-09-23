---
title: Deploy it
linkTitle: Deploy it
description: Pull, run, verify, and connect a client.
weight: 2
---

## Before you start

You need:

- **Docker**, and access to the registry the image lives in.
- **A key for port 50052** — any 64-character random string. The agent client
  must present the same one.
- **An agent client** to drive it. The agent server never places a call by
  itself; it waits to be told which room to join.

You do **not** need a GCP project, a service-account key, a model name or a
prompt on this host. All of that arrives with each call.

## 1. Pull

```sh
docker login asia-south1-docker.pkg.dev
docker pull asia-south1-docker.pkg.dev/arctic-operand-415316/ulai/agent_server:v1.02
```

Pin the tag. `latest` makes it impossible to say afterwards what was running.

## 2. Run

```sh
docker run -d --name agent_server --restart unless-stopped \
  -p 127.0.0.1:50052:50052 \
  -p 8000:8000 \
  -e APP_GRPC_LISTEN_PORT=50052 \
  -e APP_HTTP_PORT=8000 \
  -e ULAI_GRPC_API_KEY='<64-character key>' \
  asia-south1-docker.pkg.dev/arctic-operand-415316/ulai/agent_server:v1.02
```

{{% alert title="Why 127.0.0.1 on 50052" color="warning" %}}
The gRPC port has **no encryption of its own**. The key — and the credentials
each call carries — cross it in clear.

Bind it to loopback and put a TLS proxy in front, or keep it on a private
network the client shares. Do not publish it to the internet.
{{% /alert %}}

### With docker compose

```yaml
services:
  agent_server:
    image: asia-south1-docker.pkg.dev/arctic-operand-415316/ulai/agent_server:v1.02
    restart: unless-stopped
    ports:
      - "127.0.0.1:50052:50052"
      - "8000:8000"
    environment:
      APP_GRPC_LISTEN_PORT: 50052
      APP_HTTP_PORT: 8000
      ULAI_GRPC_API_KEY: "${ULAI_GRPC_API_KEY}"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s
```

## 3. Verify it started

```sh
curl -s http://localhost:8000/health
```

`ok` means the process is alive. It says nothing about calls — the server is
healthy with zero calls running, which is its normal state.

Now read back what it thinks its configuration is:

```sh
docker logs agent_server 2>&1 | grep -A 25 "agent server configuration"
```

```
[env] ---- agent server configuration ----
[env]   APP_GRPC_LISTEN_PORT   50052   # gRPC port; host is 127.0.0.1
[env]   APP_HTTP_PORT          8000    # health only
[env]   ULAI_GRPC_API_KEY      set len=64 sha256:57a4cbf1
[env]   GEMINI_PROJECT_ID      (unset) # NOT READ — the client sends it per call
[env] ---- end agent server configuration ----
[grpc] agentsession.v1 listening on [::]:50052
```

Three things to confirm:

| Check | Why |
| --- | --- |
| `ULAI_GRPC_API_KEY  set len=64` | If it says `(unset)`, **anyone who reaches the port can control calls.** |
| The `GEMINI_*` lines say `NOT READ` | Correct. Those arrive per call. |
| `[grpc] … listening` | The control port is up. |

The log also prints any variable that is set but nothing reads — usually a
rename, and worth a look.

## 4. Point a client at it

Configure the agent client with this host's address and the same key. In a
typical client that is:

| Client setting | Value |
| --- | --- |
| Orchestrator address | `host:50052` |
| Orchestrator API key | the same `ULAI_GRPC_API_KEY` |
| TLS | on, if you put a proxy in front |

Nothing appears in the agent server's log until the client connects and starts
a call. Silence here is normal.

## 5. Confirm a real call

Place one call through the client, then:

```sh
docker logs agent_server --tail 50
```

A healthy call looks like this:

```
[bridge] join: bridge=48c7ad… agent_id=d1a6b0… participant=agent_7f85…
[AIRoute] ai_project=my-project location=us-central1 creds=supplied (2347 bytes)
[bridge:48c7ad…] using the profile supplied with the request — no agent lookup, no database
[gemini] connected, greeting deferred (model=… voice=Kore)
[bridge:48c7ad…] greeting primed
```

Two lines are the ones to know:

- **`creds=supplied (N bytes)`** — the client sent credentials. If it says
  `creds=engine default`, the client sent none and the call will be refused.
- **`greeting primed`** — the backend connected and the agent is about to
  speak. If this never appears, the call died before it could talk.

## Upgrading

```sh
docker pull …/agent_server:v1.03
docker stop agent_server && docker rm agent_server
docker run -d … …/agent_server:v1.03      # same flags
```

Calls in flight during a restart end — there is no session migration. Restart
when the log is quiet, or accept the dropped calls.

## Next

- [Configuration](/docs/agent-server/configuration/) — every variable.
- [Troubleshooting](/docs/agent-server/operations/troubleshooting/) — when a
  call does not work.
