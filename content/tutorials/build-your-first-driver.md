---
title: "Build Your First Driver"
description: "A detailed walkthrough: from empty directory to working device driver"
weight: 10
toc: true
---

If you followed [Getting Started](/getting-started/), you have a working echo driver. This tutorial goes deeper: we'll build a different driver using a different pattern, and explain every design decision along the way -- why this pattern, why this structure, how the framework helps.

We're building a key-value store device driver. By the end, you'll have a working `/dev/kv` that stores and retrieves values through standard file operations.

We'll use **Pattern 2: Write-then-Read** -- the request/response pattern. Write a query, read the answer.

## What you'll build

```bash
exec 3<>/dev/kv
echo "name=bentos" >&3    # store a value
echo "name" >&3           # query a key
cat <&3                   # prints "bentos"
exec 3>&-
```

A device that speaks a simple protocol:
- Write `key=value` to store
- Write `key` to query
- Read the result

## Step 1: Project setup

Create a new Dart project:

```bash
mkdir kv-driver && cd kv-driver
dart create -t console .
```

Add the SDK to `pubspec.yaml`:

```yaml
dependencies:
  bentos_driver_sdk:
    git:
      url: https://github.com/anthropics/bentos-driver-sdk-dart.git
```

```bash
dart pub get
```

## Step 2: Choose a pattern

Our device has a clear request/response cycle: write a query, read the answer. That's **Pattern 2: Write-then-Read**.

Why not the other patterns?

- **P1 (Pure Stream)**: Read and write would be independent. But our reads depend on what was written -- the response is caused by the request.
- **P3 (Event Stream)**: Read-only. We need writes.
- **P4 (Configured Stream)**: Overkill. No config phase needed, and responses are single values, not streams.

P2 gives us:
- Automatic input accumulation (multiple `write()` calls build up the request)
- State machine enforcement (can't read before writing)
- A single `onRequest` callback -- all the plumbing is handled

## Step 3: Implement the driver

Create `bin/kv_driver.dart`:

```dart
import 'dart:convert';
import 'dart:io';
import 'dart:typed_data';

import 'package:bentos_driver_sdk/bentos_driver_sdk.dart';

void main() async {
  // Shared state: the key-value store
  final store = <String, String>{};

  // Create a WriteReadDriver with our ops contract
  final driver = WriteReadDriver<Object>(WriteReadOps(
    // Allocate per-session state (we just need a non-null marker)
    onSessionStart: (flags) => Object(),

    // THE callback: process one complete request, return one complete response
    onRequest: (input, {required session}) async {
      final text = utf8.decode(input).trim();

      if (text.contains('=')) {
        // Store: "key=value"
        final parts = text.split('=');
        final key = parts[0];
        final value = parts.sublist(1).join('=');
        store[key] = value;
        return Uint8List.fromList(utf8.encode('OK\n'));
      } else {
        // Query: "key"
        final result = store[text] ?? '(not found)';
        return Uint8List.fromList(utf8.encode('$result\n'));
      }
    },
  ));

  // Listen on a Unix socket
  final socketPath = '/tmp/bentos-kv.sock';
  await driver.serve(Uri.parse('unix://$socketPath'));
  print('KV driver listening on $socketPath');

  // Wait for Ctrl-C
  await ProcessSignal.sigint.watch().first;
  await driver.close();

  // Clean up socket file
  final f = File(socketPath);
  if (f.existsSync()) f.deleteSync();
  print('Shutdown complete.');
}
```

Let's walk through each piece:

### The store

```dart
final store = <String, String>{};
```

A simple in-memory map. Shared across all sessions. In a real driver, this might be a database connection, an API client, or any stateful backend.

### The ops contract

```dart
WriteReadOps(
  onSessionStart: (flags) => Object(),
  onRequest: (input, {required session}) async { ... },
)
```

`WriteReadOps` is the P2 ops contract. It has one required callback: `onRequest`.

**`onSessionStart`**: Called when a process opens the device. Returns per-session state. We don't need per-session state for this driver, but the framework needs a non-null marker.

**`onRequest`**: Called when the framework has accumulated a complete request and the process triggers processing (via `flush()` or `read()`). Receives the complete accumulated input as bytes. Returns the complete response as bytes.

That's it. The framework handles:
- Accumulating multiple `write()` calls into one request buffer
- Enforcing the state machine (no reads during IDLE, no writes during PROCESSING)
- Buffering the response for the `read()` path
- Poll readiness (POLLOUT in IDLE/ACCUMULATING, POLLIN in RESPONSE_READY)
- Error translation (your exceptions become POSIX errno values)

### The server

```dart
await driver.serve(Uri.parse('unix://$socketPath'));
```

The driver listens on a Unix socket. On a BentOS machine, `bentosd` connects to this socket when mounting the device. For local development, you can connect directly using the SDK's test client.

## Step 4: Run and test

```bash
dart run bin/kv_driver.dart
```

On a BentOS machine with the driver registered:

```bash
# Store some values
exec 3<>/dev/kv
echo "greeting=hello world" >&3
cat <&3    # "OK"

echo "greeting" >&3
cat <&3    # "hello world"

echo "missing" >&3
cat <&3    # "(not found)"

exec 3>&-
```

## Step 5: Add error handling

The SDK provides `DriverError` subtypes that map to POSIX errno values. Let's add validation:

```dart
onRequest: (input, {required session}) async {
  final text = utf8.decode(input).trim();

  if (text.isEmpty) {
    throw DriverError.invalidArgument('Empty request');
    // Framework returns EINVAL (22) to the caller
  }

  if (text.contains('=')) {
    final parts = text.split('=');
    if (parts[0].isEmpty) {
      throw DriverError.invalidArgument('Empty key');
    }
    store[parts[0]] = parts.sublist(1).join('=');
    return Uint8List.fromList(utf8.encode('OK\n'));
  } else {
    final value = store[text];
    if (value == null) {
      throw DriverError.notFound('Key "$text" not found');
      // Framework returns ENOENT (2) to the caller
    }
    return Uint8List.fromList(utf8.encode('$value\n'));
  }
},
```

The framework catches `DriverError` and translates to the appropriate errno. Unhandled exceptions become EIO (5). See the [Error Handling reference](/reference/error-handling/) for the full mapping.

## What you learned

1. **Pattern selection**: How to match your device's I/O model to the right pattern
2. **Ops contracts**: The minimal callback surface -- you write domain logic, the framework handles FUSE
3. **The state machine is free**: P2's 4-state machine (IDLE/ACCUMULATING/PROCESSING/RESPONSE_READY) is enforced by the framework. You never manage it.
4. **Error handling**: Throw `DriverError` subtypes, get POSIX errno values

## Next steps

- [Pattern deep-dives](/patterns/) -- understand all four patterns in detail
- [Error Handling reference](/reference/error-handling/) -- the full errno mapping
- [P4: Configured Stream](/patterns/configured-stream/) -- the inference pattern, for when you're ready for the complex stuff
