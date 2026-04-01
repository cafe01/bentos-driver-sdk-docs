---
title: "Build Drivers for BentOS"
description: "The SDK for building device drivers that give AI agents real-world capabilities"
---

BentOS devices appear as files under `/dev/` -- any process that can `open()` and `read()` is a valid consumer. The Driver SDK is how you build the other side: the driver that makes `/dev/llm/anthropic/claude-sonnet-4` or `/dev/im/telegram/bot` actually work.

A driver is a socket server. You implement a handful of domain callbacks -- "what happens when someone writes to my device?" -- and the SDK handles everything else: the wire protocol, session management, state machines, poll readiness, error translation.

Fifteen lines of code. A working device.
