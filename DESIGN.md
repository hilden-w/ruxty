# ruxty — Design Document

## Goal

Learn systems and network programming in Rust by building a TCP proxy,
architected from the start to extend toward HTTP-aware routing, pluggable
filter chains, and xDS-based dynamic configuration (as in Envoy).

---

## MVP Scope

A TCP reverse proxy that:

- Listens on a configured local address and port
- Accepts inbound TCP connections
- Dials a single configured upstream for each accepted connection
- Bidirectionally copies bytes between client and upstream
- Shuts down gracefully on SIGINT/SIGTERM

No HTTP parsing. No TLS. No config file. No filters. No routing.
The MVP exists to force engagement with async I/O, socket lifecycle,
backpressure, and task management — before any abstraction hides them.

---

## Architecture Overview

Three concerns, kept separate from day one:

**Listener** — binds the port, accepts connections, dispatches work.

**Connection handler** — owns one client socket and one upstream socket for
the lifetime of a connection. Runs the bidir copy. Will be the site where
filter chain is invoked in future versions.

**Filter chain** — not present in MVP, but the module structure reserves
space for it. A no-op passthrough placeholder establishes the interface so
the handler never needs to change shape when real filters arrive.

These map directly to the module structure below.

---

## Module Structure

```
src/
  main.rs         Entry point. Wires config → proxy → run.
  config.rs       Config struct. Hardcoded values in MVP.
  proxy.rs        Listener loop. Accepts conns, spawns handler tasks.
  conn.rs         Per-connection logic. Upstream dial, bidir copy, filter hooks.
  shutdown.rs     OS signal handling. Broadcasts shutdown to tasks.
  filter/
    mod.rs        Filter trait, FilterChain, FilterResult types.
    noop.rs       Passthrough implementation. Used in MVP.
```

This layout is stable across all planned versions. xDS, HTTP, and TLS
extensions add new modules without restructuring existing ones.

---

## LLD Decisions

### 1. Async runtime

**Decision: tokio**

Alternatives considered: `async-std`, raw OS threads.

`async-std` has a cleaner API surface but a significantly smaller ecosystem
and slower development pace. Raw threads are conceptually simpler but do not
scale past a few thousand concurrent connections without a thread-per-conn
model that wastes memory and context-switch budget.

`tokio` is the de-facto standard. `tonic` (needed for xDS gRPC) requires it.
`hyper` (needed for L7) requires it. Choosing anything else forecloses the
extension path. The cost is that tokio's scheduler and task model are
non-trivial to understand — which is a feature, not a bug, given the learning
goal.

---

### 2. Connection model

**Decision: one tokio task per accepted connection**

Each accepted connection spawns a task. The task owns both sockets for its
lifetime. When either socket closes or the shutdown signal fires, the task
exits and both sockets are dropped.

Alternative: a fixed worker pool where tasks are recycled. This adds
complexity (work-stealing queues, task state machines) with no benefit at
proxy scale. Tokio tasks are cheap — spawning per connection is idiomatic and
scales to hundreds of thousands of concurrent connections on modern hardware.

---

### 3. Filter chain design

**Decision: trait object chain (dynamic dispatch)**

This is the most consequential architectural decision for the extension path.

Two options:

**Static dispatch (generics):** The proxy type is parameterized over a filter
type. The compiler monomorphizes and inlines. Zero runtime overhead.
Downside: the filter chain is fixed at compile time. xDS requires adding and
removing filters at runtime without restarting the process. Static dispatch
makes this impossible without significant workarounds.

**Dynamic dispatch (trait objects):** The filter chain holds a `Vec` of boxed
trait objects. Filter calls go through a vtable. The overhead per filter
invocation is one pointer indirection — negligible compared to any network
I/O cost.

Dynamic dispatch wins because it matches the operational requirement: xDS is
about runtime reconfiguration. Envoy itself uses dynamic dispatch for its
filter chain. The performance argument for static dispatch does not apply to
proxy workloads.

The `Filter` trait should be minimal at MVP: a single method called at
connection establishment. HTTP-layer hooks (on headers, on body chunk) are
added to the trait in the L7 version.

---

### 4. Protocol layer for MVP

**Decision: L4 (raw TCP), not L7 (HTTP)**

Starting at L4 means writing and debugging the async bidir copy loop,
understanding how tokio splits a `TcpStream` into read/write halves, and
observing backpressure firsthand. None of this is visible if `hyper` manages
the connection.

L7 is version 2. At that point the learning goal shifts from "how do sockets
work in async Rust" to "how do you parse and route HTTP without copying
bytes unnecessarily." That transition is more meaningful after L4 is
understood.

---

### 5. Configuration

**Decision: hardcoded struct in MVP, layered toward xDS**

Three stages:

- **MVP:** `Config` is a plain Rust struct with values baked into `main`.
  No file, no parsing, no validation. The struct establishes the shape of
  what the proxy needs to know.

- **V2:** Config loaded from a TOML or YAML file via `serde`. Adds file
  watching for hot reload of static config.

- **V3:** xDS via gRPC to a control plane (Envoy-compatible). The control
  plane pushes `Listener`, `Cluster`, and `RouteConfiguration` resources.
  The proxy applies them at runtime without restart.

Skipping directly to xDS from a hardcoded struct is a mistake. The xDS
protocol assumes you already have a working mental model of what each
resource type controls. Building that model bottom-up through stages 1 and 2
makes stage 3 tractable.

---

### 6. Error handling

**Decision: typed errors with `thiserror`, no panics in connection path**

Connection-level errors (upstream unreachable, copy interrupted, timeout)
must not crash the process. Each connection handler returns a `Result`. The
listener logs the error and moves on.

`thiserror` is used to define typed error enums at each layer boundary.
`anyhow` is acceptable in `main` for top-level wiring but should not leak
into library-style modules.

---

### 7. Observability

**Decision: structured logging with `tracing` from day one**

`tracing` is the correct choice over `log` because it supports spans —
structured context that propagates through async tasks. A connection span
established at accept time can carry connection ID, client address, and
upstream address through every log line emitted during that connection's
lifetime without threading those values through every function signature.

This matters for xDS and filter debugging later: you want to know which
filter emitted which log line for which connection.

---

## Extension Roadmap

### V1 — Filter chain (first real extension)

Introduce a concrete filter: a per-connection rate limiter or a connection
logger that records bytes transferred. The goal is to exercise the trait
object chain with real state, not just validate that the interface compiles.

### V2 — HTTP/L7

Replace the raw bidir copy with an HTTP/1.1 parser. Route requests based on
Host header or path prefix. Expand the `Filter` trait with HTTP-layer hooks.
Introduces `hyper` as a dependency.

### V3 — TLS termination

Accept TLS on the inbound side using `tokio-rustls` and `rustls`. No
OpenSSL. `rustls` is memory-safe and does not require a C toolchain.
Introduces certificate loading, SNI routing, and ALPN negotiation.

### V4 — xDS dynamic config

Implement an xDS client using `tonic` (gRPC) and `prost` (protobuf codegen).
Subscribe to Listener Discovery Service (LDS) and Cluster Discovery Service
(CDS) from a control plane. Apply resource updates to the running proxy
without restart. This is the payoff for having dynamic dispatch and a
staged config model from the start.

---

## Dependency Rationale

| Crate | Purpose | When |
|---|---|---|
| `tokio` | Async runtime, TCP sockets, task scheduling | MVP |
| `tracing` | Structured, async-aware logging | MVP |
| `tracing-subscriber` | Log formatting and output | MVP |
| `thiserror` | Typed error definitions | MVP |
| `hyper` | HTTP/1.1 and HTTP/2 framing | V2 |
| `tokio-rustls` | Async TLS via rustls | V3 |
| `rustls` | TLS implementation, no C deps | V3 |
| `tonic` | gRPC client/server on tokio | V4 |
| `prost` | Protobuf codegen for xDS types | V4 |
| `serde` | Serialization for config | V2 |

---

## Learning Milestones

1. **MVP complete:** Understand tokio task model, TCP socket lifecycle,
   async bidir copy, backpressure, graceful shutdown.

2. **First filter:** Understand trait objects, dynamic dispatch, shared
   state across async tasks (`Arc`, `Mutex` vs `RwLock`), lifetime of
   filter state relative to connection lifetime.

3. **L7 proxy:** Understand protocol framing, zero-copy goals, HTTP
   connection reuse, why parsing is expensive.

4. **TLS:** Understand handshake cost, certificate management, SNI,
   the difference between termination and passthrough.

5. **xDS:** Understand control plane / data plane split, eventual
   consistency in distributed config, the cost of dynamic reconfiguration
   on hot paths.
