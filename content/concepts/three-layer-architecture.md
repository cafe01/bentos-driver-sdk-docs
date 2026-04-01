---
title: "Three-Layer Architecture"
description: "The SDK's layered design: from raw FUSE ops to domain callbacks"
weight: 10
toc: true
---

The BentOS Driver SDK has three layers. Each serves a different audience and handles a different level of abstraction.

```
Layer 1: Base Driver Protocol       raw FUSE ops over socket
    Who: framework/binding authors (few)

Layer 2: Pattern Framework          per-I/O-pattern state machine
    Who: subsystem authors (one per device category)

Layer 3: Ops Contract               domain callbacks only
    Who: driver developers (many -- this is you)
```

## Layer 3: Ops Contract (Driver Developers)

**This is where most drivers live.** You implement an ops contract -- a small set of domain callbacks -- and the pattern framework handles everything else.

An ops contract has these properties (borrowed from the Linux device model):

- **Immutable** -- once constructed, the ops don't change
- **Stateless** -- ops carry no per-session state; state lives in the runtime
- **Shareable** -- multiple device instances can share the same ops
- **Composable** -- individual callbacks can be wrapped or decorated independently

Example: the `StreamOps` contract for Pattern 1 (Pure Stream) has four callbacks:

| Callback | Required | What it does |
|---|---|---|
| `onData` | yes | Handle bytes from `write()`. Return bytes consumed. |
| `outputStream` | no | Provide a stream for `read()` to pull from. |
| `onSessionStart` | no | Allocate per-session state. |
| `onSessionEnd` | no | Clean up per-session state. |

No FUSE vocabulary. No file handles. No protobuf. The driver never sees `FuseMessage`, `open`, `release`, or `fh`. The framework translates.

## Layer 2: Pattern Framework (Subsystem Authors)

Each I/O pattern has a framework that translates between FUSE ops and domain callbacks. The framework handles:

- **State machine enforcement** -- invalid transitions return the right errno
- **Session management** -- mapping file handles to domain sessions
- **Buffering** -- accumulating writes, chunking reads
- **Poll readiness** -- computing POLLIN/POLLOUT from state
- **Error translation** -- domain errors to POSIX errno

The four pattern frameworks are:

| Pattern | Framework | Use case |
|---|---|---|
| P1: Pure Stream | `StreamDriver` | Bidirectional byte pipe |
| P2: Write-then-Read | `WriteReadDriver` | Request/response |
| P3: Event Stream | `EventStreamDriver` | Read-only event broadcast |
| P4: Configured Stream | `ConfiguredStreamDriver` | Config phase + streaming output |

See [Choosing a Pattern](/concepts/choosing-a-pattern/) for guidance on which to pick.

## Layer 1: Base Driver Protocol (Framework Authors)

The raw socket protocol. `BentosDriver` handles framing, dispatch, and connection management. You get direct access to every FUSE op:

```dart
final driver = BentosDriver(
  onOpen:    (req, ctx) => ...,
  onRead:    (req, ctx) => ...,
  onWrite:   (req, ctx) => ...,
  onFlush:   (req, ctx) => ...,
  onRelease: (req, ctx) => ...,
  onIoctl:   (req, ctx) => ...,
  onPoll:    (req, ctx) => ...,
);
```

Each callback receives the FUSE request protobuf and a `DriverContext` (file handle + connection). Return a `FuseResponse`. Return null (by omitting a callback) to reply with ENOSYS.

**You should almost never need Layer 1.** It exists for:
- Building new pattern frameworks
- Diagnostic tools (see the `playground_driver` example)
- Devices that don't fit any pattern

## How bentosd connects

The driver listens on a Unix domain socket. `bentosd` connects when mounting the device. Each `open()` from an application maps to a new logical session (identified by `fh`). The driver processes requests sequentially per session, concurrently across sessions.

```
Application          bentosd              Driver
    |                   |                   |
    |-- open() -------->|                   |
    |                   |-- FuseRequest --->|
    |                   |<-- FuseResponse --|
    |                   |                   |
    |-- write("hi") --->|                   |
    |                   |-- FuseRequest --->|
    |                   |<-- FuseResponse --|
    |                   |                   |
    |-- read() -------->|                   |
    |                   |-- FuseRequest --->|
    |                   |<-- FuseResponse --|
    |                   |                   |
    |-- close() ------->|                   |
    |                   |-- FuseRequest --->|
    |                   |<-- FuseResponse --|
```

The driver never deals with CUSE/FUSE kernel interfaces. bentosd absorbs all of that. The driver speaks only the length-prefixed protobuf protocol over the socket.
