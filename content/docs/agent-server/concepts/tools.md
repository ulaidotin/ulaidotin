---
title: Tools
description: Your functions, called by the model, answered by your process.
weight: 2
---

An agent that can only talk is half an agent. Tools are how it looks something
up in *your* system mid-call — a balance, an order status, an appointment slot.

The engine has no database and no idea what your tools mean, so it does not
answer them. You do.

## Declaring and answering are one act

A tool needs both halves to work: a **declaration**, or the model never calls
it, and a **handler**, or nothing answers when it does. Register them together:

```go
// Declared on the profile, so the model knows it exists…
profile.Tools = []ulaisdk.Tool{{
    Name:        "check_balance",
    Description: "Look up the caller's current balance",
    Params:      map[string]ulaisdk.Param{"account": {Type: "string"}},
    Required:    []string{"account"},
}}

// …and handled on the session, so something answers when it is called.
sess.Tool("check_balance", func(ctx context.Context, args map[string]any) (any, error) {
    return theirDatabase.Balance(ctx, args["account"].(string))
})
```

Registering them separately means either half can be forgotten, and both
failures are silent: a declared tool with no handler is a model told its own
tool does not exist; a handler with no declaration is code that never runs.

## Claiming

Registering a handler is what **claims** the tool. When the session starts
pumping, the SDK names every handler it holds to the server, and from then on
the server routes those calls to you instead of answering them itself.

That includes the server's own `end_call`: register a handler for it and the
ending contract becomes yours — along with the duty to answer it and to call
`Hangup` when the call really should end.

A tool you did not claim is answered by the server.

## The deadline

The server holds the turn open while you answer, and the model is **silent for
every millisecond of it**. The caller hears that silence.

So the wait is bounded, and it always ends in an answer to the model — never in
silence. If your handler does not answer in time, the model is told the lookup
failed, so it can apologise and carry on. A tool that failed and was apologised
for beats a line that went dead.

A handler doing real I/O should carry its own timeout well inside the server's.

## Errors are answers

Returning an error from a handler does **not** abandon the call. The error text
is reported to the model as the tool's result, because a model waiting on a
result that never arrives stalls the turn and the caller hears nothing at all.

## The one tool you cannot claim

`continue_conversation` — the no-op decoy declared beside `end_call` — is
answered inside the receive loop and never reaches dispatch.

That is deliberate. Realtime function calling is synchronous: the model stops
generating until the answer comes back. The decoy exists to give a model that
feels the urge to "use a tool" somewhere harmless to put it, and routing it
over the network would spend a multi-second silence avoiding a misfire that
costs nothing.

Claiming it is ignored rather than refused.

## Observing without answering

`OnToolCall` fires for **every** tool call the model makes, including the ones
the server keeps for itself. It observes; it does not answer. Registering a
handler is what answers one.
