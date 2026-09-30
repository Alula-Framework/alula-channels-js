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

## Installing

`@alula-framework/channels` is not published to npm. Install it from the
Git repository:

```sh
npm install github:Alula-Framework/alula-channels-js
```

or copy `src/` (and `types/` for TypeScript) into your project; there are
no dependencies to bring with it. The server side is alula's
`AlulaChannels` product; the protocol both speak is its
`AlulaChannelsProtocol` target (`Envelope`, `ReservedEvent`,
`ChannelErrorReason`, `ChannelCloseCode`).

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
  wire reason. The server's reasons: `unauthenticated` and `forbidden`
  (the join gate), `unmatched_topic` (no channel serves the topic),
  `already_joined`, `too_many_topics`, `reserved_topic` (the `"alula"`
  control topic), `not_joined` (a push or leave on a topic this socket
  hasn't joined), `invalid_event` (an application event starting with
  `alula:`), and `handler_error` (the channel failed; the detail stays in
  the server log).
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
and stops retrying that topic, with the server's reason (`forbidden`,
`unauthenticated`, …) as its payload. `disconnect()` is terminal — no
reconnection until `connect()` is called again; it sends `alula:close`,
which only ever travels client to server.

The server ends a socket by closing the WebSocket, and every server close
is a drop, handled alike whatever its code: pending pushes reject with
`DisconnectedError`, the state goes to `"disconnected"`, and the socket
reconnects and rejoins, reaching `"closed"` only if `reconnectDelayMs`
gives up. The code says why — `1000` normal closure, `1001` the server
is shutting down, `4000` heartbeat timeout, `4400` protocol violation,
`4408` the client stopped reading, `4410` the client fell too far behind
and frames would have been dropped — and in each case the rejoin brings
fresh state.

```js
const socket = new AlulaSocket(url, {
  heartbeatIntervalMs: 25_000, // inside the server's heartbeat timeout (default 60 s)
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
reconnect path. The server's side is `channels.heartbeat-timeout-seconds`
(default 60): keep `heartbeatIntervalMs` well below it.

## Presence

`@alula-framework/channels/presence` keeps a topic's Alula Presence list
from the server's `alula:presence_state` and `alula:presence_diff`
messages, so application code sees a list rather than diffs:

```js
import { AlulaPresence } from "@alula-framework/channels/presence";

const room = socket.channel("room:42");
const presence = new AlulaPresence(room);
presence.onChange(({ list, joins, leaves }) => render(list));
await room.join();   // the server sends the state, then diffs
```

`list()` is sorted by key; each entry's metas carry the `ref` plus the
tracked payload, whose values are strings. `joins` and `leaves` are the
net change: a meta updated in place appears in `joins` only. The rules
match alula's Swift `PresenceSync` (in `AlulaPresenceProtocol`).

## Tests

```sh
npm test   # node --test; zero dependencies
```

The suite drives the client against a scripted in-memory server and
asserts on exact frames, as alula's Swift suites do: one protocol, a
server and two clients, versioned together.
