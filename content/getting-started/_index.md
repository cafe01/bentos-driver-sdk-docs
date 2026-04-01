---
title: "Getting Started"
description: "Five minutes to a working device driver -- just copy, paste, run"
weight: 10
toc: true
---

This page gets you to a running driver in five minutes. Copy the code, run it, see it work. For a detailed walkthrough that explains every decision, see [Build Your First Driver](/tutorials/build-your-first-driver/).

## What you'll build

A bidirectional echo device at `/dev/echo`. Write bytes in, read them back. Fifteen lines of driver logic.

```
Shell:
  exec 3<>/dev/echo
  echo hello >&3
  cat <&3              # prints "hello"
  exec 3>&-
```

## Prerequisites

- [Dart SDK](https://dart.dev/get-dart) 3.0 or later
- A BentOS machine (or any Unix for local testing)

> The Dart binding is the reference implementation. Python, Rust, Go, and JavaScript bindings will follow the same patterns and ops contracts described in these docs.

## 1. Create a new project

```bash
mkdir echo-driver && cd echo-driver
dart create -t console .
```

Add the SDK dependency to your `pubspec.yaml`:

```yaml
dependencies:
  bentos_driver_sdk:
    git:
      url: https://github.com/anthropics/bentos-driver-sdk-dart.git
```

Then fetch:

```bash
dart pub get
```

## 2. Write the driver

Replace `bin/echo_driver.dart` with:

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

That's it. Four callbacks, no FUSE vocabulary, no protobuf. Just domain logic.

## 3. Run it

```bash
dart run bin/echo_driver.dart
```

The driver is now listening on a Unix socket. On a BentOS machine, `bentosd` connects to this socket and exposes it as `/dev/echo`. For local testing, you can use the SDK's test utilities or connect directly.

## What just happened

You wrote a device driver. You implemented four domain callbacks (the "ops contract"), and the SDK handled everything else: the wire protocol, session management, poll readiness, error translation.

The SDK has four I/O patterns like this one. Each gives you a different relationship between reads and writes. The echo driver uses Pattern 1 (Pure Stream) -- bidirectional, independent streams.

## Next steps

- **[Build Your First Driver](/tutorials/build-your-first-driver/)** -- a detailed tutorial that explains every line, every design decision, and why you'd choose one pattern over another
- [Choosing a Pattern](/concepts/choosing-a-pattern/) -- which of the four patterns fits your use case
- [Three-Layer Architecture](/concepts/three-layer-architecture/) -- understand the full SDK stack
- [Patterns](/patterns/) -- deep-dives into each I/O pattern
