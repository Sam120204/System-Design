# WebSockets

WebSockets provide a long-lived, bidirectional message channel between a client and a server. Either side can send text or binary messages after the connection opens. This fits interactive applications such as chat, collaborative editing, and control interfaces where both sides send frequent updates.

[![WebSocket ticker example showing client subscribe and unsubscribe messages and server price updates](https://www.hellointerview.com/_next/static/media/DElga39lrtz-.1cd_jfppgwy_q.svg)](images/websocket-messages.png)

## Connection lifecycle

1. With HTTP/1.1, the client sends an HTTP Upgrade request and the server can accept with `101 Switching Protocols`. HTTP/2 can establish WebSockets with extended CONNECT when both endpoints and intermediaries support it.
2. Both sides send messages using WebSocket frames. The protocol includes message framing, ping and pong control frames, and a close handshake.
3. After a disconnect, the client opens a new connection, reauthenticates or reauthorizes as required, and restores subscriptions or resumes from a cursor using application messages.

Use `wss://` for TLS protection. Authenticate the connection, validate the browser's Origin where applicable, and authorize subscriptions or room joins. Check proxy support and align heartbeat timing with idle timeouts. [WebSocket protocol](https://www.rfc-editor.org/rfc/rfc6455.html) · [WebSockets over HTTP/2](https://www.rfc-editor.org/rfc/rfc8441.html)

## Example of live price updates

1. A user's browser sends `{"type":"subscribe","topic":"prices:AAPL"}` over its WebSocket. The app defines the `subscribe` message and the topic naming convention.
2. The WebSocket server checks whether the user may receive that price feed and records the socket under `prices:AAPL`.
3. A backend worker processes a market-data update and publishes the result to `prices:AAPL`.
4. The WebSocket server looks up the sockets subscribed to `prices:AAPL` and pushes the price update to each of them.

The WebSocket server plays a broker-like role at the browser edge. It tracks topic subscriptions and fans events out to matching sockets. With one server, an in-memory map from topics to sockets can handle that routing. With several gateways, a shared broker such as Redis Pub/Sub can send an update to each gateway subscribed to the topic. Each gateway then delivers the update to its own sockets. WebSocket defines the connection and message framing. Topic names, subscriptions, and delivery rules belong to the app.

A price screen that only needs the latest quote can resubscribe after reconnecting, fetch a snapshot, and apply the newest version it receives. A screen that needs every tick requires a replayable data source and a cursor.

## Scale connections and preserve message semantics

- **Routing and fan-out.** Each live connection stays on one gateway for its lifetime. Remove inactive sockets from the local subscription map and keep durable message history accessible to other gateways.
- **Durability.** A client can send a unique message ID and receive an application acknowledgment after a durable commit. On reconnect, it can retry unacknowledged messages with the same IDs and fetch missed messages after its last confirmed cursor.
- **Slow clients.** Set bounds on outbound queues. Drop or coalesce ephemeral state such as typing indicators. If a client cannot keep up with durable updates, closing the stream and resuming from stored history can preserve correctness.
- **Deployments.** Drain old instances by stopping new handshakes, allowing active sessions time to finish, and staggering reconnects. Track active connections, message rates, queue sizes, and reconnect rates.

## Example of bidirectional chat

A browser can send and receive messages over the same connection. One browser sends a chat message to the server, which validates it and forwards it to the other browsers in the room. All browsers connect to the service, so messages travel through the server between them.

For durable chat, the gateway calls a service that stores the message, acknowledges the sender after commit, and publishes an event to the subscribed gateways. If a browser misses the push, it can catch up from conversation history using its last confirmed cursor. The app can track server persistence, receipt by the recipient's device, and read receipts as separate states. [Slack design case](<../../06-Instant Messaging System/slack-design-group-chat.md>)
