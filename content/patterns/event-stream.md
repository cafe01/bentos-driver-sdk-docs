---
title: "P3: Event Stream"
description: "Read-only event broadcast with first-open/last-close activation"
weight: 30
toc: true
---

## Overview

Read-only event broadcast. The driver pushes events. The framework broadcasts them to all listeners. `write()` returns EACCES. First-open activates the driver. Last-close deactivates it.

**Linux precedent**: Input subsystem (`/dev/input/event*`)

**Framework**: `EventStreamDriver<E>` / `EventStreamOps<E>` (where `E` is your event type)

## State machine

None from the driver's perspective. The framework manages per-listener event queues internally.

```
ACTIVE (from open to release -- no transitions)
```

## Ops contract

| Callback | Required | Signature | Description |
|---|---|---|---|
| `encodeEvent` | **yes** | `(E event) -> bytes` | Serialize one event to wire format. |
| `onActivate` | no | `() -> void` | First listener opened -- start producing. |
| `onDeactivate` | no | `() -> void` | Last listener closed -- stop producing. |

### Session semantics: first-open/last-close

P3 follows the Linux input subsystem model:

- `onActivate` fires when the **first** listener opens the device
- `onDeactivate` fires when the **last** listener closes
- The driver doesn't know how many listeners exist -- only whether anyone is listening

Events are broadcast to all listeners' individual queues. Each listener gets its own queue (managed by the framework), but the event source is per-device, not per-fd.

### Emitting events

The driver pushes events by calling `driver.emit(event)`. The framework serializes via `encodeEvent`, queues to all listeners, and handles delivery.

### Important behaviors

- `write()` returns EACCES -- event streams are read-only
- `read()` delivers **complete events, never partial** (whole-message semantics)
- Queue overflow: drop-oldest by default (configurable)

## Poll readiness

| Event | Condition |
|---|---|
| POLLIN | Event queue non-empty for this listener |
| POLLOUT | N/A |

## Example: `/dev/ticker`

A periodic event emitter. Ticks once per second.

```dart
import 'dart:async';
import 'dart:convert';
import 'dart:typed_data';

import 'package:bentos_driver_sdk/bentos_driver_sdk.dart';

void main() async {
  var counter = 0;
  Timer? timer;

  late final EventStreamDriver<int> driver;
  driver = EventStreamDriver<int>(EventStreamOps(
    encodeEvent: (n) => Uint8List.fromList(utf8.encode('tick $n\n')),
    onActivate: () {
      counter = 0;
      timer = Timer.periodic(const Duration(seconds: 1), (_) {
        driver.emit(++counter);
      });
    },
    onDeactivate: () {
      timer?.cancel();
      timer = null;
    },
  ));

  await driver.serve(Uri.parse('unix:///tmp/bentos-ticker.sock'));
  print('Ticker driver listening. Ctrl-C to stop.');

  await ProcessSignal.sigint.watch().first;
  timer?.cancel();
  await driver.close();
}
```

### How it works

1. **No listeners**: The driver is idle. No timer, no events.
2. **First `cat /dev/ticker`**: `onActivate` fires. A timer starts, emitting incremented counter events every second.
3. **Second `cat /dev/ticker`**: Both listeners receive all events (broadcast). `onActivate` does NOT fire again.
4. **First listener closes**: Still one listener. Nothing changes.
5. **Last listener closes**: `onDeactivate` fires. Timer stops. Back to idle.

The driver only manages the event source (the timer). The framework manages per-listener queues, delivery, and the first-open/last-close lifecycle.

### Shell test

```bash
cat /dev/ticker
# tick 1
# tick 2
# tick 3
# ^C
```

## When to use Event Stream

- Sensor data (temperature, accelerometer)
- System notifications
- Log/event tails
- Any device that pushes data to passive readers

## When NOT to use Event Stream

- If the device needs a write side → use [P1: Pure Stream](/patterns/stream/) or [P2: Write-then-Read](/patterns/write-read/)
- If events depend on input/configuration → use [P4: Configured Stream](/patterns/configured-stream/)
