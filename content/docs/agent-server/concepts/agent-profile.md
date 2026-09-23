---
title: The agent profile
description: Everything about how a call runs, sent by the client.
weight: 1
---

Almost nothing about a conversation is configured on the agent server. The
prompt, the voice, the GCP project, the service-account key, which behaviours
are on — all of it arrives with each call, in something called the **agent
profile**, sent by the agent client.

This matters to you as an operator for three reasons.

## 1. It explains the short environment list

You are not missing settings. There is no `PROMPT`, no `VOICE`, no
`GEMINI_MODEL` to set here, because those are properties of an *agent*, not of
a *host*. One agent server runs many different agents at once, each with its
own.

## 2. It means one host can serve many customers

Because the GCP project and the service-account key travel with each call, a
single agent server can run:

- agent A, billed to customer A's GCP project, on customer A's key
- agent B, billed to customer B's, on customer B's key

at the same time. As host configuration this would be impossible — one
deployment could only ever serve one account, and serving two meant running two
servers.

## 3. It means this host holds no credentials

There is no service-account key on this machine and no file to mount. If
somebody copies the image, they get no customer's credentials with it.

The trade is that **a call carrying no credentials cannot run**. There is
nothing to fall back onto, so it is refused:

```
gemini: this call carried no credentials for project "my-project" —
the agent server holds none of its own, by design. Set google_credentials_json
on the agent, or GOOGLE_APPLICATION_CREDENTIALS on the agent client that sends
its profile
```

If you see that, the fix is on the **client**, not here. See
[troubleshooting](/docs/agent-server/operations/troubleshooting/).

## What is in a profile

You do not set these — the client does — but knowing what a call carries makes
the logs readable.

| Group | Examples |
| --- | --- |
| **The conversation** | prompt, greeting, closing line, voice, model, language |
| **Placement** | GCP project, region, service-account key, which AI vendor |
| **Behaviour** | can the caller interrupt, is the greeting interruptible, silence handling, answering-machine detection |
| **Tools** | the functions this agent may call, answered by the client |

Each field falls back on its own. A client sending a project but no region gets
**that project** in the default region — not a half-configured call.

## Where the environment still helps

A handful of environment variables act as fallbacks for calls whose profile
leaves a field unset — the default AI backend, whether answering-machine
detection runs. They are listed in
[configuration](/docs/agent-server/configuration/).

The project and the key are **not** among them. Those must come from the call.
