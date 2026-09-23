---
title: Tools
description: Why an agent can look things up, and where that work happens.
weight: 2
---

An agent can do more than talk. Mid-call it can look up a balance, an order
status, an appointment slot — anything the business it belongs to can answer.

**None of that work happens on this host.** The agent server has no database
and no idea what any of those tools mean. It carries the request out to the
agent client and carries the answer back.

```
model asks for "check_balance"
        │
        ▼
  agent server ──▶ agent client ──▶ the business's own system
        ◀──────────── answer ◀───────────────┘
        │
        ▼
model continues the conversation
```

## What this means operationally

**Tools are configured on the agent, not here.** There is nothing to install,
declare or permit on this host to make a tool available.

**A slow tool is heard as silence.** While the client is answering, the model
is not speaking, and the caller hears that. The wait is bounded and always ends
in an answer to the model — if the client does not answer in time, the model is
told the lookup failed so it can apologise and carry on. A caller hearing "I
couldn't pull that up just now" is a far better outcome than a line that goes
quiet.

So a tool that is slow on the client side shows up as pauses in conversations,
not as errors in this log.

**Two tools belong to the server.** `end_call` — how a model hangs up, subject
to a contract that checks it is entitled to — and a no-op tool declared beside
it. The second exists so that a model which feels the urge to "use a tool" at a
moment that is not an ending has somewhere harmless to put it, rather than
reaching for `end_call` and cutting a live human off.

It is answered instantly, inside the server, precisely because sending it out
and back would spend a multi-second silence avoiding a mistake that costs
nothing.

## In the logs

Tool activity appears against the call's bridge id. A call that pauses oddly,
with the agent going quiet and then apologising, is usually a tool the client
was slow to answer — the place to look is the client's logs, not this one.
