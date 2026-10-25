---
title: Overview
description: What the engine does, what it refuses to do, and where it sits.
weight: 1
---

## What it is

The Ulai Agent Server is a single Go binary (`orchestrator`) that runs one side
of a voice conversation. It joins an Ulai SFU room as an ordinary participant,
connects an AI backend to that room, and manages the conversation until
somebody ends it.

Everything it owns is timing-critical:

- **The audio chain** — pre-clean, denoise, near-field foreground isolation,
  neural VAD.
- **Turn detection** — who has the floor, and when that changes.
- **Barge-in** — cutting the agent mid-sentence when the caller speaks, and
  accounting for how much of the turn was actually heard.
- **The silence ladder** — nudges when a caller goes quiet, and a hangup when
  they stay quiet.
- **The end-call contract** — the evidence a model must produce before it is
  allowed to hang up on a human.

## What it is not

It is **not** a place where anything is stored. There is no database, no agent
table, no recording bucket and no credential file. A call's configuration
arrives in the request that starts it, and everything worth keeping leaves on
the event stream for somebody else to write down.

That is a deliberate constraint rather than an omission. It is what lets one
engine serve agents belonging to different customers, billed to different GCP
projects, running on different AI vendors — and it is what makes the binary
safe to hand to somebody else.

| It owns | Somebody else owns |
| --- | --- |
| Audio, turns, barge-in, playback timing | Agent configuration and where it is stored |
| The AI session and its lifecycle | Which GCP project and key that session uses |
| Tool dispatch and the end-call contract | What a custom tool actually *does* |
| Emitting call events | Writing call records, costs and transcripts |

## Where it sits

```
  agent client  ──gRPC──▶  agent server  ──WebSocket──▶  Ulai SFU room
  (your logic)              (this)                        (the conversation)
       │                       │
       │                       └──▶ AI backend (Gemini Live, …)
       │
       └──▶ your database, your dashboards, your billing
```

The **agent client** decides which agent runs, what it says, and where it is
billed. It sends all of that with each call. The **agent server** runs the
call. The two speak over port 50052 — see [interfaces](/docs/agent-server/grpc-api/).

A client that crashes mid-call does not drop the caller. The engine carries on
with the configuration it was given; what is lost is the client's ability to
steer and to record, not the conversation.

## How it is shipped

As a container image, and only as one. There is no source build, no package to
install and no plugin to load — everything the engine needs is inside it,
including the compiled audio libraries.

What you supply is a port, a key, and a client to drive it. See
[deploy it](/docs/agent-server/getting-started/).

## What it does not carry

Audio never leaves the media path. The agent client receives transcripts and
sends text; no audio frame crosses the control port.

That is what keeps a slow or distant client from being heard: the round trip it
adds lands *between* turns, where tens of milliseconds are invisible — not
inside the frame cadence, where they are a stutter the caller hears.
