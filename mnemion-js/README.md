# Mnemion Server

The Cloudflare Worker: an MCP server providing persistent, evolving shared memory between a human and their AI agents.

**Start at the [root README](../README.md)** for what Mnemion is, the full tool/resource surface, and deployment (one-click or `npm run setup` — plain `wrangler deploy` skips the Vectorize index and `WORKER_HOST` pinning that setup handles). Architecture and design principles live in [`CLAUDE.md`](../CLAUDE.md).

## Local development

```bash
npm install
npm run dev    # wrangler dev (worker + DOs, seeded) + Vite HMR for the React app — open the Vite URL
npm test       # vitest suite (runs in workerd via @cloudflare/vitest-pool-workers)
```

Dev mode auto-approves auth (`DEV=true` is set by the dev scripts only) — no passkey or secret needed locally. MCP endpoint at `http://localhost:8787/mcp`.
