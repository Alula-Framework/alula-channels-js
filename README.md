# @alula-framework/channels

The JS/TS reference client for Alula Channels: WebSocket management, the
envelope protocol, ref/reply correlation (`push()` returns a promise resolving
on the matching `alula:reply`), the heartbeat, and automatic
reconnect-with-backoff-and-rejoin. Deliberately small and dependency-free —
protocol plumbing, not a framework.

ESM, zero dependencies, typed via `types/index.d.ts`. Works in browsers and
in Node ≥ 18 (inject a WebSocket implementation where there's no global).

> phoenix.js does not work against Alula, by design: Alula owns
> both server and clients, and this is the client it versions with the
> protocol.

## Usage

```js
import { AlulaSocket, ChannelError, TimeoutError } from "@alula-framework/channels";

const socket = new AlulaSocket("wss://example.app/socket?token=…");
await socket.connect();

const room = socket.channel("room:42");
const initialState = await room.join();          // the join is the gate

room.on("new_msg", (payload) => render(payload)); // server pushes
room.on("*", (payload, event) => log(event));     // everything, incl. rejoins

const reply = await room.push("new_msg", { body: "hi" }); // awaits alula:reply
room.send("typing", { on: true });                        // fire-and-forget (ref: null)

await room.leave();
socket.disconnect();
```

### Errors

`push()`/`join()` reject with:

- `ChannelError` — the server answered `alula:error`; `.reason` is the
  wire reason (`"forbidden"`, `"not_joined"`, …).
- `TimeoutError` — no reply before the deadline (`pushTimeoutMs`, default
  10 s). A handler that returns `.none` never replies — use `send()` for
  those events.
- `DisconnectedError` — the connection dropped while awaiting the reply.
- `NotConnectedError` — pushed before `connect()` (or while closed).

### Reconnection

Reconnection is client-driven; the server holds no resumable session. On a
drop the socket re-dials per `reconnectDelayMs` (default: doubling backoff,
100 ms → 10 s, forever) and rejoins every joined channel. The fresh initial
state is delivered to listeners as a `"alula:join"` message. A rejected
rejoin (the gate closed while you were away) arrives as `"alula:error"`
and stops retrying that topic. `disconnect()` and a server `alula:close`
are terminal — no reconnection until `connect()` is called again.

```js
const socket = new AlulaSocket(url, {
  heartbeatIntervalMs: 25_000, // keep well inside the server's 60 s timeout
  pushTimeoutMs: 10_000,
  reconnectDelayMs: exponentialBackoff({ initialMs: 100, maxMs: 10_000 }),
  webSocket: WebSocket,        // injectable (Node, tests)
});
socket.onStateChange((state) => {
  // "connecting" | "connected" | "disconnected" | "closed"
});
```

Heartbeats ride `alula:heartbeat` on the reserved `"alula"` topic; an
unanswered heartbeat is treated as a dead connection and triggers the
reconnect path.

## Tests

```sh
npm test   # node --test; zero dependencies
```

The suite drives the client against a scripted in-memory server speaking
the same wire fixtures the Swift test suite asserts on — one protocol,
three artifacts, versioned together.
