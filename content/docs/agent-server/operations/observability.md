---
title: Observability
description: The log lines worth knowing, and what they tell you.
weight: 2
---

Everything goes to stdout, so `docker logs` is the whole interface. Every line
for a call carries its bridge id, which makes one call one filter:

```sh
docker logs agent_server --tail 100
docker logs agent_server 2>&1 | grep 48c7ad0bc30f     # one call
docker logs -f agent_server                            # follow
```

There are no metrics endpoints. Ship stdout to wherever you keep logs.

## At boot

The whole configuration is printed, including variables that are **set but not
read** — usually a rename, or a setting that now travels with the call:

```
[env] ---- agent server configuration ----
[env]   APP_GRPC_LISTEN_PORT             50052   # gRPC port, all interfaces
[env]   ULAI_GRPC_API_KEY                set len=64 sha256:57a4cbf1   # demanded of callers; unset = NO AUTH
[env]   ONNXRUNTIME_LIB                  /usr/local/lib/libonnxruntime.so (file present)   # Smart Turn and Silero; the image sets it
[env]   NOT READ BY THIS PROCESS: AI_BACKEND APP_ENV APP_NAME SERVICE_HTTP_PORT
[env]   (set in the environment but nothing here looks at them — usually a rename)
[env] ---- end agent server configuration ----
```

Secrets are shown as a length and a hash, never a value. A path that names a
missing file says `!! NO SUCH FILE`.

Also at boot:

```
[grpc] agentsession.v1 listening on [::]:50052 (AgentBridge served; AgentSession/AgentDispatch served)
```

## Per call

| Line | Means |
| --- | --- |
| `[bridge] join: bridge=… agent_id=… participant=…` | The agent has a seat. |
| `[AIRoute] ai_project=… creds=supplied (N bytes) (from the agent profile)` | The profile carried a key. **The one to check first.** |
| `using the profile supplied with the request` | No database lookup happened. |
| `[gemini] connected, greeting deferred` | The backend handshake succeeded. |
| `ready — mode=… llm=…@… tts=…@… …` | A cascade call is up, with every setting it is running on. |
| `greeting primed` | The agent is about to speak. |
| `roster: N other participant(s) in room` | Who else is there. |

`[AIRoute]` never prints a credential. It shows `creds=supplied (2347 bytes)`
or `creds=engine default` — the latter meaning the profile carried none, which
is a refusal.

## Audio

When a call's profile sets `audio_nothing`, the chain is cut down to the
denoiser and the call's log says so loudly:

```
[AUDIO] audio_nothing — pure RNNoise (preprocessor + foreground gate disabled)
```

That line in production is almost always a mistake — see
[troubleshooting](/docs/agent-server/operations/troubleshooting/).

## Prompts

`PROMPT_LOG` is on unless set to `off`. Every text turn and tool result sent to
the model is logged as an `[→AI]` line — the persona, the nudges, the closing
instruction — which is the only record of what the model was actually told.

Turn it off if prompts or caller details should not reach your log store.
Audio is never logged either way.
