---
title: The agent profile
description: Everything about how a call runs, travelling with the call.
weight: 1
---

The profile is the whole configuration of one call: what the agent says, which
voice says it, which AI project pays for it, and which behaviours are on.

Send it and the server needs **no database**. Everything it would have looked
up arrived with the request. That is what lets the engine run as a process with
no storage of its own, and what lets the logic that owns agent configuration be
written in any language against any store.

## Why it is not environment

As environment, these settings pin one deployment to one tenant. An operator
running agents for two customers — each billed to their own GCP account, each
possibly on a different AI vendor — had no way to say so, and no fix short of a
second deployment.

Per request, one engine serves them all. Send the same values every time and
nothing is lost.

## What is on it

**The conversation**

| Field | Notes |
| --- | --- |
| `prompt` | The system instruction. |
| `greeting`, `closing` | The opening line and the line before hangup. |
| `persona_text` | Pushed in as a silent turn *after* the greeting — known, not spoken. |
| `voice`, `model`, `language`, `temperature` | |
| `tools` | Replaces the server's tool set for this call. See [tools](/docs/agent-server/concepts/tools/). |
| `opening` | Who speaks first. |

**Behaviour** — pointers, because absent must mean *"the engine decides"*
rather than *"off"*:

| Field | Nil means |
| --- | --- |
| `allow_interruptions` | on |
| `greeting_interruptible` | off |
| `backchannel_gate` | on — short acknowledgements don't cut the agent off |
| `silence_monitor` | the backend's own default |
| `amd_enabled` | off |
| `audio_nothing` | off |
| `lost_utterance_replay` | the engine's default |

**Placement** — where the call runs and who pays:

| Field | Notes |
| --- | --- |
| `gemini_project_id` | **Required.** The engine has none of its own. |
| `google_credentials_json` | **Required.** A service-account key, verbatim. |
| `gemini_location` | Defaults to `us-central1`. |
| `ai_backend`, `ai_classify_backend`, `ai_transcribe_model` | Falls back to the engine's defaults. |

{{% alert title="The key is a secret on the wire" color="warning" %}}
`google_credentials_json` is a service-account key. The gRPC listener has no
transport security of its own, so send it only with TLS terminated in front of
the port.

The engine never logs it — log lines show `creds=supplied (2347 bytes)`, never
the key — but the transport is yours to protect.
{{% /alert %}}

## Per field, never all-or-nothing

Each field falls back on its own. A profile naming a project but no region asks
for **that project** in the engine's default region — not for the engine's
project. There is no "half-configured" state where naming one thing silently
reverts another.

## A call with no placement is refused

The engine holds no project and no credentials, so there is nothing ambient to
fall back onto:

```
gemini: this call carried no credentials for project "my-project" —
the agent server holds none of its own, by design. Set google_credentials_json
on the agent, or GOOGLE_APPLICATION_CREDENTIALS on the agent client that sends
its profile
```

Refusing is the point. The alternative is borrowing whatever identity the host
carries — on a GCE instance that authenticates successfully and then fails the
Live handshake with `insufficient authentication scopes`, a message that
describes neither the missing setting nor the account it silently used.
