---
title: Troubleshooting
description: Symptoms, and what actually causes them.
weight: 3
---

## Start here

```sh
docker ps --filter name=agent_server           # is it running?
curl -s http://localhost:8000/health           # → ok
docker logs agent_server --tail 100            # what happened
```

Every line for a call carries its bridge id, so one call is one filter:

```sh
docker logs agent_server 2>&1 | grep 48c7ad0bc30f
```

## The call joins, then dies immediately

Look for `backend connect failed:` right after `joined session=…`.

### `this call carried no credentials for project "…"`

The profile carried a project but no key. The engine holds none of its own, so
there is nothing to fall back onto.

Confirm it in this server's log — `creds=engine default` on the `[AIRoute]`
line means the call carried no key:

```sh
docker logs agent_server 2>&1 | grep AIRoute | tail -5
```

The fix is on the **agent client**, which is the thing that must send a key.
Nothing you change on this host will help: it holds no credentials by design.

### `this call carried no AI project`

Same shape, other field. The profile named neither a project nor a key.

### `insufficient authentication scopes`

Seen on older builds that fell back to ambient credentials. The host's own
identity — a GCE instance service account — authenticated successfully and then
failed the Live handshake, because it lacks the `cloud-platform` scope Vertex
needs.

Current images refuse the call before this can happen, with a message naming
the missing field. If you see this, you are on an older image — check the tag
with `docker inspect --format '{{.Config.Image}}' agent_server`.

### `DetectAnsweringMachine needs a Classifier`

A call asked for answering-machine detection on an image that could not supply
a classifier for it. Current images turn detection off rather than failing the
call. Upgrade the image, or have the client stop asking for AMD.

## The agent joins but never speaks

Check for `greeting primed`. If it is absent, the backend never connected —
see above. If it is present and there is still silence, the caller's side is
the place to look: the room roster line says whether anyone else is actually
there.

## An unknown backend kills the process at boot

```
aibackend: AI_BACKEND="gemini-liv" is not a known backend
```

Deliberate. A whole deployment quietly running a vendor nobody asked for is
worse than refusing to start. The same value in a *request* costs only that
call.

## Calls are refused with `unauthorized`

The caller's key does not match `ULAI_GRPC_API_KEY`. The boot log shows a hash
of what the server expects:

```
[env]   ULAI_GRPC_API_KEY   set len=64 sha256:57a4cbf1
```

If it says `(unset)`, the server is accepting **everything**, and the failure
is elsewhere.

## A tool never runs

Tools are configured on the agent and answered by the agent client — there is
nothing to enable on this host. A tool that never runs is a client-side
problem.

A call that pauses, goes quiet and then apologises is usually a tool the client
answered too slowly. See [tools](/docs/agent-server/concepts/tools/).

## Captions stop after the first utterance

Every caller utterance landing in one caption is a segment that never closed.
Fixed in current images; on an older one the caller's transcript accumulates
into a single bubble while the agent's lines appear separately. Upgrade.

## The caller sounds gated or clipped

Have the client send `audio_nothing` on one call, or set `AUDIO_NOTHING=true`
on this host to test every call. It strips the chain to the denoiser alone, so
if the problem disappears it is the pre-clean stage or the near-field
foreground gate.

It is a **diagnostic**, not a setting. The full chain is the tuned one, and the
stage it removes is what keeps a noisy room out of the model.
