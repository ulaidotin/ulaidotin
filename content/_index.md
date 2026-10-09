---
title: Ulai Docs
description: Run Ulai's voice AI infrastructure on your own VMs.
params:
  body_class: td-navbar-links-all-active
---

{{% blocks/cover height="max td-below-navbar" %}}

{{< home/hero-logo >}}

<!-- prettier-ignore -->
{{% _param description %}}
{.display-6}

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

## What you deploy {.text-center}

Ulai's voice AI stack runs on your own VMs. It is made of independent
components, each shipped as a container image: the **SIP Gateway** connects
your carrier's SIP trunk and puts each phone call into an Ulai SFU voice room,
and the **Agent Server** joins that room with an AI backend and runs the
conversation. These docs cover installing, configuring and operating each one.
{.lead .text-center}

{{< home/call-path >}}

{{% /blocks/section %}}

{{% blocks/section color="ulai-tint" type="row ulai-modules" %}}

## Components {.text-center}

Each one is documented on its own: what it needs, how to run it, how to
configure it, and how to tell when something is wrong.
{.text-center .ulai-section-sub}

{{% home/module title="SIP Gateway" repo="ulai-sip-module" url="/docs/sip/"
  icon="fa-solid fa-phone-volume"
  tags="SIP over UDP · TCP · TLS, Ports 5060 + 10000–10500, Host networking" %}}

Connects your carrier's SIP trunk to Ulai SFU voice rooms. Answers the calls the
carrier sends, places calls through a configured trunk, and bridges the audio of
either into a room where an agent is waiting. Runs on a VM with a public IP.

- [Getting started](/docs/sip/getting-started/)
- [Configuration](/docs/sip/configuration/)
- [HTTP API](/docs/sip/http-api/)
- [Operations](/docs/sip/operations/)

{{% /home/module %}}

{{% home/module title="Agent Server" repo="ulai-agent-server" url="/docs/agent-server/"
  icon="fa-solid fa-wave-square"
  tags="gRPC control on :50052, Gemini Live by default, No credentials on the host" %}}

The voice engine. Joins an Ulai SFU room, connects an AI backend, and runs the
conversation. It holds no database and no credentials: all it needs is a port, a
key, and an agent client to drive it.

- [Deploy it](/docs/agent-server/getting-started/)
- [Configuration](/docs/agent-server/configuration/)
- [Interfaces](/docs/agent-server/grpc-api/)
- [Operations](/docs/agent-server/operations/)

{{% /home/module %}}

More components are documented here as they land.
{.text-center .ulai-section-sub .mt-4 .mb-0}

{{% /blocks/section %}}

{{% blocks/section color="white" type="row ulai-features" %}}

## Built to run on your VMs {.text-center}

{{% blocks/feature icon="fab fa-docker" title="One container each"
  url="docs/" %}}
Every component is a single `docker run`. Nothing to compile, and nothing to
install on the VM beyond Docker.
{{% /blocks/feature %}}

{{% blocks/feature icon="fa-solid fa-rotate" title="Upgrades are restarts"
  url="docs/sip/operations/deployment/" %}}
No state on disk: calls live in memory and routing is read fresh, so replacing a
container is a restart, not a migration.
{{% /blocks/feature %}}

{{% blocks/feature icon="fa-solid fa-user-shield" title="No credentials on the VM"
  url="docs/agent-server/concepts/agent-profile/" %}}
The Agent Server keeps no keys. The GCP project and service-account key travel
with each call, so one host can serve many customers.
{{% /blocks/feature %}}

{{% blocks/feature icon="fa-solid fa-hourglass-half" title="Drains before it stops"
  url="docs/sip/operations/deployment/" %}}
On `SIGTERM` the SIP Gateway refuses new calls and gives live ones up to 60
seconds to finish.
{{% /blocks/feature %}}

{{% blocks/feature icon="fa-solid fa-magnifying-glass" title="One call, one grep"
  url="docs/sip/operations/observability/" %}}
Every log line carries the call's number or bridge id, so following one call
through a busy host is a single filter.
{{% /blocks/feature %}}

{{% blocks/feature icon="fa-solid fa-signs-post" title="Failures you can act on"
  url="docs/sip/operations/troubleshooting/" %}}
A rejected call gets a specific SIP status, and a bad configuration reports
every problem at once.
{{% /blocks/feature %}}

{{% /blocks/section %}}

{{% blocks/section color="primary" type="row ulai-stats" %}}

## By the numbers {.text-center}

{{< home/stat value="3" label="Settings required to start the SIP Gateway"
  url="/docs/sip/configuration/" >}}
{{< home/stat value="~250" label="Concurrent calls per gateway on the default RTP range"
  url="/docs/sip/operations/networking/" >}}
{{< home/stat value="60 s" label="Drain for live calls before the SIP Gateway stops"
  url="/docs/sip/operations/deployment/" >}}
{{< home/stat value="0" label="Credentials stored on an Agent Server host"
  url="/docs/agent-server/concepts/agent-profile/" >}}

{{% /blocks/section %}}

{{% blocks/section color="dark" type="container ulai-cta text-center" %}}

## Put a call through

Stand the gateway up on a VM with a public IP, point a number at it and dial —
then deploy the Agent Server to answer.
{.lead}

<!-- prettier-ignore -->
<div class="td-cta-buttons my-4">
  <a class="btn btn-lg btn-secondary" role="button" href="docs/sip/getting-started/">
    Start with the SIP Gateway
  </a>
  <a class="btn btn-lg btn-outline-light" role="button" href="docs/agent-server/getting-started/">
    Deploy the Agent Server
  </a>
</div>

Ulai — enterprise-ready voice AI · [ulai.in](https://ulai.in)
{.small .mb-0}

{{% /blocks/section %}}
