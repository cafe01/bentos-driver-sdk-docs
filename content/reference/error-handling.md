---
title: "Error Handling"
description: "DriverError subtypes, POSIX errno mapping, and error behavior per pattern"
weight: 10
toc: true
---

## How errors work

Drivers signal errors by throwing `DriverError` subtypes from ops callbacks. The framework translates them to POSIX errno values in the `FuseResponse.err` field. Zero means success. Any non-zero value is an errno.

```dart
// In an ops callback:
throw DriverError.notFound('key "$key" does not exist');
// Framework returns FuseResponse(err: 2) -- ENOENT
```

Unhandled exceptions (anything that isn't a `DriverError`) are caught by the framework and returned as EIO (5).

## Error mapping

| DriverError subtype | errno | Value | When to use |
|---|---|---|---|
| `invalidArgument` | EINVAL | 22 | Bad input, malformed data, unknown config command |
| `busy` | EBUSY | 16 | Resource in use |
| `notFound` | ENOENT | 2 | Key not found, model not available |
| `notPermitted` | EPERM | 1 | Auth failure, quota exceeded |
| `resourceExhausted` | ENOMEM | 12 | Token limit, memory limit |
| `timedOut` | ETIMEDOUT | 110 | Provider timeout, inference timeout |
| `ioError` | EIO | 5 | Provider failure, network error |
| `notSupported` | ENOSYS | 38 | Operation not implemented |
| `accessDenied` | EACCES | 13 | Write to read-only device |
| `wouldBlock` | EAGAIN | 11 | No data ready (non-blocking mode) |

## Framework-generated errors

These errors are returned by the framework automatically -- the driver doesn't throw them:

| Condition | errno | Patterns |
|---|---|---|
| `write()` during PROCESSING | EBUSY (16) | P2, P4 |
| `write()` during RESPONSE_READY | EBUSY (16) | P2 |
| `read()` during IDLE | EAGAIN (11) | P2 |
| `write()` on read-only device | EACCES (13) | P3 |
| Unknown ioctl command | EINVAL (22) | P4 |
| Unhandled callback exception | EIO (5) | All |

## Usage patterns

### Simple validation

```dart
onRequest: (input, {required session}) async {
  final text = utf8.decode(input).trim();
  if (text.isEmpty) {
    throw DriverError.invalidArgument('Empty request');
  }
  // ... process
}
```

### Resource errors

```dart
process: (input, config, {required session}) async* {
  final response = await callProvider(input, config);
  if (response.statusCode == 429) {
    throw DriverError.resourceExhausted('Rate limit exceeded');
  }
  if (response.statusCode == 408) {
    throw DriverError.timedOut('Provider timeout');
  }
  // ... yield tokens
}
```

### Not found

```dart
onRequest: (input, {required session}) async {
  final value = store[key];
  if (value == null) {
    throw DriverError.notFound('Key "$key" does not exist');
  }
  return Uint8List.fromList(utf8.encode('$value\n'));
}
```

## Error behavior per pattern

### P1: Pure Stream
- Errors in `onData` are returned as the `write()` errno
- Errors in `outputStream` cause `read()` to return EIO
- Session continues after errors (streams are resilient)

### P2: Write-then-Read
- Errors in `onRequest` are returned as the `read()` errno
- State machine resets to IDLE after an error (ready for next request)

### P3: Event Stream
- Errors in `encodeEvent` cause the event to be dropped for that listener
- `onActivate`/`onDeactivate` errors are logged but don't affect listeners

### P4: Configured Stream
- Errors in `process` terminate the output stream and enter COMPLETE with error
- The error is queryable via `GET_ERROR` ioctl
- `read()` returns 0 (EOF) -- the caller must check `GET_ERROR` for failure details
- Errors in `decodeInput` are returned as the `write()` errno
