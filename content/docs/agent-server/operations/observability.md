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
read** — usually a rename:

```
[env] ---- agent server configuration ----
[env]   APP_GRPC_LISTEN_PORT   50052   # gRPC port; host is 127.0.0.1
[env]   ULAI_GRPC_API_KEY      set len=64 sha256:57a4cbf1
[env]   GEMINI_PROJECT_ID      (unset)   # NOT READ — the client sends it per call
[env]   NOT READ BY THIS PROCESS: APP_ENV APP_NAME SERVICE_HTTP_PORT
[env]   (set in the environment but nothing here looks at them — usually a rename)
[env] ---- end agent server configuration ----
```

Secrets are shown as a length and a hash, never a value.

Also at boot:

```
aibackend: realtime = gemini-live (default)
aibackend: classify = gemini-live (default)
[grpc] agentsession.v1 listening on [::]:50052 (AgentBridge served; AgentSession/AgentDispatch served)
```

## Per call

| Line | Means |
| --- | --- |
| `[bridge] join: bridge=… agent_id=… participant=…` | The agent has a seat. |
| `[AIRoute] ai_project=… creds=supplied (N bytes)` | The profile carried a key. **The one to check first.** |
| `using the profile supplied with the request` | No database lookup happened. |
| `[gemini] connected, greeting deferred` | The backend handshake succeeded. |
| `greeting primed` | The agent is about to speak. |
| `[ulai] roster: N other participant(s) in room` | Who else is there. |
| `call ended: reason=… duration=…ms played=…ms` | How it ended, and how much was *heard*. |

`[AIRoute]` never prints a credential. It shows `creds=supplied (2347 bytes)`
or `creds=engine default` — the latter meaning the profile carried none, which
is now a refusal.

## Turn accounting

`played=…ms` is not `duration=…ms`. It is how much agent audio the caller
actually heard, and it differs from wall-clock by every second of silence,
ringing and caller speech. It is the number that reflects what was delivered.

The same distinction appears per turn: a turn-ended event carries how much of
that turn was heard before it was cut.

## Audio

With `AUDIO_NOTHING=true` set on this host, the chain is cut down to the
denoiser and says so loudly:

```
[AUDIO] AUDIO_NOTHING=true — pure RNNoise (preprocessor + foreground gate disabled)
```

That line in production is almost always a mistake — see
[configuration](/docs/agent-server/configuration/).

## Prompts

`PROMPT_LOG=on` logs the system instruction at connect. Useful exactly once,
when an agent is behaving oddly and you want to see what it was actually told.
Noisy forever after, and it puts the prompt in your log store.
