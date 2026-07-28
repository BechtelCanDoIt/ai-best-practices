# How to Keep MCPs Low-Context
**Engineering Reference — Based on Field Reports**

---

## Overview

As MCP tool catalogs grow, naive approaches flood the model's context window with hundreds of tool manifests on every turn. This guide covers eight architectural patterns to keep that footprint small, deterministic, safe, and fast. Patterns 1–5 are context-reduction patterns; patterns 6–8 cover routing hygiene, result hygiene, and guardrails for a **federated proxy** that fronts external MCP servers. Each section describes the problem, the solution, implementation steps, and tradeoffs.

---

## Pattern 1: Expose Minimal Surface (Split Large MCP Servers)

### Problem

A single large MCP server exposing 30–50 tools injects every manifest into context on every turn — even for tasks that only need 2 or 3 of them. This wastes tokens, confuses routing, and increases latency.

### Solution

Break large MCP servers into smaller, semantically cohesive units. Each server should represent a single capability domain (e.g., "file ops," "API calls," "database queries"). A practical ceiling is **4 servers per software component**.

### How to Implement

1. Audit your existing MCP server(s) and list all exposed tools.
2. Group tools by semantic domain (nouns they operate on, or workflows they belong to).
3. Create a separate MCP server binary/process per group.
4. Update client configs (`mcp.json` or equivalent) to reference the new servers.
5. Run a smoke test: confirm each server loads only tools from its domain.

**Example split for an API platform MCP:**
```
apim-admin-mcp     → tenant/user/policy management tools
apim-analytics-mcp → metrics, logs, usage queries
apim-dev-portal-mcp → app/subscription/key management
```

### Tradeoffs

| Pro | Con |
|-----|-----|
| Each turn loads only the relevant server | More processes to deploy and monitor |
| Easier to version and update individual domains | Requires cross-server coordination for multi-domain tasks |
| Smaller blast radius if a server misbehaves | Initial refactor effort |

---

## Pattern 2: Lazy Tool Loading / Deferred Discovery

### Problem

Injecting all tool manifests at session start means the model pays the context cost for every registered tool on turn 1 — before it even knows what the user wants.

### Solution

Instead of injecting tool manifests upfront, expose a single **ToolSearch** (or `discover_tools`) tool. When the agent needs a capability, it queries the discovery tool with a natural language description, gets back the top-k relevant manifests, and those manifests are injected for the *next* turn only.

### How to Implement

1. Create a `tool_search` MCP tool that accepts a `query: string` and returns `Array<ToolManifest>`.
2. Back it with a searchable index (see Pattern 3 for the registry approach).
3. Remove all other tools from the initial context injection.
4. Instruct your system prompt: *"Use `tool_search` before calling any tool you haven't confirmed is available."*
5. On the next turn, inject only the returned manifests alongside the conversation history.

**System prompt snippet:**
```
You have access to one tool: `tool_search(query)`.
Call it with a description of what you need before attempting any other tool call.
```

### Tradeoffs

| Pro | Con |
|-----|-----|
| Near-zero manifest cost on turn 1 | Adds one extra round-trip before first tool use |
| Scales to hundreds of tools without context pressure | Discovery accuracy depends on index quality |
| Model only ever sees relevant tools | Requires maintaining the discovery index |

---

## Pattern 3: Semantic Router / Tool Registry

### Problem

Even with deferred loading, you need a reliable way to match a natural-language request to the right tool manifest. Keyword search is brittle; loading everything to let the model decide defeats the purpose.

### Solution

Build a **tool registry** backed by a semantic graph or vector index. The router accepts a query, embeds it, and returns the **top-k** most relevant tool manifests. Only those manifests enter the model's context.

### How to Implement

1. For each tool, write a rich natural-language description and embed it (e.g., using `nomic-embed-text` via Ollama).
2. Store embeddings in a vector store (pgvector is already available in your stack).
3. Expose a `discover_tools(query, k=5)` endpoint that performs cosine similarity search and returns the top-k tool schemas.
4. Wire this endpoint as the single entry-point tool in the model's initial context.
5. Tune `k` — start at 5, raise if the agent frequently needs follow-up discovery calls.

**Pseudocode:**
```python
def discover_tools(query: str, k: int = 5) -> list[ToolSchema]:
    embedding = embed(query)
    results = pgvector_search(embedding, k=k)
    return [load_schema(r.tool_id) for r in results]
```

### Tradeoffs

| Pro | Con |
|-----|-----|
| Scales to 100+ tools with low context cost | Embedding quality determines routing accuracy |
| Semantic matching handles paraphrasing and synonyms | Cold-start: all tools must be indexed before use |
| Single discovery interface, easily versioned | Needs vector infrastructure (already available: pgvector + Ollama) |

---

## Pattern 4: Higher-Level Abstractions (Skills, CLIs, Subagents)

### Problem

Fine-grained tools (one per API endpoint, one per file operation) explode manifest count and require the model to chain many calls to complete a task. The manifest overhead compounds quickly.

### Solution

Replace clusters of fine-grained tools with **higher-level abstractions**:

- **Skills** — encapsulate a workflow behind a single callable with structured I/O
- **CLI bridges** — let the model issue shell commands through a single `run_cli(command)` tool instead of loading individual tool schemas
- **Subagents** — offload heavy-tool workflows (Playwright, Figma, large API clients) to a child agent that runs its own context window

### How to Implement

**Option A — Skills:**
1. Identify clusters of 3–5 tools always called together.
2. Write a skill function that orchestrates them server-side.
3. Expose the skill as a single MCP tool with a descriptive schema.

**Option B — CLI Bridge:**
1. Wrap your toolset in a CLI (`my-tool-cli --action list-users --filter active`).
2. Expose a single `run_cli(command: string)` MCP tool.
3. The model discovers available commands via `run_cli("--help")` rather than loading manifests.

**Option C — Subagents:**
1. For heavy MCPs (Playwright, etc.), spin a child agent process with its own system prompt and tool set.
2. Expose the child agent to the parent as a single `delegate_to_browser_agent(task: string)` tool.
3. The parent never sees Playwright's manifests — only the delegated result.

### Tradeoffs

| Pro | Con |
|-----|-----|
| Dramatically reduces parent context size | Skills require upfront engineering |
| Subagents can run specialized, pre-warmed contexts | CLI execution carries security risk (see Safety section) |
| CLIs are discoverable via `--help` without embedding | Subagents add orchestration complexity |

---

## Pattern 5: Per-Session Tool Sets / Project Scoping

### Problem

A single `mcp.json` that loads every registered server means a developer debugging an integration also has Playwright, Figma, and database admin tools in context — irrelevant noise that costs tokens and creates risk.

### Solution

Maintain **multiple tool-set configuration files** and select the appropriate one per session or project. The agent only loads tools relevant to the current task scope.

### How to Implement

1. Create a base `mcp.json` with discovery-only tools (see Pattern 2).
2. Create scoped config files:
   - `mcp.apim.json` — APIM admin tools
   - `mcp.infra.json` — Docker, SSH, systemd tools
   - `mcp.analytics.json` — metrics, log queries
3. Pass the config at session start:
   ```
   claude --mcp-config mcp.apim.json
   ```
   Or set via environment variable / project `.claude/` directory.
4. Document which config to use for which project in your team wiki.

**Directory layout:**
```
.claude/
  mcp.default.json     ← discovery + common utils only
  mcp.apim.json        ← APIM-scoped tools
  mcp.acme.json        ← ACME engagement toolset
```

### Tradeoffs

| Pro | Con |
|-----|-----|
| Zero irrelevant tools in context for any given session | Requires discipline to pick the right config |
| Easy to audit what tools an agent session can access | Config files can drift from actual server state |
| Reduces accidental cross-domain tool invocations | Multi-domain tasks may need a merged config |

---

## Pattern 6: Namespace Tools & Write Router-Friendly Descriptions

### Problem

When one agent federates several MCP servers, tool names collide and blur. Two servers may both expose a `search` or `get_metric`; the model can't tell Splunk's log search from Datadog's log search, and a keyword/semantic router scoring on description text has nothing distinctive to match. Human-readable descriptions written for a docs page ("Returns metric data") give the router no signal about *which domain* the tool belongs to.

### Solution

Give every federated tool a **namespace prefix** and a **domain-tagged, action-oriented description**. The prefix disambiguates identical names across servers; the tag front-loads the routing signal so a discovery query like `"Datadog metric"` scores the right tool first. This is cheap (a string prefix) and pays off on every discovery call.

### How to Implement

1. Prefix tool *names* by origin server at registration time — `splunk__run_query`, `datadog__get_metric`, `topology__correlate_trace`. Strip the prefix before dispatching to the backend.
2. Prefix tool *descriptions* with a bracketed domain tag — `[splunk] Run an SPL query…`, `[correlation] Given a trace_id, return…`. Do this both in the source server's manifest (so it is self-describing) and, if the client enriches, at registration.
3. Write descriptions for the router, not a human: lead with the verb and the noun it operates on, name the domain, and mention the key input. Avoid marketing prose.
4. Do **not** duplicate the tool list inside the system prompt — the model already receives it via the `tools` API parameter. A hardcoded list goes stale the moment a tool is renamed.

### Tradeoffs

| Pro | Con |
|-----|-----|
| Eliminates cross-server name collisions | Slightly longer tool names in traces/logs |
| Router accuracy jumps with zero infra | Prefix convention must be applied consistently |
| Descriptions become self-documenting | Requires a one-time pass over existing manifests |

---

## Pattern 7: Tool-Result Hygiene (Bound & Neutralize)

### Problem

Discovery keeps *manifests* out of context, but tool **results** flow straight back into the loop — and they can be huge and hostile. A single Splunk query can return thousands of log lines; dumping them verbatim blows the context budget you worked to protect. Worse, result content is often attacker-influenced (a log line, an error message, a trace annotation) and can carry a prompt-injection payload that hijacks routing — especially dangerous when the agent can execute mutating runbooks.

### Solution

Treat every tool result as untrusted, unbounded input. **Bound** it (truncate or server-side summarize to a token ceiling) and **neutralize** it (strip/escape control sequences and instruction-like content) before it re-enters the agent loop as context.

### How to Implement

1. Cap result size at the source: paginate or `head`-limit large query tools; return a count + top-N + a follow-up handle rather than the full payload.
2. Summarize server-side when the raw data isn't needed verbatim — return the aggregate the agent actually reasons over, not the firehose.
3. Escape or fence untrusted result text so embedded "ignore previous instructions…" cannot be read as a directive; wrap it in a clearly-delimited data block.
4. Log the originating tool + query alongside the (bounded) result for audit and replay.

### Tradeoffs

| Pro | Con |
|-----|-----|
| Keeps context flat even on chatty tools | Truncation can hide a relevant tail row |
| Closes the top prompt-injection vector | Summarization adds server-side work |
| Bounded results = predictable latency/cost | Requires a per-tool size policy |

---

## Pattern 8: Propose-Before-Act & Least-Privilege for Mutating Tools

### Problem

Read tools are cheap to get wrong; **mutating** tools are not. A tool that restarts a service, flushes a cache, freezes deploys, or resets chaos can cause an outage if the model calls it on a bad inference or a poisoned result (Pattern 7). Exposing such tools with no approval step and no authentication turns a routing mistake — or any caller on the network — into a production incident.

### Solution

Split the catalog into read (safe, autonomous) and write (gated) tiers. Require **human-in-the-loop approval** before any mutating call, enforced in the prompt *and*, where possible, the transport. Put **authentication and least-privilege** in front of the mutating surface so only authorized callers reach it.

### How to Implement

1. Make propose-before-act a hard rule: the agent must list the candidate action and its parameters and wait for explicit approval before invoking a write tool. Never let a write tool run inside an autonomous loop unprompted.
2. Keep mutating tools scoped and idempotent — a fixed allowlist of runbooks with typed params, not an open `run_arbitrary_command`.
3. Authenticate the endpoint: require a bearer token (or delegate to an MCP gateway) so an unauthenticated caller can't invoke write tools directly.
4. Run the executor as a least-privilege service account and audit-log every mutating invocation with its originating prompt context.

### Tradeoffs

| Pro | Con |
|-----|-----|
| A routing/injection error can't mutate prod on its own | Approval step interrupts full autonomy |
| Auth shrinks the attack surface to authorized callers | Token/gateway management overhead |
| Audit log gives a reviewable action trail | Allowlist must be maintained as runbooks evolve |

---

## Safety & Security Notes

These patterns increase power but introduce risks that need explicit guardrails:

**CLI / Execute Gateway Risk**
A `run_cli` tool running as your user can execute destructive operations. Mitigations:
- Run CLI bridges as a dedicated low-privilege service account.
- Allowlist permitted commands; reject anything not on the list.
- Log all invocations with the originating prompt context.

**Sandbox Choice for Code Mode**
If using a code-mode executor (Pattern 1/2 complement), choose the sandbox carefully:
- **`quickjs-emscripten`** — safer for adversarial inputs; fewer known escapes.
- **`RestrictedPython`** — has known escape vectors; avoid for untrusted inputs.

**Prompt Injection via Tool Results**
Tool results containing attacker-controlled text can hijack routing. Always strip or escape result content before it re-enters the agent loop as context.

---

## Quick-Reference Checklist

```
[ ] Split large MCP servers into ≤4 semantic units per component
[ ] Add a ToolSearch/discover_tools gateway; remove upfront manifest injection
[ ] Back the registry with pgvector + nomic-embed-text (already in stack)
[ ] Replace fine-grained tool clusters with Skills, CLI bridges, or Subagents
[ ] Create per-project mcp.*.json files; document which config maps to which engagement
[ ] Namespace federated tool names; write domain-tagged, router-friendly descriptions
[ ] Bound and neutralize tool results before they re-enter the loop
[ ] Gate mutating tools behind propose-before-act approval + endpoint auth
[ ] Restrict CLI/execute tools to a least-privilege service account
[ ] Choose quickjs-emscripten over RestrictedPython for any code-mode sandbox
```

---

*Last updated: June 2026*
