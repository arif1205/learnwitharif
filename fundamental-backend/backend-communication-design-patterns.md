# Backend Communication Design Patterns

How backends talk — to clients and to each other. Every protocol (HTTP, gRPC, WebSockets, queues) is built from these base patterns.

---

## 1. Request–Response model

The classic. Client sends request -> server parses -> processes -> responds -> client parses -> consume.

- Client sends a request - simple
- Server parses the request - Just to understand this is the start of the request and this is the end of request. Nothing less or more.
- Server processes the request - Where the de-serialization of the body, and other things processed.
  Deserialization is costly, thats why people moves from soap (xml) to rest (json). JSON parsing is simpler but still heavy, thats why peoples are moving to protocol buffers (quick parsers)

**Where it is used?:**

- Web, HTTP, DNS, SSH - request response protocols
- RPC (remote procedure call) - send req -> call the remote server method
- SQL and Database protocols - same req res model
- APIs (REST, SOAP, GraphQL)

**Anatomy:**

- A request structure is defined by both client and server
- A fixed structure, ex. Method + PROTOCOL with Version + CRLF + BODY
- Defined by a protocol and message format (For http, it is just text)
- Any other message format needs a cost to parsing.

## 2. Synchronous vs Asynchronous Workloads 🔲

By design, everything was synchronous at the begining.

- Sync: blocked until response: Remove from CPU when the process is blocked.
- Async: fire, get notified later (callbacks, promises, epoll). It works like magic, whenever a blocking operation like "reading a file" happens, it just immediately spins up a new thread for that blocking part, and make the main thread free to go on.
- Synchronicity is a client property.
- Async in the real world: async commits, async replication, async I/O
- Sync/async is a spectrum — client can be sync while backend is async
- Database always open a transacation for any query or mutations
- **Interesting:** Database complete the transaction in the memory wall, after everything is completed, it commits (Sent to the disk for save) and returns success to the client. But it can be later failed to write in the disk for any reason. Dangerous but beautiful in design. Because writing in the disk is costly, so basically it trys to wait for some nano seconds, and get a bunch of commits and write and flush the disk at a time.

## 3. Push 🔲

Server sends data without client asking.

- Requires a persistent connection (WebSockets)
- Pros: real-time
- Cons: client must be online, must handle load, doesn't scale to heavy clients
- Example: RabbitMQ push API, chat apps

## 4. Short Polling 🔲

Client repeatedly asks "is it done yet?"

- Request → server returns handle → client polls with handle
- Pros: simple, works everywhere
- Cons: chatty, wasted requests, network noise

## 5. Long Polling 🔲

Ask and wait — server holds the request until data is ready.

- Poll but server only responds when there's data
- Kafka uses this
- Pros: less chatty
- Cons: not fully real-time, timeouts to manage

## 6. Server-Sent Events (SSE) 🔲

One request, an endless response stream.

- Response never ends; events streamed as chunks
- Built on plain HTTP
- Pros: real-time, simple client (`EventSource`)
- Cons: one-way only, client must stay connected, HTTP/1.1 6-connection limit

## 7. Publish–Subscribe (Pub/Sub) 🔲

Decouple producers from consumers via a broker.

- One publisher, many consumers, neither knows the other
- Kafka / RabbitMQ examples
- Pros: scales, decoupled, works offline
- Cons: message delivery guarantees, complexity, two points of failure

## 8. Multiplexing vs Demultiplexing 🔲

Many streams over one connection — or one stream fanned out to many.

- HTTP/2 multiplexing vs HTTP/1.1
- Connection pooling (demux)
- Browser 6-connection limit
- Proxy vs reverse proxy behavior

## 9. Stateful vs Stateless 🔲

Does the backend remember you?

- Stateful: state stored on server (sticky sessions)
- Stateless: client carries state (JWT), any server can answer
- Is a protocol stateful or stateless? (TCP stateful, HTTP stateless, QUIC?)
- What "stateless backend" really means: restart test

## 10. Sidecar Pattern 🔲

Put a proxy next to your app to handle the network layer.

- Thick client library problem → move it to a sidecar proxy
- Service mesh (Envoy, Linkerd)
- Pros: language-agnostic upgrades, one place for TLS/retries
- Cons: latency hop, complexity

---

## Resources

- Slides: [resources/Fundamentals+of+Backend+Engineering.pdf](resources/Fundamentals+of+Backend+Engineering.pdf)
- Kafka long polling: [resources/Apache-kafka-long-polling.pdf](resources/Apache-kafka-long-polling.pdf)
- RabbitMQ push: [resources/rabbitMQ-pushAPI.pdf](resources/rabbitMQ-pushAPI.pdf)
- Chrome 6-connection limit: [resources/Chrome-6-Connections-Limit.pdf](resources/Chrome-6-Connections-Limit.pdf)
