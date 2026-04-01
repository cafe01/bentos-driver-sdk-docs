---
title: "Wire Protocol"
description: "The transport layer between bentosd and drivers"
weight: 20
toc: true
---

Drivers communicate with `bentosd` over a Unix domain socket using length-prefixed protobuf messages. This page documents the wire format for binding authors and anyone who needs to understand the transport layer.

> Most driver developers never touch the wire protocol directly. The SDK handles framing and dispatch. This page is for framework authors and the curious.

## Transport framing

```
[4 bytes: payload length, big-endian uint32][payload: FuseMessage protobuf]
```

Every message is a `FuseMessage` envelope containing:

- **`id`** (uint64) -- request correlation ID, assigned by bentosd, echoed by the driver
- **`fh`** (uint64) -- file handle, multiplexes sessions over a single connection
- **`payload`** -- oneof: `FuseRequest` (bentosd to driver) or `FuseResponse` (driver to bentosd)

The protobuf schema (`fuse_wire.proto`) is the canonical wire format definition.

## Connection lifecycle

1. Driver listens on a Unix socket (e.g., `/run/bentos/drivers/echo.sock`)
2. `bentosd` connects when mounting the device
3. One TCP-like stream per connection, multiplexed by `fh`
4. Driver processes requests sequentially per `fh`, concurrently across `fh` values
5. Every request gets exactly one response with the same `id`

## FUSE operations

The driver SDK handles the **file ops** subset of FUSE:

| Op | Direction | Purpose |
|---|---|---|
| `open` | bentosd to driver | New file descriptor opened on device |
| `read` | bentosd to driver | Application called `read()` |
| `write` | bentosd to driver | Application called `write()` |
| `flush` | bentosd to driver | Application closing fd (may repeat via `dup`) |
| `release` | bentosd to driver | Last reference to fd closed |
| `ioctl` | bentosd to driver | Control plane command |
| `poll` | bentosd to driver | Application checking readiness |
| `fsync` | bentosd to driver | Application called `fsync()` |

These map directly to `file_operations` in the Linux kernel. The protobuf request/response types for each are defined in `fuse_wire.proto`.

## Request/response contract

Every request gets exactly one response. The response echoes the request's `id`. Responses carry either:

- A successful payload (e.g., `BufReply` with read data, `WriteReply` with bytes written)
- An error as a POSIX errno value in the `err` field

No response means the request is still in flight. The driver must never silently drop a request.

## Session multiplexing

A single socket connection carries multiple logical sessions, identified by `fh`. Each `open()` from an application creates a new `fh`. The driver should:

- Process requests with the same `fh` sequentially (they represent ordered operations on one file descriptor)
- Process requests with different `fh` values concurrently (they represent independent sessions)

This matches the Linux kernel's concurrency model for character devices.
