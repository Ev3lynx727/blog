---
title: "Oh My MCP PART 1: The Day the VM Crawled"
description: How 45 MCP server processes, a six-fold increase in SDK dependencies, and one convenience flag transformed a lean development VM into a sluggish mess.
tags: [mcp, architecture, performance, ai]
---
The VM felt slow.

Not crash-and-burn slow. Not out-of-disk-space slow. Just... sluggish. The kind where you type a character and see it on screen half a heartbeat later. The kind where your editor stutters when you switch tabs. No single crash — just a persistent, nagging feeling that something was wrong.

I opened a terminal and typed the oldest diagnostic in the book:

```bash
$ ps aux | wc -l
```

The number came back high. I checked again:

```
load average: 6.97, 11.28, 9.86
```

My VM has eight cores. A load of seven means the real CPU is oversubscribed. But I wasn't running anything. Or so I thought.

## The 45 Guests at the Party

```bash
$ ps aux | grep -c node
43
```

Forty-three Node.js processes. Just sitting there. Most of them doing nothing. Each one eating 70, 80, 100 megabytes of memory — just to exist. Like guests at a party who showed up, grabbed a drink, and never left.

But who invited them?

I opened the config file and found the answer: **MCP servers**. Every tool my AI agent could use — reading files, searching code, running commands — was backed by a separate server process. And somewhere along the way, every server was configured with the equivalent of:

```
"run this, I don't care which version, just get the newest one"
```

## Why Does My File Reader Need a Web Framework?

I checked one server. Just one. The filesystem tool that reads and writes files.

```bash
$ du -sh node_modules/@modelcontextprotocol/sdk/
4.2M    677 files
```

Four megabytes. Six hundred seventy-seven files. For a protocol that's supposed to be simple stdio communication.

I checked the dependency tree and found **Express** — a full web framework. **Hono** — another web framework. **ajv** — a JSON Schema validator. **jose** — a JWT library for OAuth. Sitting inside a process that does nothing but read files and talk over stdin.

*Why does my file reader need a web framework?*

That question became the investigation.

## The Breadcrumbs

I traced the SDK version history. What I found was a quiet bloat timeline:

- **v0.4.0 (Nov 2024):** 71 files, 3 dependencies. Clean and minimal.
- **v1.21.0 (Mar 2025):** Express, ajv, cors moved from devDependencies to runtime — 13 deps. First bloat point.
- **v1.24.0 (Mar 2025):** `jose` added for OAuth — 14 deps. Second bloat point.
- **v1.25.0 (Apr 2025):** `hono` + `@hono/node-server` added — 17 deps, 677 files, 4.2MB. The SDK now ships **two competing web frameworks** as required dependencies.
- **v1.29.0 (current):** 677 files. A 10x growth from where it started.

Every time I reloaded my environment, `npx -y` pulled `latest` from the npm registry. And "latest" silently upgraded from a 71-file minimal SDK to a 677-file kitchen sink. I never noticed because no version was pinned.

## The Six Root Causes

The investigation uncovered a layered cake of inefficiencies:

**1. npx -y overhead.** Every server ran through a three-process chain: `sh → npx → node`. Two extra processes per server doing zero real work. Six enabled servers = **12 dead wrapper processes**.

**2. SDK bloat.** A stdio-only protocol shipped two web frameworks as required dependencies. Every server paid the import cost for Express, Hono, ajv, and jose — none of which it used.

**3. No shared dependencies.** Each `npx -y` loaded the SDK independently — 4.2MB, 17 deps, repeated six times into six separate V8 isolates. No shared memory, no shared `node_modules`.

**4. Silent supply chain drift.** No version pinning meant every reload could silently upgrade the SDK. The drift from v0.4.0 to v1.29.0 happened over weeks, unnoticed — until the VM crawled.

**5. JSON serialization tax.** The filesystem server wrapped every file operation in JSON-RPC framing. Special characters (backticks, quotes) broke the envelope, causing retries. Estimated waste: 180,000 tokens per day — roughly $15-45/month burned on retrying writes.

**6. The meta pattern.** Every root cause is a wrapper. `npx -y` wraps the server in a shell process. The SDK wraps stdio in HTTP frameworks. JSON-RPC wraps file writes in serialization framing. **Every wrapper is a failure mode.**

## What Survived

The investigation didn't stop at diagnosis. It produced a **local-first architecture** that removes every unnecessary layer:

- Direct `node` paths instead of `npx -y`
- SDK pinned to a lean version (or forked to stdio-only)
- Single shared `node_modules`
- Raw stdio transport instead of JSON-RPC framing

The principles are captured as a governance skill — a living document that enforces monthly health checks, version audits, and process count baselines. The fix is repeatable, not one-time.

And the SDK itself? The right abstraction is transport-split packages: core protocol types in one package, HTTP transport in another, OAuth helpers in a third. Consumers pay only for what they use.

## The One Sentence

If you take one thing from this investigation, let it be this:

**Every wrapper is a failure mode.** Fewer layers, fewer failures, leaner system.

Once you see it, you'll see it everywhere — in your tools, your dependencies, your configs, your code. The moment you ask "why does this tool need another tool to run?" is the moment you start building better systems.

---

*This is PART 1 of the Oh My MCP trilogy. PART 2 covers the migration from npx-based to local-first architecture. PART 3 covers governance, security, and building lean servers.*