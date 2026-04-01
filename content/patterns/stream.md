---
title: "P1: Pure Stream"
description: "Bidirectional byte pipe -- read and write are independent, concurrent streams"
weight: 10
toc: true
---

## Overview

The simplest pattern. Read and write are independent streams. No state machine. No causal link between write and read.

**Linux precedent**: Serial ports (`/dev/ttyS*`), PTY

**Framework**: `StreamDriver<S>` / `StreamOps<S>` (where `S` is your per-session state type)

## State machine

None. The session is active from open to release.

```
ACTIVE (from open to release -- no transitions)
```

## Ops contract

| Callback | Required | Signature | Description |
|---|---|---|---|
| `onData` | **yes** | `(bytes, {session}) -> int` | Bytes from `write()`. Return bytes consumed. |
| `outputStream` | no | `({session}) -> Stream<bytes>` | Pull-model output for `read()`. |
| `onSessionStart` | no | `(flags) -> S` | Allocate per-session state. |
| `onSessionEnd` | no | `({session}) -> void` | Clean up per-session state. |

The `session` parameter is whatever `onSessionStart` returned. If you don't provide `onSessionStart`, session is null.

## Poll readiness

| Event | Condition |
|---|---|
| POLLIN | Output stream has bytes available |
| POLLOUT | Always (driver can always accept writes) |

POLLIN and POLLOUT are **independent** -- both may be set simultaneously. This is the defining characteristic of a pure stream.

## Example: `/dev/echo`

Write bytes in, read them back. The complete driver:

```dart
import 'dart:async';
import 'dart:typed_data';

import 'package:bentos_driver_sdk/bentos_driver_sdk.dart';

void main() async {
  final driver = StreamDriver<StreamController<Uint8List>>(StreamOps(
    onSessionStart: (flags) => StreamController<Uint8List>(),
    onData: (data, {required session}) {
      session!.add(Uint8List.fromList(data));
      return data.length;
    },
    outputStream: ({required session}) => session!.stream,
    onSessionEnd: ({required session}) => session!.close(),
  ));

  await driver.serve(Uri.parse('unix:///tmp/bentos-echo.sock'));
  print('Echo driver listening. Ctrl-C to stop.');

  await ProcessSignal.sigint.watch().first;
  await driver.close();
}
```

### How it works

1. **`onSessionStart`**: Each time a process opens the device, the framework calls `onSessionStart`. We create a `StreamController<Uint8List>` -- a simple in-memory buffer that acts as both a sink (for writes) and a stream (for reads).

2. **`onData`**: When the process calls `write()`, the framework calls `onData` with the bytes. We push them into the `StreamController` and report all bytes consumed.

3. **`outputStream`**: When the process calls `read()`, the framework pulls from this stream. Since it's the same `StreamController`, bytes written on one side appear on the other.

4. **`onSessionEnd`**: When the process closes the device, we close the `StreamController`.

### Shell test

```bash
exec 3<>/dev/echo
echo hello >&3
cat <&3              # prints "hello"
exec 3>&-
```

## When to use Pure Stream

- Bidirectional channels (chat, serial communication)
- Devices where read and write are logically independent
- Proxy/passthrough devices
- Any device where "write causes read" is NOT the model

## When NOT to use Pure Stream

- If writes must complete before reads make sense → use [P2: Write-then-Read](/patterns/write-read/)
- If the device only produces events (no write side) → use [P3: Event Stream](/patterns/event-stream/)
- If you need a config phase before processing → use [P4: Configured Stream](/patterns/configured-stream/)
