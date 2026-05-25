# Domain 2 — Tool Design & MCP Integration: Cheatsheet

**Exam weight: 18%. Pass cut: 720/1000.**
**Appears primarily in: Customer Support Resolution Agent, Multi-Agent Research System, Developer Productivity Tools scenarios.**

---

## The Three Global Rules That Decide Most Questions

1. **Tool descriptions are the routing layer, not documentation.** Misrouting is a description problem until proven otherwise. Fix descriptions before adding classifiers, few-shot examples, or merging tools.
2. **Fix at the cheapest layer that addresses the root cause.** Edit text before adding infrastructure. `tool_choice` before prompt instructions. Environment variables before relocating files. Scoped tools before merging agents.
3. **Errors must distinguish "tool couldn't look" from "tool looked and found nothing".** `isError: true` means access failure. `isError: false` with empty result means success-with-no-data. Conflating them breaks all recovery logic.

---

## TS 2.1 — Tool Interface Design

**The core principle:** tool descriptions ARE the routing mechanism. The model has nothing else when choosing between similar tools.

**A production-grade tool description contains 5 components:**
1. **Primary purpose** — one sentence
2. **Input contract** — formats, types, constraints
3. **Example queries it handles well** — concrete strings
4. **Edge cases and limitations** — what it won't do
5. **Explicit boundaries vs. similar tools** — "Use this for X. For Y, use `other_tool`." ← the one most teams skip

**The misrouting fix-ranking (memorise verbatim):**

| Fix | Rank | Why |
|---|---|---|
| **Expand tool descriptions** | **1st — correct** | Low effort, high leverage, fixes root cause |
| Few-shot examples in system prompt | 2nd | Treats symptom, token overhead on every call |
| Merge tools into one | 3rd | Loses useful semantic distinction, high refactor cost |
| Train a routing classifier | 4th — wrong | Extra model call, latency, new failure point |

**The exam's preferred answer is ALWAYS "expand descriptions" as the first step for a misrouting scenario.**

**Tool splitting — the other side:**

Code smell: a tool description containing "or" between behaviours, or "depending on" between conditions. Example:

> "Processes refunds, partial refunds, store credits, or exchanges depending on policy and customer status."

This is unsplittable at runtime. The model can't route to it reliably. **Split into purpose-specific tools** with tight contracts:
- `issue_full_refund(order_id, reason)`
- `issue_partial_refund(order_id, amount, reason)`
- `issue_store_credit(customer_id, amount, reason)`
- `process_exchange(order_id, replacement_sku, reason)`

**Renaming a generic tool does NOT fix this. Splitting does.**

**The system prompt interaction trap:**

Even with perfect tool descriptions, misrouting can persist if the system prompt contains keyword-sensitive instructions ("Always look up the customer first") that override tool descriptions.

**Rule: after fixing tool descriptions, audit the system prompt for keyword conflicts.** Don't add *more* routing rules to the prompt — that competes with the tool descriptions. Remove the conflicting keyword associations instead.

---

## TS 2.2 — Structured Error Responses

**The MCP `isError` flag is the agent's primary recovery signal.** Without structured error metadata, agents guess — and guess badly (retry things that won't succeed, escalate things that needed a moment).

**The four error categories (memorise the table):**

| Category | Examples | Retryable? | Agent response |
|---|---|---|---|
| **Transient** | Network timeout, 503, rate limit | Yes (same input) | Backoff + retry, then escalate |
| **Validation** | Wrong format, missing field, malformed ID | Yes (after fixing input) | Correct input and retry |
| **Business** | Refund exceeds limit, account locked, not eligible | **No** | Switch workflow; surface description to user |
| **Permission** | 401, 403, missing scope | No (by this agent) | Escalate or request elevated creds |

**Structured error payload shape:**
```json
{
  "isError": true,
  "errorCategory": "business",
  "isRetryable": false,
  "description": "Refund of £450 exceeds £200 self-service limit. Customer must speak with a supervisor."
}
```
The `description` for business errors should be customer-friendly — the agent will often surface it to the user.

**THE most-tested distinction in 2.2:**

| Case | Response | Agent action |
|---|---|---|
| **Access failure** (tool can't reach source) | `isError: true`, category transient/permission | Decide whether to retry |
| **Valid empty result** (tool queried successfully, found nothing) | `isError: false`, empty array / null result | Tell user "no results" |

**Conflating these is the canonical 2.2 exam trap.** Returning `[]` for both "no such customer" AND "database unreachable" causes the agent to retry endlessly and escalate with a misleading description, when the real answer was "this customer doesn't exist."

**Retryability subtlety:**
- Transient `isRetryable: true` = retry **with the same input**
- Validation `isRetryable: true` = retry **only after correcting the input**

**Error propagation in multi-agent systems:**

Subagents handle local recovery. Only propagate what they cannot resolve.

- Transient error in a subagent → retry locally, fall back to alternative source, *then* escalate
- When propagating, include: **what was attempted**, **partial results if any**, **recommended action**
- Coordinator should evaluate partial results from successful subagents rather than aborting the whole task on a single subagent failure

**Retry budget:** even retryable errors have limits. A well-designed tool may include `retryAfter` and `maxRetries` hints. "Agent retried 50 times" in a setup = missing retry budget.

---

## TS 2.3 — Tool Distribution and `tool_choice`

**Tool overload:**
- **4-5 tools per agent is the production sweet spot.**
- Reliability degrades sharply past 7-10 tools — softmax over descriptions thins probability margins
- **Scope tools to roles**: a synthesis agent should NOT have web_search; a web search agent should NOT have document analysis tools

**The `tool_choice` values (know each verbatim):**

| Need | `tool_choice` |
|---|---|
| Model may or may not call a tool | `"auto"` (default) |
| Model MUST call some tool, of its choice | `"any"` |
| Model MUST call this specific tool | `{"type": "tool", "name": "extract_intent"}` |

**Force-a-specific-first-step pattern (exam favourite):**

Scenario: "Every customer message must first extract intent and entities before routing. Currently done via system prompt instruction; ~15% of messages skip the extraction."

**Correct fix:** set `tool_choice: {"type": "tool", "name": "extract_intent"}` on the first turn. Switch to `"auto"` for subsequent turns. The model literally cannot output anything other than the forced tool call. 15% miss rate → 0%.

**Distractors to reject:**
- "Add stronger instruction to system prompt" — probabilistic, what caused the 15% miss in the first place
- "Use `tool_choice: any`" — model could pick a different tool
- "Validate after the fact" — too late, wrong layer

**Rule: when the requirement is "the model MUST do X first", reach for `tool_choice: {"type": "tool", "name": "X"}`, not a prompt instruction. Prompts request; `tool_choice` requires.**

**THE Q9-style pattern — scoped cross-role tools:**

Setup: synthesis agent frequently returns control to coordinator for one-line fact verifications. 85% are simple lookups, 15% need multi-source cross-referencing. Round-trip latency dominates.

**Wrong fixes:**
- Give synthesis agent full `web_search` + `fetch_page` → defeats role scoping, tool overload, prompt injection risk
- Cache → doesn't help with novel claims
- Merge synthesis + coordinator → destroys architecture
- Train a smaller model → massive effort for a tool-design problem

**Correct fix:** add a **scoped, constrained** `verify_fact(claim, max_sources=3)` tool to the synthesis agent. Tool description **explicitly states** that complex multi-source verifications must be routed back to the coordinator.

**Pattern in one sentence:** for high-frequency simple operations, give the consuming agent a constrained version of the capability; complex cases still escalate to the owner.

**Replacing generic tools with constrained alternatives:**
- `fetch_url(url)` → `load_document(document_url)` — validates allowed domains, document content-types, strips scripts
- `run_sql(query)` → `query_orders(filters)` — single-table, parameterised
- `send_email(...)` → `send_customer_notification(template_id, ...)` — templated, restricted recipients

Same capability, narrower surface, reduced blast radius.

---

## TS 2.4 — MCP Server Integration

**Scoping hierarchy:**

| Level | Path | Version-controlled? | Purpose |
|---|---|---|---|
| **Project** | `.mcp.json` (repo root) | Yes (committed, shared) | Team-wide integrations for this project |
| **User** | `~/.claude.json` | No (personal) | Personal/experimental servers across all projects |

**Tool discovery happens at connection time — all tools from all configured servers are made available simultaneously.** Too many configured servers re-creates the tool-overload problem at a different layer.

**Credential handling — `${VAR}` expansion (most-tested 2.4 pattern):**

Wrong:
```json
{ "env": { "JIRA_API_TOKEN": "ATATT3xFf_actual_token_here" } }
```

Right:
```json
{ "env": { "JIRA_API_TOKEN": "${JIRA_API_TOKEN}" } }
```

**Rule: "shared config, personal credentials". The `.mcp.json` structure stays committed and shared. Secrets live in each developer's local environment.**

**Don't fix a credential leak by relocating the file.** A *team* integration belongs in `.mcp.json` so new teammates have it. The fix is parameterisation, not relocation. **Also: rotate any leaked token — git history is forever.**

**MCP resources (content, not actions):**
- Tools = *actions* the agent can take
- Resources = *content* the agent can read (catalogues, schemas, summaries)

Example: `jira://projects/INGEST/summary`, `db://schemas/orders`, `github://repos/acme/api/issues/labels`.

**Exam framing:** "Agent makes many exploratory tool calls at conversation start to discover available data" → expose **resources** that catalogue what's available. The agent reads the catalogue once, then queries precisely.

**Build vs. use decision:**

**Use a community MCP server when:**
- Standard third-party system (Jira, GitHub, Slack, Notion, Postgres)
- Maintained server exists
- Usage pattern fits

**Build custom only when:**
- Team-specific internal system
- Community server fundamentally cannot support the workflow
- Compliance/security policy prohibits the community server

**Exam preference: "Configuration over construction" and "fork over fresh build".** When a community server is 90% of what you need (e.g., just want different return fields), post-process its responses or fork it — don't write from scratch.

**The built-in-tool preference trap:**

When MCP tools overlap with Claude Code built-ins (Grep, Glob, Read), the agent may reach for the built-ins because their descriptions are tighter. **Fix: enhance the MCP tool's description with examples and explicit boundary clauses** (same principle as 2.1). E.g., "Use this for semantic queries like 'where do we handle payment retries'. Prefer Grep for exact-string searches like 'TODO: fix'."

---

## TS 2.5 — Built-in Tools

**Grep vs Glob — THE most-tested distinction:**

| Tool | Searches | Returns | Mnemonic |
|---|---|---|---|
| **Grep** | File **contents** | Matching lines + filenames | "What does the code *say*?" |
| **Glob** | File **paths** | File paths matching pattern | "What files *exist*?" |

**Grep use cases:** function callers, error messages, import statements, TODO comments, identifier occurrences.
**Glob use cases:** all test files (`**/*.test.tsx`), all configs (`**/*.config.{js,ts,json}`), files by extension.

**Mirror traps:**
- "Find all React component files" → **Glob** `**/*.tsx`, NOT Grep `import React`
- "Find all callers of `formatDate`" → **Grep** `formatDate(`, NOT Glob (can't match function calls with a path pattern)

**Read / Write / Edit:**

| Tool | Cost | When |
|---|---|---|
| **Read** | Tokens proportional to file size | When you need to see contents |
| **Edit** | Low (diff-only) | Targeted modification via unique text anchor |
| **Write** | Full file in context | Complete rewrites or Edit fallback |

**Edit-failure escalation ladder (memorise):**
1. **Expand the `old_string` anchor** with surrounding context until unique — the FIRST move
2. If anchor cannot be made unique (truly identical surroundings) → **Read + Write** the full file
3. If intent was actually "change every occurrence" → use `replace_all: true`

Distractors to reject: "use sed" (wrong tool category), "delete and recreate" (destructive).

**Incremental codebase understanding (anti-pattern + correct pattern):**

**Anti-pattern: read everything upfront.** Loading ten 500-line files burns tens of thousands of tokens before any reasoning. Context budget destroyed.

**Correct pattern:**
1. **Grep** for entry points (function names, error strings, identifiers)
2. **Read** only the entry point file
3. **Grep** for the identifiers found in that file across the codebase
4. **Read** only the implementations that matter

**Wrapper-module tracing:** for codebases with re-exports (`index.ts` does `export { formatDate } from './date'`), Grep finds both definitions and re-exports. Identify which references are definitions vs. re-exports vs. call sites, then trace the chain.

**Grep → Glob sequencing pattern (exam favourite):**

Task: "Find all callers of `legacyAuth`, then find the test files for those callers."

Correct sequence:
1. **Grep** `legacyAuth(` → returns caller files (`userService.ts`, `paymentService.ts`, ...)
2. Derive expected test filenames from caller names
3. **Glob** `**/{userService,paymentService}.test.{ts,tsx}` → returns test files

**Rule: Grep first when you're discovering content; Glob second to narrow paths around the results.** Mirror form: known file set → check contents goes Glob → Grep. **The order depends on what is the seed vs. the filter.**

---

## Distractor-Recognition Cribsheet

When you see these answer options, **reject them**:

- "Train a routing classifier model…" (for misrouting — descriptions first)
- "Merge the two tools into one…" (for misrouting — loses distinction)
- "Add few-shot examples to the system prompt…" (for misrouting — token cost, treats symptom)
- "Add a stronger system prompt instruction telling the model to always call X first…" (use `tool_choice: {"type": "tool", "name": "X"}`)
- "Use a `PreToolUse` hook to enforce…" (Claude Code harness feature, NOT Anthropic API agent design — wrong layer)
- "Move the `.mcp.json` to `~/.claude.json` to fix the credential leak…" (use `${VAR}` expansion in the shared file instead)
- "Add `.mcp.json` to `.gitignore`…" (defeats team sharing)
- "Build a custom MCP server because we have specific workflows…" (evaluate community server first)
- "Give the synthesis agent full `web_search`…" (defeats role scoping)
- "Merge the synthesis and coordinator agents…" (destroys architecture)
- "Increase the retry count…" (for empty-result-misclassified-as-error scenarios)
- "Glob first to enumerate all source files, then Grep…" (when task is content-seeded)
- "Read every file in the directory upfront…" (context-budget killer)
- "Rename the tool to better reflect what it does…" (when the actual problem is splitting)

When you see these, **lean toward them**:

- "Expand both tool descriptions with purpose, input contract, examples, and explicit boundary clauses"
- "Split the generic tool into purpose-specific tools with tight contracts"
- "Audit the system prompt for keyword conflicts that override tool descriptions"
- "Return `isError: false` with an empty result for the valid-no-data case"
- "Distinguish access failures from valid empty results in the tool's error contract"
- "Subagent performs local recovery; coordinator evaluates partial results"
- "Set `tool_choice: {\"type\": \"tool\", \"name\": \"X\"}` to force the mandatory first step"
- "Add a scoped, constrained `verify_fact` tool to the consuming agent with explicit fallback to the coordinator for complex cases"
- "Replace the generic `fetch_url` with `load_document` that validates the URL surface"
- "Use `${VAR}` expansion in `.mcp.json`; rotate the leaked token"
- "Expose MCP resources for catalogue data so the agent stops probing"
- "Evaluate the community MCP server first; fork or post-process if a gap exists"
- "Enhance the MCP tool's description with examples and boundary clauses vs. built-in tools"
- "Grep for content first, derive filenames, Glob for related paths"
- "Expand the Edit anchor with surrounding context; fall back to Read + Write only if no unique anchor exists"

---

## One-Line Mnemonics

- **Misrouting:** descriptions are the routing layer — expand them first.
- **"Or" / "depending on" in a tool description:** split the tool.
- **Misrouting persists after fixing descriptions:** audit the system prompt for keyword conflicts.
- **`isError: true`:** the tool couldn't look. **`isError: false` + empty:** the tool looked and found nothing.
- **Empty array for "no such customer":** wrong — that's a valid empty result, not a transient error.
- **Validation retryable:** retry only after correcting the input. **Transient retryable:** retry with the same input.
- **Subagent failure:** local recovery first; only propagate what's unresolved, with partial results attached.
- **Coordinator should NOT abort** the whole task on one subagent failure if others returned results.
- **Tool count:** 4-5 per agent. More degrades selection.
- **"Must do X first":** `tool_choice: {"type": "tool", "name": "X"}` — never a prompt instruction.
- **High-frequency simple operation across roles:** scoped cross-role tool with explicit fallback to the owner.
- **Generic powerful tool:** replace with constrained alternative (`load_document` not `fetch_url`).
- **Credentials in committed `.mcp.json`:** `${VAR}` expansion + rotate the leaked token. Don't relocate the file.
- **Exploratory tool calls at conversation start:** expose MCP resources.
- **Custom MCP server proposed:** evaluate community server first. Fork before fresh build.
- **MCP tool losing to built-in:** enhance the MCP tool's description with examples and boundaries.
- **Grep vs Glob:** content = Grep, paths = Glob. Wrong choice = wasted time or impossible task.
- **Edit fails with "multiple matches":** expand the anchor first; Read + Write only as fallback.
- **Context-budget killer:** reading everything upfront. Grep for entry points, Read selectively.
- **Grep → Glob:** content-seeded discovery (find usages → find their tests). **Glob → Grep:** path-seeded check (known file set → check contents).
- **Claude Code hooks (`PreToolUse` etc.) are CLI-harness features.** API-level agent design uses `tool_choice`, `tools`, and structured tool results. Don't reach for a hook on an API-design question.

---

## Layer-Awareness Rule (carry-forward correction from teaching)

Many wrong answers come from reaching for the right idea at the **wrong layer**:

| Layer | Tools available |
|---|---|
| **Anthropic API agent design** | `tool_choice`, `tools`, system prompt, tool result structure, `isError` |
| **Claude Code CLI harness** | `PreToolUse`, `PostToolUse`, `Stop` hooks, settings.json |
| **MCP server config** | `.mcp.json`, `~/.claude.json`, `${VAR}` expansion, resources |
| **Tool implementation** | Error categories, retry hints, input validation, scoped surface |

**When you feel the pull toward a fix, ask which layer the question is testing. The exam is at the API + MCP + tool-design layers. Hooks are usually the wrong answer in Domain 2.**

Print this. Read it the morning of the exam.
