# Claude Certified Architect — Study Resources

Community study materials for the **Claude Certified Architect: Foundations** certification by Anthropic. Includes a full-length interactive practice exam, domain cheat sheets, and sample questions with detailed explanations.

**→ [Take the Practice Exam](https://practice-exam-deploy.vercel.app)**

---

## About the certification

The **Claude Certified Architect: Foundations** exam tests whether you can make the right architectural decisions when building production systems with Claude — not just whether you know the API. Questions are scenario-based: you're given a real production failure, a design choice, or a debugging situation, and asked to identify the root cause and the most effective fix.

Pass score is **720 / 1000**. The exam uses a scaled scoring model, and the hardest questions test judgment under constraint — when multiple options look valid, only one is architecturally sound.

Official courses and registration: [anthropic.skilljar.com](https://anthropic.skilljar.com/)

---

## What's in this repo

| Resource | Description |
|---|---|
| [`practice-exam.html`](practice-exam.html) | 54-question interactive practice exam across all 5 domains. No build step — open in any browser. |
| [`cheat-sheet/domain1.md`](cheat-sheet/domain1.md) | Agentic Architecture & Orchestration (27%) |
| [`cheat-sheet/domain2.md`](cheat-sheet/domain2.md) | Tool Design & MCP Integration (18%) |
| [`cheat-sheet/domain3.md`](cheat-sheet/domain3.md) | Claude Code Configuration & Workflows (20%) |
| [`cheat-sheet/domain4.md`](cheat-sheet/domain4.md) | Prompt Engineering & Structured Output (20%) |
| [`cheat-sheet/domain5.md`](cheat-sheet/domain5.md) | Context Management & Reliability (15%) |
| [`cheat-sheet/sample-questions.md`](cheat-sheet/sample-questions.md) | Official sample questions with analysis |

---

## Domain breakdown

```
Domain 1 — Agentic Architecture & Orchestration   27%  ████████████████████████████░░░░░░░░░
Domain 2 — Tool Design & MCP Integration          18%  ██████████████████░░░░░░░░░░░░░░░░░░░
Domain 3 — Claude Code Configuration & Workflows  20%  ████████████████████░░░░░░░░░░░░░░░░░
Domain 4 — Prompt Engineering & Structured Output 20%  ████████████████████░░░░░░░░░░░░░░░░░
Domain 5 — Context Management & Reliability       15%  ███████████████░░░░░░░░░░░░░░░░░░░░░░
```

### Domain 1 — Agentic Architecture & Orchestration (27%)

The highest-weighted domain. Tests how you design and debug multi-agent systems: hub-and-spoke topology, task decomposition, parallel execution, local error recovery, and when to use programmatic enforcement over prompt instructions. The core insight: **prompts are probabilistic, hooks are deterministic** — when a failure causes financial or security harm, only code-level enforcement is acceptable.

Key topics: agentic loops, stop_reason handling, coordinator/subagent isolation, tool prerequisite gates, idempotency, parallel Task calls.

### Domain 2 — Tool Design & MCP Integration (18%)

Tests whether you can design tools the model will use correctly. Tool descriptions are the routing layer — if two tools have near-identical descriptions, the model will misroute regardless of what your system prompt says. The exam also tests the critical distinction between `isError: true` (couldn't reach the source) and `isError: false` with empty results (reached it, found nothing) — these require opposite coordinator responses.

Key topics: description design, tool_choice modes, MCP configuration (`.mcp.json` vs `~/.claude.json`), error categories, principle of least privilege.

### Domain 3 — Claude Code Configuration & Workflows (20%)

Tests your understanding of the Claude Code file hierarchy and when each layer applies. The most common exam trap: conventions written to `~/.claude/CLAUDE.md` (user-level, never git-tracked) instead of `.claude/CLAUDE.md` (project-level, shared with the team). Also tests CI/CD usage (`-p` flag, `--output-format json`, `--json-schema`) and when to use plan mode vs direct execution.

Key topics: CLAUDE.md hierarchy, skills vs always-on instructions, path-scoped rules, plan mode triggers, non-interactive CLI flags, incremental PR review.

### Domain 4 — Prompt Engineering & Structured Output (20%)

Tests the difference between prompting that *looks* helpful and prompting that actually changes behavior. "Be conservative" sounds useful — it changes nothing because it gives the model no decision rule. "Flag only when described behavior contradicts actual code; skip TODO markers and style preferences" is specific enough to work.

Key topics: categorical criteria vs vague confidence instructions, few-shot examples for inconsistent output, tool_use guarantees (syntax only, not semantics), nullable schema fields, Message Batches API selection, multi-pass review architecture.

### Domain 5 — Context Management & Reliability (15%)

Tests how you handle the ways context degrades in long-running systems. Progressive summarization destroys high-precision tokens first — `$247.83 by Friday` becomes "a refund" and "soon." The fix: persistent fact blocks injected verbatim outside the summarized history. Also tests lost-in-the-middle mitigation, escalation triggers, graceful degradation with coverage annotations, and structured error propagation.

Key topics: persistent facts blocks, lost-in-the-middle positioning, explicit escalation triggers, isError semantics, graceful degradation, /compact vs fresh session.

---

## How to study

1. **Read the cheat sheet for a domain** — each one leads with the 3 global rules that resolve most questions, then covers specific mechanics and common exam traps.
2. **Take the practice exam domain by domain** — use "Study Domains First" on the exam start screen to go through the study guide before each section.
3. **Review missed questions** — the exam shows full explanations and surfaces all missed questions at the end with the correct reasoning.
4. **Focus on Domain 1** — it's 27% of the exam and the concepts (hooks vs prompts, coordinator tracing, parallel Task calls) appear as distractors in other domains too.

If you already work with Claude regularly, the cheat sheets alone cover roughly 80% of what you need.

---

## Run the practice exam locally

No dependencies, no build step:

```bash
git clone https://github.com/hamzafarooq/claude-certified-architect
open claude-certified-architect/practice-exam.html
```

---

## Contributing

Got a question from the real exam that you're allowed to share? Found an explanation that could be sharper? PRs are welcome.

See [CONTRIBUTING.md](CONTRIBUTING.md) for the question format and submission checklist.

---

## License

MIT — see [LICENSE](LICENSE).

---

Built by [Hamza Farooq](https://github.com/hamzafarooq) · [LinkedIn](https://linkedin.com/in/hamzafarooq) · [Newsletter](https://boringbot.substack.com/)
