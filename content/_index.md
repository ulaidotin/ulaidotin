---
title: Ulai SIP Gateway
description: SIP trunks in, Ulai SFU voice rooms out.
params:
  body_class: td-navbar-links-all-active
---

{{% blocks/cover
  title="Ulai SIP Gateway"
  height="full td-below-navbar"
  image_anchor="top"
%}}

<!-- prettier-ignore -->
{{% _param description %}}
{.display-6}

<!-- prettier-ignore -->
<div class="td-cta-buttons my-5">
  <a {{% _param btn-lg primary %}} href="docs/">
    Read the docs
  </a>
  <a {{% _param btn-lg secondary %}}
    href="{{% param github_project_repo %}}"
    target="_blank" rel="noopener noreferrer">
    Get the code
    {{% _param FA brands github "" %}}
  </a>
</div>

{{% blocks/link-down color="info" %}}

{{% /blocks/cover %}}

{{% blocks/lead color="white" %}}

One Go service that puts a phone call into an Ulai SFU room. A carrier's
INVITE is routed, admitted and answered; an API call dials out through a
trunk. Either way the caller ends up as an ordinary WebRTC participant
talking to whoever else is in the room — a browser, or an AI agent.

{{% /blocks/lead %}}

{{% blocks/section color="primary" type="row" %}}

{{% blocks/feature title="Inbound" icon="fa-phone-volume" url="/docs/sip/concepts/call-flows/#inbound" %}}

A carrier INVITE is resolved against the routing store — domain, dialled
number and source IP — then answered, published as `CALL_ANSWERED`, and
bridged into a room created for it.

{{% /blocks/feature %}}

{{% blocks/feature title="Outbound" icon="fa-tower-broadcast" url="/docs/sip/http-api/" %}}

`POST /sip/originate` takes a number and a trunk id. Host, transport and
digest credentials come from the routing store, never from the request.

{{% /blocks/feature %}}

{{% blocks/feature
  title="Source on GitHub" icon="fab fa-github"
  url="https://github.com/ulaidotin/ulai-sip-module"
%}}

Go 1.26, sipgo for signalling, pion for WebRTC and SRTP, one cgo
dependency (libopus) for the room side of the audio path.

{{% /blocks/feature %}}

{{% /blocks/section %}}

{{% blocks/section color="white" type="row" %}}

{{% blocks/feature title="μ-law in, Opus out" icon="fa-wave-square" url="/docs/sip/concepts/media/" %}}

8 kHz G.711 from the carrier is decoded, resampled to 48 kHz and encoded
to Opus for the room — and back again, mixed, on the way down.

{{% /blocks/feature %}}

{{% blocks/feature title="Configuration" icon="fa-sliders" url="/docs/sip/configuration/" %}}

Three required environment variables, a routing store, and a public IP the
carrier can actually reach.

{{% /blocks/feature %}}

{{% blocks/feature title="Running it" icon="fa-server" url="/docs/sip/operations/" %}}

Host networking, an RTP port range, a drain on shutdown, and the two
health checks that tell you both legs are alive.

{{% /blocks/feature %}}

{{% /blocks/section %}}
