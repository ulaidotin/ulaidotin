---
title: Documentation
linkTitle: Docs
menu: { main: { weight: 20 } }
weight: 20
---

Ulai is built from independent modules. Each one is documented on its own, with
its own configuration, API and operational notes.

## Modules

{{% alert title="SIP Gateway" color="primary" %}}
**[ulai-sip-module](/docs/sip/)** — connects SIP telephony to Ulai SFU voice
rooms. Answers calls a carrier sends to it, places calls through a configured
trunk, and bridges the audio of either into a room where an agent or a browser
is waiting.

[Overview](/docs/sip/overview/) ·
[Getting started](/docs/sip/getting-started/) ·
[Configuration](/docs/sip/configuration/) ·
[HTTP API](/docs/sip/http-api/)
{{% /alert %}}

{{% alert title="Agent Server" color="primary" %}}
**[ulai-agent-server](/docs/agent-server/)** — the voice engine, shipped as a
container image. Joins an Ulai SFU room, connects an AI backend, and runs the
conversation: the audio chain, turn detection, barge-in and the end-call
contract. It holds no database and no credentials — everything about how a call
runs arrives with the call.

[Overview](/docs/agent-server/overview/) ·
[Deploy it](/docs/agent-server/getting-started/) ·
[Configuration](/docs/agent-server/configuration/) ·
[Interfaces](/docs/agent-server/grpc-api/)
{{% /alert %}}

More modules are documented here as they land.
