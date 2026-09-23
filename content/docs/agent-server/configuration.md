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
are set but *not read* — usually a rename:

```
[env] ---- agent server configuration ----
[env]   APP_GRPC_LISTEN_PORT             50052   # gRPC port; host is 127.0.0.1
[env]   ULAI_GRPC_API_KEY                set len=64 sha256:57a4cbf1
[env]   GEMINI_PROJECT_ID                (unset)   # NOT READ — the client sends the project per call
...
[env] ---- end agent server configuration ----
```

## Process

| Variable | Default | What it does |
| --- | --- | --- |
| `APP_GRPC_LISTEN_PORT` | `50052` | The gRPC control surface. Binds all interfaces inside the container. |
| `APP_HTTP_PORT` | `8080` | `/health`, and nothing else. |
| `ULAI_GRPC_API_KEY` | *(none)* | Required as `x-api-key` on every RPC. **Unset means no authentication.** |
| `PROMPT_LOG` | *(off)* | Logs the system instruction at connect. Useful once, noisy forever. |
| `APP_PPROF_PORT` | `6060` | pprof. Binds container-loopback, so publishing the port does nothing. |
| `PROFILER_ENABLED` | `false` | Starts pprof at all. |
| `GOMAXPROCS`, `GOMEMLIMIT`, `GOGC` | *(auto)* | Runtime tuning. `GOMEMLIMIT` is derived from the cgroup limit when unset. |

{{% alert title="ULAI_GRPC_API_KEY" color="warning" %}}
Leave it unset and the server accepts unauthenticated calls — including ones
that disconnect live agents. It logs a warning saying so, once, at startup.

The gRPC listener has **no transport security of its own**. The key, and any
credentials a client sends with a call, cross the wire in clear unless TLS is
terminated in front of the port. Publish it to `127.0.0.1` and put a proxy
there.
{{% /alert %}}

## Backend defaults

These name the *default* AI backend for calls whose profile does not choose
one. A profile that names its own overrides them.

| Variable | Default | What it does |
| --- | --- | --- |
| `AI_BACKEND` | `gemini-live` | Carries the conversation. **An unknown value is fatal at boot.** |
| `AI_CLASSIFY_BACKEND` | `gemini-live` | Judges what answered a dialled call. |
| `AI_TRANSCRIBE_MODEL` | `gemini-3.1-flash-lite` | Transcribes the recording afterwards. |
| `ULAI_LIVE_URL`, `ULAI_LIVE_VOICE`, `ULAI_LIVE_INSECURE`, `ULAI_LIVE_OUTPUT_RATE` | *(none)* | Only read when a backend is `ulai-live`. |

An unknown value in the *environment* is fatal, because a whole deployment
quietly running a vendor nobody asked for is worse than refusing to boot. An
unknown value in a *request* costs that one call and is reported — one caller's
typo should not take the fleet down.

## Call defaults

Fallbacks for calls whose profile leaves the corresponding field unset.

| Variable | Default | What it does |
| --- | --- | --- |
| `AMD_ENABLED` | `false` | Answering-machine detection. |
| `AUDIO_NOTHING` | `false` | Strips the caller audio chain to the denoiser alone. Diagnostic. |
| `LOST_UTTERANCE_REPLAY` | on | `off` disables audio replay of a dropped utterance; anything else enables it. |

## Not read at all

These are printed at boot as `NOT READ` so a stale value cannot look
load-bearing:

| Variable | Why |
| --- | --- |
| `GEMINI_PROJECT_ID` | The client sends the project with each call. |
| `GEMINI_LIVE_LOCATION` | The client sends the region; unset defaults to `us-central1`. |
| `GOOGLE_APPLICATION_CREDENTIALS` | **This engine holds no credentials.** The client sends the key with each call. |

A call that arrives with no project or no key is refused with a message naming
the field and both places it can be set. It is *not* quietly run against
whatever ambient identity the host happens to carry — on a GCE instance that
succeeds at authentication and then fails the Live handshake with `insufficient
authentication scopes`, which describes neither the missing setting nor the
account it borrowed.
