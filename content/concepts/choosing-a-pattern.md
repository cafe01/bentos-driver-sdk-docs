---
title: "Choosing a Pattern"
description: "Which I/O pattern fits your driver? A decision guide with real-world reasoning."
weight: 30
toc: true
---

The Driver SDK provides four I/O patterns. Each models a different relationship between reads and writes. Choosing the right one is the most important design decision for your driver.

## The quick answer

| You need... | Pattern | Framework |
|---|---|---|
| Bidirectional byte pipe | P1: Pure Stream | `StreamDriver` |
| Request in, response out | P2: Write-then-Read | `WriteReadDriver` |
| Push events to readers | P3: Event Stream | `EventStreamDriver` |
| Config phase + streaming output | P4: Configured Stream | `ConfiguredStreamDriver` |
| Full FUSE control / diagnostics | L1: Raw | `BentosDriver` |

If that table answered your question, go build. If not, read on.

## The key question: how do read and write relate?

The four patterns exist because there are four fundamentally different relationships between the read and write sides of a device:

### P1: Read and write are independent

The write side and read side are separate streams. Writing does not cause reading. Reading does not depend on writing. Both can happen concurrently.

**Think**: serial port, PTY, chat connection, bidirectional pipe.

**Poll behavior**: POLLIN and POLLOUT can both be set simultaneously. They're independent.

**Choose P1 when**: your device is a bidirectional channel where each direction operates independently.

### P2: Write causes read

Writing submits a request. Reading retrieves the response. You must write before you can read. The causal link between write and read is the defining characteristic.

**Think**: HTTP request/response (but as a device), database query, RPC call.

**Poll behavior**: POLLIN and POLLOUT are **mutually exclusive**. When you can write (accepting a request), you can't read (no response yet). When you can read (response ready), you can't write (still delivering the response). This mutual exclusivity IS the pattern.

**Choose P2 when**: your device processes a complete request and produces a complete response.

### P3: Only read (events pushed to you)

The device produces events. Readers consume them. Writing is not supported (returns EACCES). The driver pushes events; the framework broadcasts them to all listeners.

**Think**: input device (`/dev/input/event*`), sensor, notification stream, log tail.

**Poll behavior**: POLLIN when events are queued. POLLOUT is not applicable.

**Choose P3 when**: your device is a read-only event source with potentially multiple listeners.

### P4: Configure, then stream

Heavy upfront configuration (via ioctl or initial writes), then submit input, then stream output. A 6-state machine manages the lifecycle. This is the most complex pattern -- and the one that inference drivers use.

**Think**: ALSA audio (`/dev/snd/pcm*`), video encoder, **LLM inference**. Set parameters, submit input, stream output tokens.

**Poll behavior**: Changes with state. POLLOUT during config/input. POLLIN during streaming output. Mutually exclusive across phases.

**Choose P4 when**: your device needs configuration before processing, and produces streaming output from submitted input.

## Real-world examples

| Device | Pattern | Why |
|---|---|---|
| `/dev/echo` | P1 | Write bytes in, read them back -- independent streams |
| `/dev/kv` | P2 | Write a key query, read the value -- request/response |
| `/dev/ticker` | P3 | Emits periodic events -- read-only event source |
| `/dev/synth` | P4 | Configure transformation, write input, stream output |
| `/dev/llm/anthropic/claude-sonnet-4` | P4 | Configure model params, write prompt, stream tokens |
| `/dev/im/telegram/bot` | P1 | Send messages (write) and receive messages (read) independently |
| `/dev/input/keyboard` | P3 | Keyboard emits key events -- read-only |

## When nothing fits: Layer 1

If your device genuinely doesn't match any pattern, use `BentosDriver` directly (Layer 1). You get raw FUSE op callbacks with full control. But this means you handle session management, poll readiness, state machines, and error translation yourself.

The `playground_driver` example is a good starting point -- it's an instrumented workbench that logs every FUSE op.

## Complexity comparison

| Pattern | Ops callbacks | State machine | Typical driver LOC |
|---|---|---|---|
| P1: Pure Stream | 2-4 | None | ~15 |
| P2: Write-then-Read | 1-3 | 4 states (framework-managed) | ~20 |
| P3: Event Stream | 1-3 | None (framework tracks listeners) | ~15 |
| P4: Configured Stream | 4-8 | 6 states (framework-managed) | ~45 |
| L1: Raw | 1-8 | You manage it | varies |

The pattern frameworks earn their complexity budget. P4 manages a 6-state machine, buffering, ioctl routing, and cancellation -- so you don't have to.
