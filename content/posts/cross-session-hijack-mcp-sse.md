---
title: "Cross-Session Hijack in an MCP-over-SSE Flow"
date: 2026-03-05T00:00:00+00:00
draft: false
summary: "A session routing flaw in an MCP-over-SSE server allowed one authenticated session to receive responses meant for another session."
---

> **Disclosure note**: This write-up is anonymized by design. Product/vendor names, extension identifiers, and exact repository paths are intentionally omitted. The issue was reported through a private bug bounty program, fixed before publication, and rewarded.

## Introduction

This research started with a simple question:

- If two MCP clients connect at the same time, does each session keep its own response stream?

At first, everything looked legitimate. Each client got a different SSE endpoint, requests were accepted, and nothing looked obviously broken.

Then I started correlating response IDs across two inboxes.

That is where things went wrong.

The server processed requests from one session but delivered responses to another session if that session connected later. In practice, whoever connected last could become the receiver for someone else's responses.

This was not a generic localhost concern. It was a session routing flaw between authenticated sessions.

## The Problem

The MCP transport was stored as mutable global state and overwritten on each new SSE connection, making response delivery connection-order dependent instead of session-bound.

## Why This Bug Class Matters

In multi-client protocol servers, authentication alone is not enough. Isolation is the security boundary.

You can have:

- Valid authentication
- Valid session IDs
- Valid request parsing

...and still fail security if response ownership is not bound to the originating session.

If request `id=9002` comes from session A, only session A must receive the response for `id=9002`.

## Architecture Snapshot

The relevant flow looked like this:

1. `GET /<sse-endpoint>` establishes a long-lived server-to-client stream.
2. The server emits a session-specific POST endpoint back to each client.
3. The client sends JSON-RPC over that POST endpoint (`initialize`, `tools/list`, etc).
4. The server sends JSON-RPC responses over SSE.

Expected:

- Request from session A -> response to session A.

Observed under concurrency:

- Request from session A -> response to session B (if B connected later).

## Root Cause

Deobfuscated logic (simplified):

```js
async connect(transport) {
  this._transport = transport;
  await this._transport.start();
}
```

Connection entrypoint (simplified):

```js
app.get('/revenuecat-sse', async (req, res) => {
  const transport = createTransport(req, res);
  await server.connect(transport);
});
```

The issue is straightforward:

1. Session A connects, `_transport = A`.
2. Session B connects, `_transport = B`.
3. Session A sends a request.
4. The response emitter uses `_transport` (now B).

So response routing is effectively driven by a "last connection wins" global pointer.

## Exploit Flow

Minimal attacker model:

- The attacker can open a second client session to the same local MCP service.
- The attacker does not need to forge victim requests.
- The attacker only needs to connect after the victim.

Sequence:

```text
Victim A         Server (global transport)          Attacker B
   |                        |                          |
   |---- SSE connect ------>| _transport = A           |
   |                        |                          |
   |                        |<----- SSE connect -------|
   |                        | _transport = B           |
   |---- POST request(A) -->| handles victim request   |
   |                        |----- response ---------->|
   |   (no response)        |                          |
```

From a protocol perspective, this is a cross-session response hijack.

## Suggested Fix

Global mutable transport should not exist in multi-session mode.

Use explicit per-session routing:

1. Maintain a `sessionId -> transport` mapping.
2. Carry originating session identity through request handling.
3. Send the response only through the transport mapped to that session.
4. Delete the mapping on disconnect.
5. If the design is intentionally single-session, reject concurrent sessions explicitly.

Reference model:

```js
const sessions = new Map();

function onConnect(sessionId, transport) {
  sessions.set(sessionId, transport);
}

async function onRequest(sessionId, request) {
  const response = await handleRequest(request);
  const transport = sessions.get(sessionId);
  if (!transport) throw new Error('session transport missing');
  await transport.send(response);
}

function onDisconnect(sessionId) {
  sessions.delete(sessionId);
}
```
