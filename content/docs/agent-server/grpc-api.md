---
title: gRPC API
description: Three services — bridges, sessions, dispatch.
weight: 5
---

The control surface is gRPC, `agentsession.v1`. There is no HTTP API beyond
`/health`.

Every RPC requires the API key as `x-api-key` metadata when
`ULAI_GRPC_API_KEY` is set. Server reflection is registered, so `grpcurl` and
Postman work without a checked-out `.proto` — behind the same key.

The supported client is the Go SDK,
[`ulaisdk`](https://github.com/ulaidotin/ulai-agent-sdk). Nothing in it names
gRPC: options, errors and events are the SDK's own, so the transport can change
without your code changing with it. Other languages generate from the `.proto`.

## AgentBridge — where the agent is

| RPC | Purpose |
| --- | --- |
| `Join` | Put an agent into an existing room. |
| `Disconnect` | Take it out again. |

```go
b, err := client.JoinBridge(ctx, ulaisdk.JoinBridgeRequest{
    AgentID:            "agent_42",     // required
    BridgeID:           roomID,          // required — the server never creates rooms
    ControlPlaneURL:    cpURL,           // required — a property of the ROOM
    ControlPlaneAPIKey: cpKey,
    Profile:            profile,         // the whole agent configuration
})
```

`Join` returns once the agent has a **seat** — not once it is talking.
Resolving the configuration and connecting the AI backend take a second or two
more and continue after it returns. Blocking on them would hold a request open
through a model handshake for failures the caller cannot act on.

`BridgeID` is required because the server never creates a room. It used to,
when the field was empty, and that was a trap: the agent got a room nobody else
had been told about, joined it alone, and was cut for silence seconds later.

Disconnecting an agent does **not** tear the room down. The other participants
are still talking to each other.

## AgentSession — the conversation

`Run` is a bidirectional stream. `Attach` must be its first message; every
other command is refused on an unbound stream.

**You receive:** transcripts (partial and final), turn started/ended with how
much of an agent turn was actually *heard*, agent state, tool calls, token
usage, and how the call ended.

**You send:**

| Command | What it does |
| --- | --- |
| `Say` | Speak this text, as written. For a disclosure or a number read back. |
| `GenerateReply` | Hand the floor back to the model, optionally with one-turn instructions. |
| `InjectContext` | Add a fact to the model's context **without speaking it**. |
| `Interrupt` | Stop the agent mid-utterance. |
| `Hangup` | End the call, optionally running the closing sequence first. |
| `ToolResult` | Answer a tool call. See [tools](/docs/agent-server/concepts/tools/). |
| `HandledTools` | Name the tools this client answers. Sent automatically at `Run`. |

Detaching is **not** hanging up. If the client process dies mid-call the server
carries on with its own logic — dropping a caller because a client crashed is
the worse of the two failures. Ending a call is always explicit.

## AgentDispatch — calls as they start

`Subscribe` streams an `Assignment` for each call the server starts, so a
client can drive calls it did not itself place.

Each assignment carries an **attach deadline**. Miss it and the call is not
dropped — it carries on under the server's own logic. What is lost is
everything the client would have added: its tools, its handlers, its overrides.

## Errors

The SDK translates transport failures into its own vocabulary, and always
keeps the server's message alongside the sentinel:

```go
switch {
case errors.Is(err, ulaisdk.ErrUnauthorized):      // bad or missing key
case errors.Is(err, ulaisdk.ErrUnavailable):       // unreachable — worth a retry
case errors.Is(err, ulaisdk.ErrNotSupported):      // this build does not serve it
case errors.Is(err, ulaisdk.ErrBridgeNotFound):    // the room is gone
case errors.Is(err, ulaisdk.ErrBridgeFull):        // at its participant limit
}
```

A build that forwards calls rather than running them has no sessions of its
own, and answers `AgentSession` and `AgentDispatch` with `ErrNotSupported`
saying exactly that. `AgentBridge` works on both.
