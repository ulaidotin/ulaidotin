---
title: Concepts
description: How the gateway is put together and how a call moves through it.
weight: 3
---

Three things are worth understanding before changing anything:

- **[Call flows](/docs/sip/concepts/call-flows/)** — the ordered steps of an inbound
  and an outbound call, and why the order differs.
- **[Routing plane](/docs/sip/concepts/routing/)** — how a call is admitted, how a
  trunk is resolved, and what gets published.
- **[Media pipeline](/docs/sip/concepts/media/)** — codecs, SRTP, jitter, DTMF, and
  the ways a call ends when nobody sends a `BYE`.
