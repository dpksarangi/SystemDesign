# Real-Time Communication — Interview Notes

## 1. Communication Options

### Short Polling

Client repeatedly sends HTTP requests at a fixed interval.

```text
Client ── request ──> Server
Client <── response ── Server
        wait
Client ── request ──> Server
```

**Best for**
- Periodic status checks
- Simple dashboards
- Low-frequency updates
- Cases where real-time delivery is not required

**Problems**
- Generates requests even when nothing changes
- Latency depends on polling interval
- Can create significant unnecessary load at scale

**Mental model:**  
> "Tell me if anything changed."

---

## 2. Long Polling

Client sends a request and the server keeps it open until an event is available or a timeout occurs.

```text
Client ───────────────> Server
                         ...
                         wait
                         ...
Client <─────────────── Server

Client ───────────────> Server
```

**Best for**
- Near-real-time notifications
- Legacy systems
- Environments where SSE/WebSocket is unavailable

**Advantages**
- Less wasteful than short polling
- Works over normal HTTP

**Problems**
- Every event requires another HTTP request
- More connection/request lifecycle overhead
- Reconnect and timeout handling are application concerns

**Mental model:**  
> "Don't answer until you have something to tell me."

---

## 3. HTTP Streaming / Chunked Response

The server progressively sends pieces of one HTTP response instead of waiting to construct the entire response.

```text
Client ── HTTP request ──> Server

Client <── response chunk 1
Client <── response chunk 2
Client <── response chunk 3
Client <── response chunk 4
                  ...
Client <── response complete
```

Common use cases:

- LLM/token streaming
- Large generated responses
- Progressive report generation
- Long-running processing results
- Streaming API responses

### Important distinction

HTTP chunked transfer is a **transport mechanism**, not an application-level event protocol.

`multipart/mixed` is a response format that can structure multiple MIME parts; it is different from chunked transfer itself.

**Mental model:**  
> "Progressively send the response to this request."

---

## 4. SSE — Server-Sent Events

SSE keeps an HTTP connection open and allows the server to continuously send events.

```text
Client ── connect ──> Server

Client <── event 1 ── Server
Client <── event 2 ── Server
Client <── event 3 ── Server
Client <── event 4 ── Server
                    ...
```

Direction:

```text
Server ─────────────> Client
```

**Best for**
- Live comments
- Notifications
- Live dashboards
- Activity feeds
- Job/progress updates
- Price/status updates
- News/event streams

**Advantages**
- Simple HTTP-based model
- Native browser `EventSource`
- Built-in browser reconnect behavior
- Natural fit for server → client streaming

**Limitations**
- Primarily one-way
- Client → server communication still uses normal HTTP APIs
- Long-lived connection still needs scaling and failure handling

**Mental model:**  
> "Keep this connection open and tell me whenever something happens."

---

## 5. WebSocket

WebSocket provides a persistent bidirectional connection.

```text
Client ═══════════════ Server
       ⇄
```

Both sides can send messages at any time.

**Best for**
- Chat
- Multiplayer games
- Collaborative editing
- Presence
- Typing indicators
- Real-time control systems
- Applications with frequent client → server and server → client messages

**Advantages**
- Bidirectional
- Low latency
- Efficient for continuous interactive communication

**Costs**
- More connection lifecycle management
- Application-level reconnect/heartbeat handling
- More complex connection state and scaling

**Mental model:**  
> "Give both sides a persistent communication channel."

---

## 6. WebTransport

WebTransport is a modern HTTP/3 + QUIC-based transport that supports bidirectional communication, reliable streams, and datagrams.

**Useful for**
- Very low-latency applications
- Modern multiplayer/gaming workloads
- Applications that benefit from multiple independent streams
- Cases where unreliable datagrams are useful

For most system-design interviews, do not choose WebTransport by default. Mention it when the requirements specifically call for modern QUIC-based, low-latency communication.

**Mental model:**  
> "Modern QUIC-based bidirectional transport with streams and datagrams."

---

# 7. Comparison

| Mechanism | Direction | Persistent | Main Use |
|---|---|---:|---|
| Short Polling | Client → Server | No | Periodic checks |
| Long Polling | Client → Server request, delayed response | Request-level | Near-real-time fallback |
| HTTP Streaming | Server → Client response | Until response completes | Progressive response generation |
| SSE | Server → Client | Yes | Continuous server events |
| WebSocket | Bidirectional | Yes | Interactive real-time systems |
| WebTransport | Bidirectional | Yes | Advanced low-latency/QUIC use cases |

---

# 8. Decision Tree

```text
Need real-time updates?
        |
       No
        |
   Short Polling

        Yes
        |
        v
Is the main requirement
server → client streaming?
        |
     +--+--+
     |     |
    Yes    No
     |      |
    SSE   Bidirectional?
             |
          +--+--+
          |     |
         Yes    No
          |      |
      WebSocket  Long Polling
```

HTTP streaming is a separate pattern when the requirement is to progressively produce **one response**, rather than maintain an ongoing event stream.

WebTransport is an advanced option when QUIC/HTTP3-specific low-latency capabilities are valuable.

---

# 9. How to Choose in an Interview

Do not choose a protocol because it is "faster."

Choose based on:

1. **Directionality**
   - Server → client?
   - Client ↔ server?

2. **Connection lifetime**
   - One request?
   - Long-lived stream?

3. **Event frequency**
   - Occasional?
   - Continuous?

4. **Latency requirement**
   - Seconds acceptable?
   - Near-instant delivery required?

5. **Browser/client support**
   - What clients need to connect?

6. **Operational complexity**
   - Reconnection
   - Heartbeats
   - Load balancing
   - Connection state
   - Failure recovery

### Key rule

> Match the communication model to the requirement, rather than defaulting to WebSocket for every real-time problem.

---

# 10. Live Video Comments Example

For the live comments system:

```text
Client → Server
    POST comment/reply
          ↓
       HTTP API

Server → Client
    new comments
          ↓
         SSE
```

Why not WebSocket?

Because the real-time requirement is primarily:

```text
Server ─────────────> Client
```

The client does not need a persistent bidirectional channel for posting comments.

Why not polling?

Because high-scale polling creates unnecessary requests and adds latency.

Why not HTTP streaming?

Because we want an ongoing **event stream**, not just a progressively generated response to one request.

Therefore:

> **HTTP for writes + SSE for live delivery + cursor-based HTTP GET for history/recovery.**

This combination keeps the design simple while matching the actual communication requirements.

---

# 11. Capacity Planning — Long-Lived Connections

A common interview question is:

> **How many SSE/WebSocket connections can one server handle?**

There is no universal fixed number. It depends heavily on:

- Server memory
- CPU
- Connection implementation
- TLS termination
- Message frequency
- Message size
- Kernel/file-descriptor limits
- Network bandwidth
- Runtime/framework
- Per-connection application state

The important point is to **calculate from resource constraints instead of quoting a magic number**.

## Memory-based estimation

Assume:

```text
Available memory for connections = 8 GB
Estimated memory per connection = 50 KB
```

Then:

```text
8 GB ≈ 8,000 MB
50 KB ≈ 0.05 MB

connections ≈ 8,000 / 0.05
            ≈ 160,000
```

But this is only a theoretical estimate.

Reserve memory for:

- JVM/runtime
- Application heap
- Buffers
- Thread/runtime structures
- Connection metadata
- GC overhead
- OS
- Safety margin

If only 4 GB is realistically available:

```text
4 GB / 50 KB ≈ 80,000 connections
```

A production design might deliberately target much lower capacity, for example 40–60K connections/server, after load testing and safety margins.

> **The number is an engineering estimate, not a protocol limit.**

---

# 12. Network-Based Capacity

Connection count is not enough. The message rate matters.

Suppose:

```text
100,000 active connections
Average event = 500 bytes
Each client receives 1 event/sec
```

Then outbound traffic is approximately:

```text
100,000 × 500 bytes × 1/sec
= 50 MB/sec
≈ 400 Mbps
```

At:

```text
5 events/sec
```

the same server would need approximately:

```text
2 Gbps
```

So a server that can maintain 100K idle connections may **not** be able to deliver 100K high-frequency streams.

### Key rule

> **Long-lived connection capacity and message throughput are separate capacity dimensions.**

---

# 13. File Descriptor Limits

Each TCP connection consumes a file descriptor.

If:

```text
ulimit -n = 100,000
```

then the practical connection limit cannot exceed that, and the application needs descriptors for other resources too.

For example:

```text
100,000 FD limit
- 10,000 internal/network/file descriptors
= ~90,000 possible client connections
```

Do not simply set the OS limit extremely high and assume the server can handle the resulting connections. Memory, CPU and network capacity still apply.

---

# 14. SSE vs WebSocket — Infrastructure Impact

An important interview question:

> **What changes if we replace SSE with WebSocket?**

The high-level infrastructure can remain almost identical:

```text
                    Load Balancer
                    /     |     \
                   /      |      \
                  v       v       v
                 S1      S2      S3
                  |       |       |
               SSE/WS   SSE/WS  SSE/WS
                  \       |       /
                   \      |      /
                    Redis / PubSub
```

The major change is at the connection/protocol layer.

## SSE → WebSocket changes

### Client

SSE:

```text
EventSource(url)
```

WebSocket:

```text
new WebSocket(url)
```

### Server endpoint

SSE:

```http
GET /events
Content-Type: text/event-stream
```

WebSocket:

```text
HTTP Upgrade
Connection: Upgrade
Upgrade: websocket
```

### Message model

SSE:

```text
server → event → client
```

WebSocket:

```text
server ⇄ message ⇄ client
```

### Reconnection

SSE:

- Browser EventSource provides basic reconnect behavior
- Application still needs cursor/recovery semantics

WebSocket:

- Application normally implements reconnect
- Backoff + jitter should be used
- Resume/recovery state needs to be designed

---

# 15. What Does NOT Need to Change When Moving SSE → WebSocket?

A good architecture should make this migration relatively localized.

These components can remain:

```text
Client
   |
   v
Load Balancer
   |
   v
BroadcastService
   |
   v
Redis Pub/Sub
   ^
   |
CommentService
   |
   v
MongoDB
```

The following remain conceptually unchanged:

- MongoDB data model
- Comment APIs
- Reply APIs
- Cursor-based history API
- Redis Pub/Sub
- BroadcastService
- Load Balancer
- Horizontal scaling
- Failure recovery concept
- Client deduplication

Primarily the **real-time connection layer** changes.

This is a useful HLD principle:

> **Keep the business/event pipeline independent from the transport protocol.**

---

# 16. When Would We Actually Migrate SSE → WebSocket?

Suppose the product initially has:

```text
Client → Server
POST comment
POST reply

Server → Client
live comments
```

SSE is sufficient.

Later requirements become:

```text
Client ⇄ Server

typing indicators
presence
acknowledgements
read receipts
real-time reactions
live moderation commands
client-side subscriptions
```

Now bidirectional communication becomes important.

At that point:

```text
SSE
  ↓
WebSocket
```

becomes a reasonable migration.

---

# 17. Migration Strategy: SSE → WebSocket

Avoid changing the entire system at once.

### Phase 1

Introduce WebSocket support in BroadcastService:

```text
BroadcastService
 ├── SSE connections
 └── WebSocket connections
```

Both consume the same Redis events.

```text
Redis Pub/Sub
      |
      v
BroadcastService
   /          \
 SSE          WebSocket
```

### Phase 2

Move clients to WebSocket.

### Phase 3

Remove SSE after adoption is complete.

The important architectural property is:

> **Redis/event processing should not care whether the final transport is SSE or WebSocket.**

---

# 18. Example Capacity Calculation for Live Comments

Assume:

```text
1 million concurrent viewers
10 BroadcastService instances
```

If distributed evenly:

```text
1,000,000 / 10
= 100,000 connections/server
```

Now assume:

```text
Average comment event = 300 bytes
Average delivery rate = 0.2 events/sec/client
```

Approximate outbound traffic:

```text
100,000 × 300 × 0.2
= 6,000,000 bytes/sec
≈ 6 MB/sec
≈ 48 Mbps/server
```

This looks manageable from a network perspective.

But if the event rate becomes:

```text
2 events/sec/client
```

then:

```text
100,000 × 300 × 2
= 60 MB/sec
≈ 480 Mbps/server
```

Now network and serialization/fan-out costs become much more significant.

### Interview takeaway

Always calculate:

```text
connections/server
×
events/sec/client
×
average event size
```

to estimate outbound bandwidth.

---

# 19. Fan-Out Is Often More Important Than Connection Count

Suppose:

```text
Video V1
500,000 viewers
```

A single comment must be delivered to 500K connections.

If the viewers are spread over:

```text
10 broadcaster instances
```

each instance sends the event to approximately:

```text
500,000 / 10
= 50,000 clients
```

So:

```text
Redis
  |
  +--> B1 → 50K clients
  +--> B2 → 50K clients
  +--> ...
  +--> B10 → 50K clients
```

This is why the Load Balancer and horizontally scaled broadcaster tier are important.

---

# 20. SSE vs WebSocket — Practical Decision Table

| Requirement | SSE | WebSocket |
|---|---:|---:|
| Server → client stream | Excellent | Excellent |
| Client → server stream | HTTP separately | Excellent |
| Bidirectional realtime | No | Excellent |
| Simple browser integration | Excellent | Good |
| Automatic browser reconnect | Better | Application-managed |
| Live comments | Excellent | Excellent |
| Notifications | Excellent | Usually unnecessary |
| Chat | Possible but awkward | Excellent |
| Typing indicators | Possible with HTTP | Excellent |
| Presence | Possible | Excellent |
| Collaborative editing | Possible but unsuitable | Excellent |
| Protocol complexity | Lower | Higher |
| Migration from SSE | N/A | Moderate |

---

# 21. Interview Rule of Thumb

### Use Short Polling when:

> Updates are periodic and real-time isn't important.

### Use Long Polling when:

> You need near-real-time behavior but cannot use a persistent streaming mechanism.

### Use HTTP Streaming when:

> One request produces a progressively generated response.

### Use SSE when:

> The dominant requirement is continuous server → client events.

### Use WebSocket when:

> Both client and server need continuous real-time communication.

### Use WebTransport when:

> The application specifically benefits from HTTP/3/QUIC, multiple streams, or low-latency datagrams.

---

# 22. One Important Interview Warning

Never answer:

> "A server can handle 100K WebSocket connections."

or:

> "SSE supports 50K connections."

Those are not meaningful universal limits.

Instead say:

> **"I'll estimate the per-connection memory, file descriptors, CPU, and network bandwidth, then load-test to determine the safe connection density per instance. I'll horizontally scale the broadcaster tier based on that measured capacity."**

That is the Staff-level answer.



## WebSocket vs WebTransport

### WebSocket

Use WebSocket when the application needs **normal bidirectional realtime communication**.

```text
Client <======== persistent WebSocket connection ========> Server
```

Good use cases:
- Chat
- Typing indicators
- Presence
- Collaborative editing
- Realtime commands
- Multiplayer applications where a simple bidirectional channel is sufficient

Why choose it:
- Mature and widely supported
- Simple programming model
- Large ecosystem and operational experience
- Good fit for most bidirectional realtime applications

### WebTransport

WebTransport runs over **HTTP/3 + QUIC** and provides:
- Bidirectional communication
- Multiple independent reliable streams
- Unreliable datagrams
- QUIC-level connection behavior

```text
Client
   |
   | HTTP/3 + QUIC
   v
WebTransport Server
   |
   +-- reliable stream A
   +-- reliable stream B
   +-- datagrams
```

The important point is that WebTransport is not simply "WebSocket but faster."

Choose WebTransport when the application actually benefits from:
- Multiple independent streams
- Unreliable datagrams
- Very latency-sensitive communication
- QUIC-specific behavior
- Different reliability/ordering requirements for different message types

Typical examples:
- Advanced multiplayer games
- Realtime simulation
- Applications sending frequent state updates where some updates can be dropped
- Systems that benefit from independent streams

### Why QUIC can matter

WebSocket traditionally provides one ordered byte stream over TCP.

With TCP, packet loss can cause head-of-line blocking: later data waits for the missing data to be recovered.

QUIC provides independent streams within a connection, so loss on one stream does not have to block unrelated streams in the same way.

This matters only when the application can actually exploit that property.

### WebSocket vs WebTransport

| Requirement | WebSocket | WebTransport |
|---|---|---|
| Bidirectional communication | Yes | Yes |
| Simple realtime APIs | Excellent fit | Usually unnecessary |
| Mature ecosystem | Very strong | Newer |
| Multiple independent streams | No | Yes |
| Unreliable datagrams | No | Yes |
| QUIC/HTTP3 | No | Yes |
| Chat / presence / collaboration | Usually preferred | Usually unnecessary |
| Advanced realtime gaming | Can work | Can be a strong fit |
| Operational simplicity | Simpler | More involved |

### Interview decision rule

Do **not** choose WebTransport simply because it is newer.

Use this decision tree:

```text
Server → client only?
        |
       Yes
        |
        +--> Continuous events → SSE
        |
       No / bidirectional
        |
        +--> Ordinary realtime communication → WebSocket
        |
        +--> Need multiple independent streams
             or unreliable datagrams
             or QUIC-specific behavior
                    |
                   Yes
                    |
               WebTransport
```

Strong interview answer:

> "Our requirement is server-to-client live delivery, so SSE is sufficient. If we needed bidirectional realtime communication, I'd consider WebSocket. I'd choose WebTransport over WebSocket only if the application benefits from QUIC-specific capabilities such as independent streams or unreliable datagrams. Otherwise WebSocket's maturity and simpler operational model make it the more pragmatic choice."

The broader principle:

> **A more capable transport is not automatically a better architecture. Pay the additional complexity only when its capabilities solve an actual requirement.**

---

## Live Comments: Transport Decision

For the live-comments system:

```text
MongoDB
   |
CommentService
   |
Redis Pub/Sub
   |
BroadcastService
   |
   +---- SSE ------------> Clients
```

### Why SSE?

The primary requirement is:

**Server → client live comment delivery**

Clients create comments through normal HTTP APIs, while the broadcast service pushes new comments through SSE.

This keeps the design simple.

### When would we move to WebSocket?

If the product evolves to require several client→server realtime interactions, for example:
- Typing indicators
- Presence
- Reactions
- Read receipts
- Realtime moderation commands
- Other bidirectional events

Then WebSocket becomes a natural candidate.

The business pipeline can remain mostly unchanged:

```text
MongoDB → CommentService → Redis Pub/Sub → BroadcastService
                                                |
                                     SSE / WebSocket
                                                |
                                              Client
```

### When would we move to WebTransport?

Only if there is a stronger requirement such as:
- Multiple independent realtime streams
- Unreliable datagrams
- QUIC-specific latency/transport behavior

For ordinary live comments, WebTransport would add capabilities that the application does not need.

---

## Migration Principle

The transport should be treated as an implementation detail behind the realtime gateway/broadcast layer.

For example:

```text
                    +--> SSE
                    |
Redis → BroadcastService → WebSocket
                    |
                    +--> WebTransport
```

The event production path should not need to change just because the client transport changes.

What changes is primarily:
- Connection lifecycle
- Framing/protocol handling
- Backpressure
- Reconnect/resumption behavior
- Ordering/reliability semantics
- Client implementation

For WebTransport specifically, define which messages require:
- Reliable delivery
- Ordering
- Independent streams
- Unreliable datagrams

before choosing the transport.

