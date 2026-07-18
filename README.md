# perplexity-mcp-slim — EngageWisdom fork

A trimmed fork of the [official Perplexity MCP server](https://github.com/perplexityai/modelcontextprotocol) (`@perplexity-ai/mcp-server`) that advertises only two tools:

- `perplexity_search` — fast web search with citations
- `perplexity_research` — deep multi-source investigation

The upstream `perplexity_ask` and `perplexity_reason` tools are **removed from the MCP tool list entirely**, so agents can't discover or reach for them. Removing tools at the source is the only filter that works uniformly across Cursor, Claude Code, Cline, and Codex. Behavioral rules ("do not use") drift; per-agent permission filters need four separate configs; skills don't hide tools.

Everything else in this fork is byte-identical to upstream. Same protocol, same HTTP transport, same tests, same license (MIT).

---

## Install (once per machine)

```bash
git clone https://github.com/EngageWisdom/perplexity-mcp-slim.git ~/perplexity-mcp-slim
cd ~/perplexity-mcp-slim && npm install
```

`npm install` runs the `prepare` script which builds `dist/`. Verify `~/perplexity-mcp-slim/dist/index.js` exists.

Template 10's `verify-template_10-*.py --yes` scripts will do this for you automatically on any machine where the fork is missing, and will also switch any agent MCP config still pointing at the upstream npm package to point at this fork instead.

---

## Configure your MCP client

Point each agent's MCP config at the built entry file. Absolute path required — MCP clients do not expand `~`:

```json
"perplexity": {
  "command": "node",
  "args": ["/Users/YOURNAME/perplexity-mcp-slim/dist/index.js"],
  "env": {"PERPLEXITY_API_KEY": "pplx-..."}
}
```

For Codex (TOML):

```toml
[mcp_servers.perplexity]
command = "node"
args = ["/Users/YOURNAME/perplexity-mcp-slim/dist/index.js"]
env = {PERPLEXITY_API_KEY = "pplx-..."}
```

The exact paths for each agent's config file are in [`template_10/mcp_setup/README.md`](https://github.com/EngageWisdom/) (EW-internal — same file also covers the auto-switch behavior).

---

## What differs from upstream

Only `src/server.ts` is modified. The changes are:

1. **Removed** the entire `server.registerTool("perplexity_ask", ...)` block (~40 lines).
2. **Removed** the entire `server.registerTool("perplexity_reason", ...)` block (~40 lines).
3. **Rewrote** the server-info `instructions` blob near the top of `createPerplexityServer(...)` to describe only the two remaining tools.
4. **Rewrote** the cross-reference lines in the descriptions of `perplexity_search` and `perplexity_research` (they used to say "use `perplexity_ask` instead" — now they point at each other).

That's it. All other server code, tools, HTTP handling, validation, and tests are untouched.

Compare against upstream directly:

```bash
git diff upstream/main..main -- src/server.ts
```

---

## Update from upstream

Upstream (`perplexityai/modelcontextprotocol`) releases new versions periodically. To pull an update:

### 1. Fetch and merge upstream

```bash
cd ~/perplexity-mcp-slim
git fetch upstream
git merge upstream/main
```

**Expected outcome:** a merge conflict inside `src/server.ts`. The conflict markers will land almost entirely on the two tool blocks that this fork removes and on the descriptions this fork rewrote. Nothing else should conflict in a routine upstream release.

### 2. Re-apply the fork's removals

Open `src/server.ts` and, for each conflict block:

- **If the conflict is inside a `server.registerTool("perplexity_ask", ...)` or `server.registerTool("perplexity_reason", ...)` block** — take **"ours" (delete the block entirely)**. Both `<<<<<<< HEAD` and `=======` sides are gone; the entire `server.registerTool(...);` call is removed.
- **If the conflict is inside the `instructions:` string at the top of `createPerplexityServer(...)`** — keep "ours" version (the shorter one that mentions only `perplexity_search` and `perplexity_research`).
- **If the conflict is inside the `description:` field of `perplexity_search` or `perplexity_research` registerTool blocks** — keep "ours" version (the one whose cross-reference points at the other allowed tool, not at `perplexity_ask` or `perplexity_reason`).
- **Any other conflict** — read the upstream change carefully. It is probably a genuine improvement to keep. Take "theirs" unless it reintroduces a removed tool.

After resolving, confirm no `perplexity_ask` or `perplexity_reason` references remain:

```bash
grep -nE 'perplexity_(ask|reason)' src/server.ts
```

Expected output: **zero lines**.

### 3. Rebuild and verify

```bash
npm install    # runs prepare script → tsc → chmod dist/*.js
```

Confirm the compiled dist advertises exactly the two tools:

```bash
PERPLEXITY_API_KEY=dummy node -e "
const { spawn } = require('child_process');
const p = spawn('node', ['dist/index.js'], { stdio: ['pipe', 'pipe', 'inherit'] });
let out = '';
p.stdout.on('data', d => out += d.toString());
const send = o => p.stdin.write(JSON.stringify(o) + '\n');
send({jsonrpc:'2.0',id:1,method:'initialize',params:{protocolVersion:'2024-11-05',capabilities:{},clientInfo:{name:'t',version:'1'}}});
send({jsonrpc:'2.0',method:'notifications/initialized'});
send({jsonrpc:'2.0',id:2,method:'tools/list',params:{}});
setTimeout(() => {
  for (const l of out.split('\n').filter(Boolean)) {
    try { const m = JSON.parse(l); if (m.id===2) console.log('TOOLS:', m.result.tools.map(t=>t.name).join(', ')); } catch(e) {}
  }
  p.kill(); process.exit(0);
}, 1500);
"
```

**Expected output:** `TOOLS: perplexity_research, perplexity_search` (exactly two, in either order).

If more than two tools appear, the removal wasn't complete — re-check step 2 and re-run step 3.

### 4. Commit and push

```bash
git add src/server.ts
git commit -m "Re-remove perplexity_ask and perplexity_reason after upstream merge to <upstream-tag>"
git push origin main
```

### 5. Restart your MCP agents

Each agent (Cursor, Claude Code, Cline, Codex) needs a full quit + relaunch to reload the MCP server. In each agent, verify **Settings → MCP** (or equivalent) shows the `perplexity` server with exactly two tools.

**Estimated total time per upstream release: 5–10 minutes.**

---

## Upstream remote setup

If you cloned this fork directly, add the upstream remote so `git fetch upstream` works:

```bash
cd ~/perplexity-mcp-slim
git remote add upstream https://github.com/perplexityai/modelcontextprotocol.git
git fetch upstream
```

Check it stuck:

```bash
git remote -v
# origin    https://github.com/EngageWisdom/perplexity-mcp-slim.git (fetch)
# origin    https://github.com/EngageWisdom/perplexity-mcp-slim.git (push)
# upstream  https://github.com/perplexityai/modelcontextprotocol.git (fetch)
# upstream  https://github.com/perplexityai/modelcontextprotocol.git (push)
```

---

## License

MIT (inherited from upstream). See [LICENSE](LICENSE). Upstream authorship and copyright preserved.

---

# Upstream README (below)

Everything below this line is the upstream `@perplexity-ai/mcp-server` README, kept verbatim for reference. Note that the badges, install commands, and tool descriptions there reference the full four-tool upstream package, not this slim fork.

---

# Perplexity API Platform MCP Server

[![Install in Cursor](https://custom-icon-badges.demolab.com/badge/Install_in_Cursor-000000?style=for-the-badge&logo=cursor-ai-white)](https://cursor.com/en/install-mcp?name=perplexity&config=eyJ0eXBlIjoic3RkaW8iLCJjb21tYW5kIjoibnB4IiwiYXJncyI6WyIteSIsIkBwZXJwbGV4aXR5LWFpL21jcC1zZXJ2ZXIiXSwiZW52Ijp7IlBFUlBMRVhJVFlfQVBJX0tFWSI6IiJ9fQ==)
&nbsp;
[![Install in VS Code](https://custom-icon-badges.demolab.com/badge/Install_in_VS_Code-007ACC?style=for-the-badge&logo=vsc&logoColor=white)](https://vscode.dev/redirect/mcp/install?name=perplexity&config=%7B%22type%22%3A%22stdio%22%2C%22command%22%3A%22npx%22%2C%22args%22%3A%5B%22-y%22%2C%22%40perplexity-ai%2Fmcp-server%22%5D%2C%22env%22%3A%7B%22PERPLEXITY_API_KEY%22%3A%22%22%7D%7D)
&nbsp;
[![npm version](https://img.shields.io/npm/v/%40perplexity-ai%2Fmcp-server?style=for-the-badge&logo=npm&logoColor=white&color=CB3837)](https://www.npmjs.com/package/@perplexity-ai/mcp-server)

The official MCP server implementation for the Perplexity API Platform, providing AI assistants with real-time web search, reasoning, and research capabilities through Sonar models and the Search API.

Full upstream documentation: <https://github.com/perplexityai/modelcontextprotocol>
