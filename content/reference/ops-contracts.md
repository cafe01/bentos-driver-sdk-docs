---
title: "Ops Contracts"
description: "Complete callback reference for all patterns and the base driver"
weight: 20
toc: true
---

Every driver implements an ops contract -- a set of callbacks that define the driver's domain behavior. This page lists all ops contracts in one place.

## P1: StreamOps

**Framework**: `StreamDriver<S>`

| Callback | Required | Signature | Description |
|---|---|---|---|
| `onData` | **yes** | `(List<int> data, {S? session}) -> int` | Bytes from `write()`. Return bytes consumed. |
| `outputStream` | no | `({S? session}) -> Stream<Uint8List>` | Output stream for `read()`. |
| `onSessionStart` | no | `(int flags) -> S` | Allocate per-session state. |
| `onSessionEnd` | no | `({S? session}) -> void` | Clean up per-session state. |

## P2: WriteReadOps

**Framework**: `WriteReadDriver<S>`

| Callback | Required | Signature | Description |
|---|---|---|---|
| `onRequest` | **yes** | `(Uint8List input, {S? session}) -> Future<Uint8List>` | Process complete request, return complete response. |
| `onSessionStart` | no | `(int flags) -> S` | Allocate per-session state. |
| `onSessionEnd` | no | `({S? session}) -> void` | Clean up per-session state. |

## P3: EventStreamOps

**Framework**: `EventStreamDriver<E>`

| Callback | Required | Signature | Description |
|---|---|---|---|
| `encodeEvent` | **yes** | `(E event) -> Uint8List` | Serialize one event to bytes. |
| `onActivate` | no | `() -> void` | First listener opened -- start producing. |
| `onDeactivate` | no | `() -> void` | Last listener closed -- stop producing. |

## P4: ConfiguredStreamOps

**Framework**: `ConfiguredStreamDriver<C, I, O, S>`

| Callback | Required | Signature | Description |
|---|---|---|---|
| `defaultConfig` | **yes** | `() -> C` | Default configuration for new sessions. |
| `process` | **yes** | `(I input, C config, {S? session}) -> Stream<O>` | Core operation. Takes input + config, yields output chunks. |
| `encodeOutput` | **yes** | `(O chunk, {C config}) -> Uint8List` | Serialize one output chunk for `read()`. |
| `decodeInput` | **yes** | `(Uint8List data, {C config}) -> I` | Deserialize `write()` bytes into domain input. |
| `onCancel` | no | `({S? session}) -> void` | Abort in-flight processing (called on DROP ioctl). |
| `onDrain` | no | `({S? session}) -> void` | Graceful completion hook. |
| `onSessionStart` | no | `() -> S` | Allocate per-session state. |
| `onSessionEnd` | no | `({S? session}) -> void` | Release per-session state. |

### ConfigCodec (required for P4)

P4 also requires a `ConfigCodec<C>`:

| Method | Signature | Description |
|---|---|---|
| `apply` | `(C current, int command, Uint8List data) -> C` | Apply an ioctl command to config. Return new config. |
| `encode` | `(C config, int command) -> Uint8List` | Encode config field for ioctl query response. |

## L1: BentosDriver (Raw)

Direct FUSE op callbacks. Each receives the request protobuf and a `DriverContext`.

| Callback | Signature | Description |
|---|---|---|
| `onOpen` | `(OpenReq, DriverContext) -> FuseResponse?` | New fd opened |
| `onRead` | `(ReadReq, DriverContext) -> FuseResponse?` | `read()` called |
| `onWrite` | `(WriteReq, DriverContext) -> FuseResponse?` | `write()` called |
| `onFlush` | `(FlushReq, DriverContext) -> FuseResponse?` | fd closing |
| `onRelease` | `(ReleaseReq, DriverContext) -> FuseResponse?` | Last fd reference closed |
| `onIoctl` | `(IoctlReq, DriverContext) -> FuseResponse?` | ioctl command |
| `onPoll` | `(PollReq, DriverContext) -> FuseResponse?` | Readiness query |
| `onFsync` | `(FsyncReq, DriverContext) -> FuseResponse?` | `fsync()` called |

Return `null` (by omitting the callback) to reply with ENOSYS.

## Ops contract properties

All ops contracts share these properties (inherited from the Linux device model):

- **Immutable**: Once constructed, callbacks don't change
- **Stateless**: No per-session state in the ops object -- state lives in session context
- **Shareable**: Multiple device instances can share one ops object
- **Composable**: Individual callbacks can be wrapped or decorated independently
