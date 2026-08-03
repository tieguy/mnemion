# Mnemion

Cloudflare Worker entry: an OAuth-wrapped MCP server whose one declarative route table is the whole HTTP surface.

## zones
- owner-trusted: Full kernel access — the ownerDataCtx capability (the only trusted:true reader/writer).
- storage: The SQLite / R2 substrate beneath every chart.
- served-untrusted: Public reads + ingress/upload writes — the servedDataCtx capability (trusted:false).
- public-egress: Served HTTP responses (/o /p /f) and the /ws broadcast — leaves the DO in the clear.
- agent-mcp: The MCP tool surface — agent input, consent-gated, never trusted with the kernel.
- federated: Cross-hive resolve over the network — a sovereign foreign hive.

## works when
- src/index.ts exists at root
- wrangler.toml exists at root
- README.md exists at root
- src/index.ts imports @cloudflare/workers-oauth-provider

## why

The worker entry keeps the entire HTTP surface as one scannable declarative route table (method, pattern, auth gate, handler per line) so the system's shape is graspable from the declarations alone, per the "code as schematic" principle. OAuthProvider wraps the worker to own the OAuth 2.1 / DCR / token flow and intercept `/mcp`, `/token`, `/register` before dispatch, so the rest of the code never re-implements auth plumbing.
