---
title: "P2: Write-then-Read"
description: "Request/response -- write submits a request, read retrieves the response"
weight: 20
toc: true
---

## Overview

Request/response pattern. Write accumulates a request. Flush or read triggers processing. Read delivers the response. The causal link between write and read defines the pattern.

**Linux precedent**: Custom protocol devices, PPP

**Framework**: `WriteReadDriver<S>` / `WriteReadOps<S>`

## State machine

```
IDLE ──write()──> ACCUMULATING ──flush/read()──> PROCESSING ──done──> RESPONSE_READY ──read()──> IDLE
                  (more writes append)
```

- **IDLE**: Ready for a new request. `write()` accepted, `read()` returns EAGAIN.
- **ACCUMULATING**: Bytes written but not yet submitted. More `write()` calls append.
- **PROCESSING**: Request submitted to `onRequest`, awaiting response. `write()` returns EBUSY. `read()` blocks (or EAGAIN in non-blocking mode).
- **RESPONSE_READY**: Response available. `read()` returns response bytes. `write()` returns EBUSY until response fully consumed.

The framework enforces all transitions. Invalid operations return the appropriate errno.

## Ops contract

| Callback | Required | Signature | Description |
|---|---|---|---|
| `onRequest` | **yes** | `(bytes, {session}) -> Future<bytes>` | Process complete request, return complete response. |
| `onSessionStart` | no | `(flags) -> S` | Allocate per-session state. |
| `onSessionEnd` | no | `({session}) -> void` | Clean up per-session state. |

One callback. The framework handles accumulation, state machine enforcement, and buffering.

## Poll readiness

| State | POLLIN | POLLOUT |
|---|---|---|
| IDLE | no | **yes** |
| ACCUMULATING | no | **yes** |
| PROCESSING | no | no |
| RESPONSE_READY | **yes** | no |

**POLLIN and POLLOUT are mutually exclusive.** This is the causal link between write and read manifesting in poll readiness. When the device can accept writes, it has no response to read. When a response is ready, it won't accept new writes until the response is consumed.

## Example: `/dev/kv`

A key-value store. Write `key=value` to store, write `key` to query, read the result.

```dart
import 'dart:convert';
import 'dart:typed_data';

import 'package:bentos_driver_sdk/bentos_driver_sdk.dart';

void main() async {
  final store = <String, String>{};

  final driver = WriteReadDriver<Object>(WriteReadOps(
    onSessionStart: (flags) => Object(),
    onRequest: (input, {required session}) async {
      final text = utf8.decode(input).trim();
      if (text.contains('=')) {
        final parts = text.split('=');
        store[parts[0]] = parts.sublist(1).join('=');
        return Uint8List.fromList(utf8.encode('OK\n'));
      } else {
        return Uint8List.fromList(
          utf8.encode('${store[text] ?? "(not found)"}\n'),
        );
      }
    },
  ));

  await driver.serve(Uri.parse('unix:///tmp/bentos-kv.sock'));
  print('KV driver listening. Ctrl-C to stop.');

  await ProcessSignal.sigint.watch().first;
  await driver.close();
}
```

### How it works

1. The process writes `"name=bentos\n"` -- the framework accumulates it
2. The process calls `read()` (or `flush()` then `read()`) -- this triggers the state transition to PROCESSING
3. The framework passes the complete accumulated input to `onRequest`
4. `onRequest` parses the input, stores the value, returns `"OK\n"`
5. The framework buffers the response and enters RESPONSE_READY
6. `read()` returns `"OK\n"`
7. Back to IDLE, ready for the next request

The driver only implements `onRequest`. The entire accumulation-submission-response cycle is framework-managed.

### Shell test

```bash
exec 3<>/dev/kv
echo "name=bentos" >&3    # store
echo "name" >&3           # query
cat <&3                   # prints "bentos"
exec 3>&-
```

## When to use Write-then-Read

- Any request/response device
- Database query interfaces
- RPC-style devices
- Lookup services
- Any device where "write a question, read the answer" is the model

## When NOT to use Write-then-Read

- If read and write are independent → use [P1: Pure Stream](/patterns/stream/)
- If the device only produces events → use [P3: Event Stream](/patterns/event-stream/)
- If you need configuration before processing, or streaming output → use [P4: Configured Stream](/patterns/configured-stream/)
