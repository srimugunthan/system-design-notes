## WebSockets vs Server-Sent Events (SSE)

Both enable real-time data pushing, but they're built for different communication patterns.

### Core Difference

**WebSockets** = full-duplex, bidirectional channel (both sides talk freely)
**SSE** = unidirectional, server → client only (client uses regular HTTP to send data back)

---

### Protocol & Connection

| | WebSockets | SSE |
|---|---|---|
| Protocol | `ws://` / `wss://` — upgrades from HTTP | Plain HTTP/HTTPS |
| Connection | Persistent TCP socket, custom framing | Long-lived HTTP response (`text/event-stream`) |
| Browser API | `new WebSocket(url)` | `new EventSource(url)` |
| Auto-reconnect | Manual | Built-in |
| HTTP/2 support | Separate spec needed | Native (multiplexed) |

---

### When to Use Which

**Use WebSockets when:**
- Client also sends frequent messages (chat, multiplayer games, collaborative editing)
- You need low-latency bidirectional communication
- Binary data (audio, video frames, file chunks)

**Use SSE when:**
- Server pushes updates, client rarely/never responds in-stream (stock tickers, live logs, LLM token streaming)
- You want simplicity — works over standard HTTP, easier to proxy/cache
- You need auto-reconnect and event IDs out of the box

---

### Quick Code Comparison

**SSE (server, Node.js):**
```js
res.setHeader('Content-Type', 'text/event-stream');
res.write(`data: ${JSON.stringify(payload)}\n\n`);
```

**SSE (client):**
```js
const es = new EventSource('/stream');
es.onmessage = (e) => console.log(e.data);
```

**WebSocket (client):**
```js
const ws = new WebSocket('wss://example.com');
ws.onmessage = (e) => console.log(e.data);
ws.send('hello from client');   // ← SSE can't do this
```

---

### The Practical Punchline

SSE is what Anthropic uses for Claude's streaming API responses — the server streams tokens, the client just listens. WebSockets would be overkill there. But if you were building a collaborative whiteboard where users draw simultaneously, you'd want WebSockets.
