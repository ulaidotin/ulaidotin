---
title: The shape of a call
description: From join to hangup, and who decides what along the way.
weight: 3
---

## Joining

The room exists first. Whoever created it is the only party that knows which
room the other participants are in, so the agent server never creates one — it
is told which to join.

```
client ──JoinBridge──▶ server ──join──▶ control plane
                          │
                          ├──▶ SFU room     (WebSocket, Opus)
                          └──▶ AI backend   (the placement on the profile)
```

`JoinBridge` returns once the agent has a seat. The backend handshake
continues after that.

## Who speaks first

On a bridge the agent does. The room is already live, nobody rang, and an agent
that joins and stays silent reads as a broken connection.

`opening` overrides this. Set it to caller when the agent is joining a
conversation already in progress and should listen rather than announce itself.

## During

The engine owns the timing. Every 20 ms of caller audio runs the chain —
pre-clean, denoise, near-field foreground isolation, neural VAD — and the VAD
is the authoritative turn driver.

The agent client receives transcripts and turn events and can steer between
turns. What it cannot do is touch the media path: there is no way to send audio
into the call and no way to change the audio processing. Timing stays with the
engine.

## Barge-in

When the caller speaks over the agent, the agent stops. What matters afterwards
is not what the agent *generated* but what the caller actually **heard**, so a
turn-ended event carries both:

> An agent turn generated in full and heard for 300 ms did not happen, whatever
> the transcript says.

A client deciding "have they been told about the fee?" needs the second number,
not the first.

## Ending

Three ways a call ends:

| | |
| --- | --- |
| The caller hangs up | The leg drops; the room reports it. |
| The client ends it | Optionally running the closing sequence first. |
| The model calls `end_call` | Subject to the end-call contract. |

`end_call` is not a request the model gets for free. It must produce evidence —
which of the legitimate endings this is, whether the closing question was
actually spoken and answered — and that evidence is cross-checked against what
the engine independently observed. A rejected `end_call` is answered with a
re-prompt, so the agent keeps talking rather than going silent.

Every rejection is a prompt, and the engine is careful that rejections do not
themselves become a source of repetition.

## Detaching is not hanging up

If the client process dies mid-call, the engine carries on. The caller is not
dropped because the record-keeper crashed; the cost is a reporting gap, which
is the right one to take.
