# Domain 1 — Agentic Architecture & Orchestration: Cheatsheet

**Exam weight: 27% (highest single domain). Pass cut: 720/1000.**

---

## The Three Global Rules That Decide Most Questions

1. **Deterministic > probabilistic when stakes are real.** Financial / security / compliance / safety = hooks or programmatic gates. Style / tone / formatting = prompts.
2. **Trace failures to their origin.** Scope problems = upstream (decomposition / partitioning / context-passing). Quality problems = subagent prompts / tools. Don't blame the agent that faithfully executed a bad brief.
3. **Proportionate fixes.** Single agent beats multi-agent when single suffices. Prompts beat hooks when stakes are low. Don't over-engineer.

---

## TS 1.1 — Agentic Loops

**Lifecycle:** send → inspect `stop_reason` → if `tool_use`, execute tools, append `tool_result` blocks as `role: "user"`, loop → if `end_turn`, done.

**`stop_reason` is the ONLY authoritative termination signal.**

**Three anti-patterns (all wrong answers):**
- Parsing natural language for "I'm done" phrases
- Iteration cap as the *primary* stopping mechanism (caps are safety nets, not termination)
- Checking `content[0].type == "text"` — tool_use responses can begin with a text block

**Tool result shape:**
```python
{"role": "user", "content": [
  {"type": "tool_result", "tool_use_id": "<id>", "content": "...", "is_error": False}
]}
```
Parallel tool_uses → one tool_result per id, all in one user turn.

---

## TS 1.2 — Multi-Agent Orchestration

**Topology: hub-and-spoke only.** Coordinator at centre; subagents are spokes; ALL communication routes through the coordinator. Subagents NEVER talk to each other.

**The isolation principle (most-misunderstood concept):**
- Subagents do NOT inherit coordinator history
- Subagents do NOT share memory across invocations
- Every byte they need must be in their prompt

**Coordinator's 7 responsibilities:** decompose → select → partition scope → pass context → aggregate → iteratively refine → route for observability.

**Failure-tracing rules:**
| Symptom | Root cause |
|---|---|
| Missing topics in output | Decomposition too narrow |
| Duplicated work / redundant sources | Scope partitioning sloppy |
| Shallow on every topic | Subagent prompts or tool budgets |
| Missing source attribution | Context-passing destroyed metadata |

---

## TS 1.3 — Subagent Invocation & Context Passing

**Task tool** is the only mechanism to spawn subagents. **`allowedTools` must include `"Task"`** or the coordinator physically cannot delegate.

**AgentDefinition fields:** `description` (used for selection — write like a function docstring), `prompt`, `tools` (subset of coordinator's), optionally `model`.

**Context-passing rules:**
- Pass **complete findings**, not pointers
- Pass **structured metadata** (claim + source_url + snippet), not collapsed prose
- Write **goal-oriented** coordinator prompts, NOT procedural step-by-step

**Parallel spawning:** emit multiple `Task` tool_uses in a *single* coordinator response → executes concurrently. Sequential = separate turns = triple latency. Only safe when subagents are independent.

**`fork_session`:** independent branches from a shared expensive baseline. Use for "explore N alternatives from the same analysis."

**Decision:**
- Many isolated subagents, no shared baseline → parallel `Task`
- Branches that share an expensive computed state → `fork_session`

---

## TS 1.4 — Workflow Enforcement & Handoff

**Enforcement spectrum:**

```
PROBABILISTIC ──────────────────────────► DETERMINISTIC
prompt rules → few-shot → routing → hooks / prerequisite gates
~95-99%        ~98%      ~98%       100%
```

**The decision rule (memorise verbatim):**
> If a single failure causes financial loss, security breach, regulatory violation, or safety harm → **programmatic enforcement (hooks)**.
> If a single failure causes only a bad-but-recoverable UX → **prompts**.

**Canonical exam scenarios → all answer = hooks:** refund verification, trade limits, compliance checks, manager approvals, age/permission gates, audit logging.

**The four-distractor pattern for high-stakes questions:**
1. Hook / prerequisite gate ← **always correct**
2. Stronger system prompt
3. Few-shot examples
4. Specialised subagent with stronger prompt

**Multi-concern requests:** decompose → investigate in parallel with shared context → synthesise unified resolution.

**Human handoff payload (self-contained, no transcript expected):**
- Customer ID
- Conversation summary (2-3 sentences)
- **Root cause analysis** ← exam frequently tests this as missing
- Action taken so far
- **Recommended action** (incl. refund amount) ← exam frequently tests this as missing
- Compliance / urgency flags

---

## TS 1.5 — Agent SDK Hooks

| Hook | Fires | Use cases |
|---|---|---|
| **PreToolUse** | Before tool executes | Gates, authorisation, prerequisite checks, financial limits, capturing **prior state hash** for audit |
| **PostToolUse** | After tool, before model sees result | Normalisation (timestamps, currencies, field names), redaction (PII), truncation, enrichment, result logging |

**Keyword triggers:**
- "Prior state" / "pre-condition" / "before X happens" → **PreToolUse**
- "Heterogeneous formats" / "normalise" / "redact" / "model is confused by varying outputs" → **PostToolUse**

**Hooks vs subagents:**
- **Rule that must be enforced** → hook
- **Capability that needs different reasoning / prompt / tools** → subagent

**Hooks don't rank tools.** Hooks gate, block, or transform. Use prompts + tool descriptions for "prefer this tool over that one."

---

## TS 1.6 — Task Decomposition Strategies

| Pattern | When | Tell |
|---|---|---|
| **Fixed Sequential Pipeline** | Stable structure, each step consumes prior step's output | "Every email follows the same flow" |
| **Dynamic Decomposition (typed subagents)** | Varying requests, role-typed subagents (search/doc/synth) | "Same types invoked, scope varies per query" |
| **Orchestrator–Worker** | Uniform workers differentiated only by prompt-supplied scope | "N research workers, each handling a different sub-topic" |
| **Evaluator–Optimiser** | Explicit checkable quality criteria, first-pass often inadequate | "Iterate until coverage / citations / format pass" |

**Discriminator (Pattern 2 vs Pattern 3):** typed roles → dynamic decomposition; uniform clones → orchestrator–worker.

**Evaluator–optimiser fails when:** the evaluator cannot reliably distinguish good from bad. Subjective criteria → loop spins.

**Over-engineering trap:** if a single-agent loop works, the exam's preferred answer is to *not* refactor to multi-agent. Multi-agent adds latency, cost, and failure surface — warranted only when single-agent genuinely cannot do the job.

---

## TS 1.7 — Error Handling, Retries, Observability

**Three error categories:**
| Type | Examples | Response |
|---|---|---|
| **Transient** | 429, 503, network blip | Retry with exponential backoff + **jitter**, capped |
| **Permanent** | 400, 401, 403, 404, malformed input | Don't retry — feed back to model or escalate |
| **Logical** | Tool returned empty / irrelevant results | Feed back to model with context, model changes strategy |

**Anti-pattern:** blanket retry on every error type. Always classify first.

**Idempotency:** generate a UUID / hash of `{operation + customer + amount}` on the **first** attempt, pass it on every retry, server-side dedup. Without this, a transient retry can double-charge.

**Paired-fix pattern (exam favourite):** when both retry policy is broken AND duplication occurs, you need *both* error classification *and* idempotency keys. Either alone is insufficient.

**Errors must reach the model when the model can do something useful.** Silent harness error-swallowing (returning empty string for failed tool calls) makes the model produce confident wrong answers.

**Observability minimum bar:**
- Trace ID per run, span IDs per loop iteration and tool call
- Structured logging of stop_reason, tool calls, hook decisions, token usage
- Per-subagent cost & latency telemetry (multi-agent costs compound)
- Replay capability via stored history

**Reliability patterns:**
- **Circuit breaker** — N failures in window → stop calling for cooldown
- **Fallback agent / tool** — degraded answer > no answer
- **Cost / token budgets** — enforced at harness, not by prompt
- **Per-tool timeouts** — no tool blocks the loop indefinitely

---

## Distractor-Recognition Cribsheet

When you see these answer options for a high-stakes scenario, **reject them**:
- "Add a stronger system prompt instruction…"
- "Include few-shot examples demonstrating…"
- "Route to a specialised subagent whose prompt emphasises…"
- "Increase the model's temperature / use a larger model…"
- "Forward the full conversation transcript to the human reviewer"
- "Catch errors and return an empty string silently"
- "Retry on every error type with the same delay"
- "Have subagent A pass results directly to subagent B"

When you see these, **lean toward them**:
- "PreToolUse hook that blocks…"
- "PostToolUse hook that normalises / redacts…"
- "Idempotency key generated on first attempt"
- "Pass structured data with claim → source mappings"
- "Classify errors and handle each category appropriately"
- "Coordinator's decomposition step is too narrow" (for missing-topic scenarios)
- "Add `Task` to coordinator's `allowedTools`" (for "won't delegate" bugs)
- "Bounded timeout with fallback message" (for latency hangs)

---

## One-Line Mnemonics

- **Loop termination:** `stop_reason` or it's wrong.
- **Subagent state:** nothing carries over. The prompt is the universe.
- **High stakes:** hooks, every time, no matter how strong the prompt wording.
- **Missing topics:** blame the coordinator's decomposition.
- **Duplicate work:** blame the coordinator's partitioning.
- **Missing citations:** blame the context-passing format.
- **Won't delegate:** check `allowedTools` for `"Task"`.
- **Prior state in audit:** PreToolUse only.
- **Heterogeneous tool outputs:** PostToolUse normaliser.
- **Double-charged customer:** idempotency key.
- **Hanging agent:** timeout + fallback, not prompt instruction.
- **Evaluator–optimiser:** dies when criteria aren't objectively checkable.

Print this. Read it the morning of the exam.
