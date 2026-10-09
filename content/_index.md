---
title: Ulai Docs
description: >-
  Deploy, configure and operate the Ulai voice AI platform on infrastructure you
  control.
params:
  body_class: td-navbar-links-all-active
---

{{% blocks/cover height="max td-below-navbar" %}}

{{< home/hero-logo >}}

Enterprise voice AI on infrastructure you control
{.display-6}

Deploy, configure and operate every component of the Ulai platform, from the
SIP trunk to the AI agent.
{.ulai-hero-sub}

<!-- prettier-ignore -->
<div class="td-cta-buttons my-5">
  <a class="btn btn-lg btn-secondary" role="button" href="docs/">
    <i class="fa-solid fa-rocket me-2" aria-hidden="true"></i>Get started
  </a>
  <a class="btn btn-lg btn-outline-light" role="button" href="docs/sip/">
    <i class="fa-solid fa-phone-volume me-2" aria-hidden="true"></i>SIP Gateway
  </a>
  <a class="btn btn-lg btn-outline-light" role="button" href="docs/agent-server/">
    <i class="fa-solid fa-wave-square me-2" aria-hidden="true"></i>Agent Server
  </a>
</div>

{{% /blocks/cover %}}

{{% blocks/section color="white" type="container ulai-intro" %}}

## Platform overview {.text-center}

Ulai is a modular voice AI platform that you deploy in your own environment.
Each component ships as a self-contained container image with a single
responsibility: the **SIP Gateway** bridges your carrier's telephony into Ulai's
real-time voice rooms, and the **Agent Server** brings an AI agent into the
conversation. This documentation covers every component end to end:
requirements, deployment, configuration and day-two operations.
{.lead .text-center}

{{< home/call-path >}}

{{% /blocks/section %}}

{{% blocks/section color="ulai-tint" type="row ulai-modules" %}}

## Components {.text-center}

Each component is documented independently, with its own requirements,
configuration reference, interfaces and operational guidance.
{.text-center .ulai-section-sub}

{{% home/module title="SIP Gateway" repo="ulai-sip-module" url="/docs/sip/"
  icon="fa-solid fa-phone-volume"
  tags="SIP over UDP / TCP / TLS, G.711 ↔ Opus transcoding, SRTP media" %}}

Bridges SIP telephony into Ulai voice rooms. Answers inbound calls from your
carrier, originates outbound calls through configured trunks, and delivers each
caller into a room as a standard WebRTC participant.

- [Getting started](/docs/sip/getting-started/)
- [Configuration](/docs/sip/configuration/)
- [HTTP API](/docs/sip/http-api/)
- [Operations](/docs/sip/operations/)

{{% /home/module %}}

{{% home/module title="Agent Server" repo="ulai-agent-server" url="/docs/agent-server/"
  icon="fa-solid fa-wave-square"
  tags="gRPC control API, Gemini Live by default, Stateless host" %}}

The real-time conversation engine. Joins a voice room, connects the AI backend
and manages the dialogue: audio processing, turn-taking, interruptions and call
completion. Agent configuration and credentials are supplied with each call and
never stored on the host.

- [Deployment](/docs/agent-server/getting-started/)
- [Configuration](/docs/agent-server/configuration/)
- [Interfaces](/docs/agent-server/grpc-api/)
- [Operations](/docs/agent-server/operations/)

{{% /home/module %}}

Additional components will be documented as they become available.
{.text-center .ulai-section-sub .mt-4 .mb-0}

{{% /blocks/section %}}

{{% blocks/section color="white" type="row ulai-features" %}}

## Built for production {.text-center}

{{% blocks/feature icon="fab fa-docker" title="Container-native"
  url="docs/" %}}
Every component ships as a single, self-contained container image. No build
toolchain, and no host dependencies beyond Docker.
{{% /blocks/feature %}}

{{% blocks/feature icon="fa-solid fa-rotate" title="Stateless upgrades"
  url="docs/sip/operations/deployment/" %}}
No local state to migrate. Upgrading a component is a matter of replacing its
container.
{{% /blocks/feature %}}

{{% blocks/feature icon="fa-solid fa-user-shield" title="Credential-free hosts"
  url="docs/agent-server/concepts/agent-profile/" %}}
The Agent Server stores no keys. Cloud project and credentials arrive with each
call, so a single deployment can serve multiple tenants.
{{% /blocks/feature %}}

{{% blocks/feature icon="fa-solid fa-hourglass-half" title="Graceful shutdown"
  url="docs/sip/operations/deployment/" %}}
On `SIGTERM` the SIP Gateway stops accepting new calls and gives calls in
progress up to 60 seconds to complete.
{{% /blocks/feature %}}

{{% blocks/feature icon="fa-solid fa-magnifying-glass" title="Per-call traceability"
  url="docs/sip/operations/observability/" %}}
Every log line is tagged with the call's number or bridge ID, so any single
call can be traced end to end with one filter.
{{% /blocks/feature %}}

{{% blocks/feature icon="fa-solid fa-signs-post" title="Actionable diagnostics"
  url="docs/sip/operations/troubleshooting/" %}}
Rejected calls return a precise SIP status, and configuration errors are
reported together at startup.
{{% /blocks/feature %}}

{{% /blocks/section %}}

{{% blocks/section color="primary" type="row ulai-stats" %}}

## At a glance {.text-center}

{{< home/stat value="3" label="Required settings to start the SIP Gateway"
  url="/docs/sip/configuration/" >}}
{{< home/stat value="~250" label="Concurrent calls per gateway with the default RTP range"
  url="/docs/sip/operations/networking/" >}}
{{< home/stat value="60 s" label="Graceful drain window when the SIP Gateway shuts down"
  url="/docs/sip/operations/deployment/" >}}
{{< home/stat value="0" label="Credentials stored on Agent Server hosts"
  url="/docs/agent-server/concepts/agent-profile/" >}}

{{% /blocks/section %}}

{{% blocks/section color="dark" type="container ulai-cta text-center" %}}

## Ready to deploy?

Start with the SIP Gateway to bring calls in, then deploy the Agent Server to
handle the conversation.
{.lead}

<!-- prettier-ignore -->
<div class="td-cta-buttons my-4">
  <a class="btn btn-lg btn-secondary" role="button" href="docs/sip/getting-started/">
    Deploy the SIP Gateway
  </a>
  <a class="btn btn-lg btn-outline-light" role="button" href="docs/agent-server/getting-started/">
    Deploy the Agent Server
  </a>
</div>

Ulai — enterprise-ready voice AI · [ulai.in](https://ulai.in)
{.small .mb-0}

{{% /blocks/section %}}
