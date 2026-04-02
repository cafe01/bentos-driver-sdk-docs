---
title: "P4: Configured Stream"
description: "Heavy upfront configuration, then streaming output -- the inference pattern"
weight: 40
toc: true
---

## Overview

The most complex pattern. Configure via ioctl or initial writes, submit input, stream output. A 6-state machine manages the full lifecycle including cancellation and reset.

**This is the pattern that LLM inference drivers use.** Set model parameters, write a prompt, stream tokens.

**Linux precedent**: ALSA (`/dev/snd/pcm*`)

**Framework**: `ConfiguredStreamDriver<C, I, O, S>` / `ConfiguredStreamOps<C, I, O, S>`

- `C` = config type
- `I` = decoded input type
- `O` = output chunk type
- `S` = per-session state type

## State machine

```
OPEN ──ioctl/write()──> CONFIGURED ──flush/read()──> PROCESSING
  ^                         |                            |
  |                    (write accumulates)        first output chunk
  |                                                      |
  |                                                      v
  |                                                 STREAMING
  |                                                      |
  |                                              output stream ends
  |                                                      |
  |                                                      v
  |                                                 DRAINING
  |                                             (last chunks buffered)
  |                                                      |
  |                                             all output consumed
  |                                                      |
  |                                                      v
  |                                                 COMPLETE
  |                                                      |
  └──────── RESET / new write() / DROP ──────────────────┘
```

### States

- **OPEN**: No configuration applied. Accepts ioctl config commands. `write()` advances to CONFIGURED with default config.
- **CONFIGURED**: Config applied. Accepts `write()` (accumulates input) and more ioctl config. `flush()` or `read()` triggers processing.
- **PROCESSING**: Input submitted to `process` callback. `write()` returns EBUSY. `read()` blocks.
- **STREAMING**: Output chunks flowing. `read()` returns buffered chunks. `write()` returns EBUSY.
- **DRAINING**: Output stream complete, but buffered chunks remain for reading.
- **COMPLETE**: All output consumed. `read()` returns 0 (EOF). New `write()` restarts the cycle.

### Forced transitions

- **DROP** (ioctl): From any active state to CONFIGURED. Cancels in-flight processing. Calls `onCancel`.
- **RESET** (ioctl): From any state to OPEN. Clears all state including config.

## Ops contract

| Callback | Required | Signature | Description |
|---|---|---|---|
| `defaultConfig` | **yes** | `() -> C` | Default config for new sessions. |
| `process` | **yes** | `(I, C, {session}) -> Stream<O>` | The core operation. Stream must eventually end. |
| `encodeOutput` | no* | `(O, {config}) -> bytes` | Serialize one output chunk for `read()`. |
| `decodeInput` | no* | `(bytes, {config}) -> I` | Deserialize current-cycle write bytes into domain input. |
| `onQuery` | no | `(int cmd, {session}) -> FutureOr<bytes>` | Route a read-direction ioctl to the driver. |
| `onCancel` | no | `({session}) -> void` | Abort in-flight processing (called on DROP). |
| `onDrain` | no | `({session}) -> void` | Graceful completion hook. |
| `onSessionStart` | no | `() -> S` | Allocate per-session state. |
| `onSessionEnd` | no | `({session}) -> void` | Release per-session state. |

\* Required unless the subsystem layer provides a canonical default. If null at call time with no subsystem default, the framework returns ENOSYS.

### `onQuery` — read-direction ioctls

The config codec is write-only: it handles ioctl commands that mutate config. Some ioctls are queries — they read state rather than write it. These are routed to `onQuery`.

```dart
ConfiguredStreamOps(
  // ...
  onQuery: (int cmd, {required session}) async {
    return switch (cmd) {
      LLM_GET_METADATA => encodeMetadata(session.modelInfo),
      LLM_GET_ERROR    => encodeError(session.lastError),
      _ => throw DriverError.invalidArgument('Unknown query $cmd'),
    };
  },
)
```

`onQuery` is called for any ioctl that the config codec does not handle (unknown command falls through). It receives the raw command integer and the current session. Return serialized bytes for the response.

`onQuery` is available in any state — it is not phase-gated like config ioctls. Callers can query metadata while in COMPLETE, or query error details after a failed stream.

### Optional `encodeOutput` / `decodeInput`

Both fields are nullable. If your subsystem defines canonical serialization (the subsystem author provides defaults), you can omit them:

```dart
// Subsystem provides encodeOutput and decodeInput defaults.
// Driver only needs to supply process logic.
ConfiguredStreamOps(
  defaultConfig: MyConfig.new,
  process: (input, config, {required session}) async* {
    yield* callProvider(input, config);
  },
  onSessionStart: MySession.new,
)
```

When a driver omits `encodeOutput` or `decodeInput`, the subsystem layer fills in the default. If no subsystem default exists, the framework returns ENOSYS with a message directing the subsystem to provide one.

### `decodeInput` and multi-turn write semantics

`decodeInput` receives the accumulated write buffer for the **current cycle only**. It does not receive history from previous write/read cycles on the same fd.

A cycle is one pass through the state machine: CONFIGURED → (writes accumulate) → PROCESSING → STREAMING → COMPLETE. When the caller writes again after COMPLETE, a new cycle begins with a fresh buffer.

**Cross-cycle state is caller-managed.** If the application needs multi-turn context (e.g., conversation history), the caller assembles and injects that context into each write. The driver does not accumulate history.

The session object (from `onSessionStart`) is the driver's only cross-cycle state — appropriate for things that genuinely belong to the session (authentication, active model handle), not for conversation history.

```
Cycle 1: write("hello") → read() → "olleh"
Cycle 2: write("world") → read() → "dlrow"
          ↑
          decodeInput sees only "world", not "hello world"
          Caller is responsible for including prior context if needed
```

### The config codec

P4 requires a `ConfigCodec<C>` that maps raw ioctl commands to typed config mutations. The driver never sees raw ioctl bytes -- the codec translates.

```dart
class MyConfigCodec extends ConfigCodec<MyConfig> {
  @override
  MyConfig apply(MyConfig current, int command, Uint8List data) {
    return switch (command) {
      0x01 => current.copyWith(param1: decodeParam1(data)),
      0x02 => current.copyWith(param2: decodeParam2(data)),
      _ => throw DriverError.invalidArgument('Unknown config $command'),
    };
  }

  @override
  Uint8List encode(MyConfig config, int command) => Uint8List(0);
}
```

### Framework ioctls

The framework reserves ioctl type byte `0xBE` for built-in control commands:

| Command | Description |
|---|---|
| `GET_STATE` | Query current state machine phase |
| `DRAIN` | Acknowledge stream completion |
| `DROP` | Cancel in-flight processing, back to CONFIGURED |
| `RESET` | Clear everything, back to OPEN |
| `GET_ERROR` | Query stream error (in COMPLETE state) |

## Poll readiness

| State | POLLIN | POLLOUT |
|---|---|---|
| OPEN | no | yes |
| CONFIGURED | no | yes |
| PROCESSING | no | no |
| STREAMING | yes (when chunks buffered) | no |
| DRAINING | yes (remaining data) | no |
| COMPLETE | no | no |

Poll readiness is **fully automatic** -- computed by the framework from the state machine. The driver never implements poll.

## Example: `/dev/synth`

A text transformation device. Optionally uppercases, then reverses text, streaming one character at a time with a configurable delay.

```dart
import 'dart:async';
import 'dart:convert';
import 'dart:typed_data';

import 'package:bentos_driver_sdk/bentos_driver_sdk.dart';

/// Subsystem config
final class SynthConfig {
  final bool uppercase;
  final int delayMs;
  const SynthConfig({this.uppercase = false, this.delayMs = 100});
  SynthConfig copyWith({bool? uppercase, int? delayMs}) =>
      SynthConfig(
        uppercase: uppercase ?? this.uppercase,
        delayMs: delayMs ?? this.delayMs,
      );
}

/// Config codec -- maps ioctl commands to config mutations
final class SynthConfigCodec extends ConfigCodec<SynthConfig> {
  static const setUppercase = 0x01;
  static const setDelay = 0x02;

  @override
  SynthConfig apply(SynthConfig current, int command, Uint8List data) {
    return switch (command) {
      setUppercase => current.copyWith(uppercase: data[0] != 0),
      setDelay => current.copyWith(
        delayMs: ByteData.sublistView(data).getInt32(0, Endian.little),
      ),
      _ => throw DriverError.invalidArgument('Unknown config $command'),
    };
  }

  @override
  Uint8List encode(SynthConfig config, int command) => Uint8List(0);
}

void main() async {
  final driver = ConfiguredStreamDriver<SynthConfig, String, String, Object>(
    ConfiguredStreamOps(
      defaultConfig: SynthConfig.new,
      process: (input, config, {required session}) async* {
        final text = config.uppercase ? input.toUpperCase() : input;
        for (final char in text.trim().split('').reversed) {
          await Future<void>.delayed(
            Duration(milliseconds: config.delayMs),
          );
          yield char;
        }
      },
      encodeOutput: (chunk, {required config}) =>
          Uint8List.fromList(utf8.encode(chunk)),
      decodeInput: (data, {required config}) => utf8.decode(data),
      onSessionStart: Object.new,
    ),
    configCodec: SynthConfigCodec(),
  );

  await driver.serve(Uri.parse('unix:///tmp/bentos-synth.sock'));
  print('Synth driver listening. Ctrl-C to stop.');

  await ProcessSignal.sigint.watch().first;
  await driver.close();
}
```

### How it works

1. **OPEN**: Device opened. Default config: `uppercase: false, delayMs: 100`.
2. **CONFIGURED**: Optionally, ioctl commands adjust config (uppercase, delay).
3. **`write("hello")`**: Input accumulated. State stays CONFIGURED (or advances from OPEN).
4. **`read()` or `flush()`**: Triggers processing. `decodeInput` converts bytes to String. `process` receives the string and config.
5. **STREAMING**: `process` yields characters one at a time. Each is serialized via `encodeOutput` and buffered for `read()`.
6. **DRAINING**: The `process` stream ends. Remaining buffered chunks are consumed.
7. **COMPLETE**: All output read. `read()` returns 0 (EOF). New `write()` restarts the cycle.

### Shell test

```bash
exec 3<>/dev/synth
echo "hello" >&3
cat <&3              # prints "o l l e h" (streamed)
exec 3>&-
```

## The inference pattern

LLM inference drivers use P4:

- **Config phase**: Model parameters (temperature, max tokens, system prompt) via ioctl
- **Input phase**: The user prompt via `write()`
- **Processing**: Prompt submitted to the LLM provider
- **Streaming**: Tokens streamed back via `read()`
- **Cancellation**: DROP cancels generation mid-stream

The config codec maps ioctl commands to inference parameters. `process` calls the LLM API and yields tokens. `encodeOutput` serializes tokens for the read path. The entire FUSE lifecycle -- state machine, poll, buffering, cancellation -- is handled by the framework.

## When to use Configured Stream

- LLM inference
- Audio/video processing
- Any device with upfront configuration and streaming output
- Heavy computation where input is submitted and output streams back

## When NOT to use Configured Stream

- If you don't need a config phase → simpler patterns suffice
- If output is a single response (not a stream) → use [P2: Write-then-Read](/patterns/write-read/)
- If the device only produces events → use [P3: Event Stream](/patterns/event-stream/)
