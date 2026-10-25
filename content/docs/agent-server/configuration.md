---
title: Configuration
description: The short list that is environment, and the long list that is not.
weight: 4
---

Configuration is split by a single rule:

> If it decides **how a call runs**, it travels with the call.
> If it decides **how the process runs**, it is an environment variable.

That split is why the environment list below is so short, and why the same
binary can serve agents belonging to different customers at the same time. The
per-call half is documented under
[the agent profile](/docs/agent-server/concepts/agent-profile/).

The process prints its whole configuration at boot, including which variables
are set but *not read* — usually a rename, or a setting that has moved to the
call:

```
[env] ---- agent server configuration ----
[env]   APP_GRPC_LISTEN_PORT             50052   # gRPC port, all interfaces
[env]   ULAI_GRPC_API_KEY                set len=64 sha256:57a4cbf1   # demanded of callers; unset = NO AUTH
[env]   ONNXRUNTIME_LIB                  /usr/local/lib/libonnxruntime.so (file present)   # Smart Turn and Silero; the image sets it
...
[env] ---- end agent server configuration ----
```

## Process

This is the whole list. Nothing else in the environment is read.

| Variable | Default | What it does |
| --- | --- | --- |
| `APP_GRPC_LISTEN_PORT` | `50052` | The gRPC control surface. Binds all interfaces inside the container. |
| `APP_HTTP_PORT` | `8080` | `/health`, and nothing else. |
| `ULAI_GRPC_API_KEY` | *(none)* | Required as `x-api-key` on every RPC. **Unset means no authentication.** |
| `PROMPT_LOG` | *(on)* | Logs every text turn and tool result sent to the model. `off` stops it. Audio is never logged. |
| `APP_PPROF_PORT` | `6060` | pprof. Binds container-loopback, so publishing the port does nothing. |
| `PROFILER_ENABLED` | `false` | Starts pprof at all. |
| `ONNXRUNTIME_LIB` | *(set by the image)* | Where ONNX Runtime is, for the turn and voice-activity models cascade calls use. Leave it alone. |
| `GOMAXPROCS`, `GOMEMLIMIT`, `GOGC` | *(auto)* | Runtime tuning. `GOMEMLIMIT` is derived from the cgroup limit when unset. |

{{% alert title="ULAI_GRPC_API_KEY" color="warning" %}}
Leave it unset and the server accepts unauthenticated calls — including ones
that disconnect live agents. It logs a warning saying so, once, at startup.

The gRPC listener has **no transport security of its own**. The key, and any
credentials a client sends with a call, cross the wire in clear unless TLS is
terminated in front of the port. Publish it to `127.0.0.1` and put a proxy
there.
{{% /alert %}}

## No longer read

Earlier images also read call settings from the environment. These are now
ignored, because every call setting arrives with the call:

`AI_BACKEND`, `AI_CLASSIFY_BACKEND`, `AI_TRANSCRIBE_MODEL`, `ULAI_LIVE_*`,
`CASCADE_*`, `AMD_ENABLED`, `AUDIO_NOTHING`, `LOST_UTTERANCE_REPLAY`,
`GEMINI_PROJECT_ID`, `GEMINI_LIVE_LOCATION`, `GOOGLE_APPLICATION_CREDENTIALS`.

Remove them from your deployment. While they are set, the boot report lists the
`AI_*`, `GEMINI_*`, `GOOGLE_*` and `ULAI_*` ones on its `NOT READ BY THIS
PROCESS` line.

A call that arrives with no project or no key is refused with a message naming
the field and both places it can be set. It is *not* quietly run against
whatever ambient identity the host happens to carry — on a GCE instance that
succeeds at authentication and then fails the Live handshake with `insufficient
authentication scopes`, which describes neither the missing setting nor the
account it borrowed.
