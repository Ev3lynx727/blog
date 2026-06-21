---
title: Oh My MCP PART 2: Migration Day
description: How we replaced 12 wrapper processes, 510MB of RAM overhead, and a silent supply chain drift with local installs, pinned versions, and one shared node_modules.
tags: [mcp, devops, architecture, performance]
---
I opened the config file. Eleven entries. Six running. Zero pinned versions.

Every server was configured with the same convenient spell: `npx -y`. The "-y" flag that says "yes, I trust you, just run the latest." The flag that bypasses local resolution and pulls from the npm registry every single time.

I had been paying a convenience tax I didn't even know existed.

## The Three-Process Chain

Here's what `npx -y @modelcontextprotocol/server-everything` actually does:

```
sh -c "npx @pkg"          → shell wrapper, ~1.5ms spawn cost
↓
npm exec resolves pkg     → reads registry metadata, ~85MB RSS
↓
node ./real-server.js     → the actual server, ~85MB RSS
```

Three processes. Two of them do zero real work. Six enabled servers = **12 dead processes** burning CPU scheduler time and memory.

Node.js spawn throughput is ~651 requests per second (on a Hetzner CCX33). For comparison, Go does 5,227. Every `spawn()` blocks the main thread. When you have 12 wrapper processes per session, that's 12 main-thread blocks before the actual server even starts.

And `-y` makes it worse: npm docs confirm that with `-y`, npx "will always use a freshly-installed, temporary version" — even if the package is already cached locally. Every restart triggers a registry resolve.

## The Fix Is Boring

The fix isn't clever. It's the opposite of clever:

```bash
npm init -y
npm install @modelcontextprotocol/server-everything@1.0.4 --save-exact
npm install @modelcontextprotocol/server-sequential-thinking@1.0.0 --save-exact
```

Then reference by direct path:

```jsonc
{
  "command": ["node", "./node_modules/@modelcontextprotocol/server-everything/dist/index.js"]
}
```

No shell wrapper. No npm resolver. One process. Zero network calls.

Or use `npx` without the `-y` flag — which checks `$PATH` and `node_modules/.bin` first. If found locally, it executes directly: zero network, zero extra spawns. The `-y` flag is the expensive one. Without it, npx is thin.

## The Hidden Cost of "Latest"

Every server was pulling `@modelcontextprotocol/sdk` at whatever `latest` resolved to. And `latest` silently drifted from v0.4.0 (71 files, 3 deps) to v1.29.0 (677 files, 17 deps) — a 10x increase in file count.

Nobody noticed because no version was pinned. There was no `package.json`, no lockfile, no audit trail. The SDK grew three times its original size while the servers kept saying "yes, just run the latest."

The fix is equally boring:

```bash
npm install @modelcontextprotocol/sdk@0.4.0 --save-exact
```

This pins the SDK to the last version before the bloat points. For stdio-only servers, SDK v0.4.0 has everything you need — Zod for schema validation, raw-body and content-type for message framing. No Express, no Hono, no ajv, no jose.

## What Changed

| Before | After |
|--------|-------|
| 12 wrapper processes | 0 |
| 6 npx -y network calls per restart | 0 |
| SDK v1.29.0 (17 deps, 677 files) | SDK v0.4.0 (3 deps, 71 files) |
| No version pinning | All versions pinned with --save-exact |
| ~510MB RAM for server processes | ~200MB |
| ~100MB stale npx caches | 0 |
| "It worked yesterday" drift | Deterministic, reproducible |

## The Discipline

The migration took an hour. The discipline lasts forever.

Every new server goes through the same workflow:
1. Install with `--save-exact`
2. Reference by direct `node` path
3. Commit the lockfile
4. Never use `-y` in production configs

Monthly health checks catch drift: `npm outdated`, process count baseline, memory snapshot. If a server needs an update, you bump the version explicitly — the `latest` tag has no place in a local-first architecture.

## Migration Day

That's it. One afternoon of replacing convenience shortcuts with deliberate choices. The VM breathes again. The load average dropped from 6.97 to under 2.0. The editor doesn't stutter.

The fix was never about the code. It was about the config.

---

*This is PART 2 of the Oh My MCP trilogy. PART 1 covered the discovery of 45 server processes and SDK bloat. PART 3 covers governance, security, and building lean servers.*