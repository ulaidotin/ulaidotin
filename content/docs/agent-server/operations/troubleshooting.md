---
title: Troubleshooting
description: Symptoms, and what actually causes them.
weight: 3
---

## The call joins, then dies immediately

Look for `backend connect failed:` right after `joined session=…`.

### `this call carried no credentials for project "…"`

The profile carried a project but no key. The engine holds none of its own, so
there is nothing to fall back onto.

Check the `[AIRoute]` line: `creds=engine default` confirms it. Then check the
**client** — it is the thing that must send a key. In a deployment where the
client reads one from disk, its own boot log says whether it found one:

```
[pool] AI placement ready: project=…, key 2347 bytes
[pool] WARNING: no service-account key … calls will be refused
```

### `this call carried no AI project`

Same shape, other field. The profile named neither a project nor a key.

### `insufficient authentication scopes`

Seen on older builds that fell back to ambient credentials. The host's own
identity — a GCE instance service account — authenticated successfully and then
failed the Live handshake, because it lacks the `cloud-platform` scope Vertex
needs.

Current builds refuse the call before this can happen, with a message naming
the missing field. If you see this, the build is old.

### `DetectAnsweringMachine needs a Classifier`

A profile asked for answering-machine detection on a build that could not
supply a classifier. Current builds resolve the two together and turn detection
off rather than failing the call. If you see it, the build is old — or set
`amd_enabled` false.

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

Tools need both halves: declared on the profile, and a handler registered
before the session starts pumping. A handler registered after `Run` is one the
server was never told about, and it answers the call itself.

`continue_conversation` cannot be claimed at all — see
[tools](/docs/agent-server/concepts/tools/).

## Captions stop after the first utterance

Every caller utterance landing in one caption is a segment that never closed.
Fixed in current builds; on an older one the caller's transcript accumulates
into a single bubble while the agent's lines appear separately.

## The caller sounds gated or clipped

Try one call with `audio_nothing` on the profile. It strips the chain to the
denoiser alone, so if the problem disappears it is the pre-clean stage or the
near-field foreground gate.

It is a **diagnostic**, not a setting. The full chain is the tuned one, and the
stage it removes is what keeps a noisy room out of the model.
