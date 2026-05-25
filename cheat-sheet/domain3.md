# Domain 3 — Claude Code Configuration & Workflows: Cheatsheet

**Exam weight: 20%. Pass cut: 720/1000.**
**Appears primarily in: Code Generation with Claude Code, Developer Productivity Tools, Claude Code for CI/CD scenarios.**

---

## The Three Global Rules That Decide Most Questions

1. **Configuration > invocation.** When the team needs something to happen consistently, automatically, for everyone — the answer is a file in the right location, not a command someone has to remember to run.
2. **Path prefix is the whole answer.** `~/.claude/` is personal, never shared. `.claude/` (project) is shared via git. Same filename, opposite scope. Memorise the paths cold; reasoning won't save you.
3. **Load only what's needed.** Always-loaded standards live in CLAUDE.md. Pattern-matched standards live in `.claude/rules/` with `paths:` frontmatter. On-demand workflows live in skills/commands. Don't bloat one mechanism with another mechanism's job.

---

## TS 3.1 — CLAUDE.md Hierarchy

**The three levels — memorise the paths verbatim:**

| Level | Path | Scope | Version-controlled? |
|---|---|---|---|
| **User** | `~/.claude/CLAUDE.md` | Only you, on your machine | **No** |
| **Project** | `.claude/CLAUDE.md` or root `CLAUDE.md` | Everyone on the repo | Yes |
| **Directory** | `<subdir>/CLAUDE.md` | Only when working in that subdir | Yes |

**Directory CLAUDE.md scopes downward and local, never upward.** A `src/backend/CLAUDE.md` does NOT affect work in `src/`.

**THE most-tested 3.1 trap — the divergent-developers scenario:**

Setup: "Developer A's Claude Code follows team conventions. Developer B, on the same repo, same branch, gets inconsistent behaviour."

**Root cause 9/10 times:** Developer A wrote the conventions to `~/.claude/CLAUDE.md` (user-level, lives in their home directory). Git never carried it. Developer B's clone has nothing.

**Diagnostic instinct:** *Divergent behaviour between teammates on the same repo → suspect user-level vs project-level config first.*

**Trap distractors:**
- "Run `/memory reload`" — can't refresh a file that doesn't exist on the machine
- "Restart Claude Code" — same problem
- "Move into directory-level CLAUDE.md" — wrong tool for cross-cutting standards

**Modular organisation (two equally valid patterns):**
1. **`@import` syntax** inside CLAUDE.md — reference external files, import per-package standards
2. **`.claude/rules/` directory** — topic-specific files (`testing.md`, `api-conventions.md`, `deployment.md`) as an alternative to one massive CLAUDE.md

**`/memory` command — the diagnostic tool:**
Lists which memory files are currently loaded. **Use to confirm the diagnosis**, not to fix the bug. If a file you expected isn't there, you've found your bug — but the *fix* is to put the file in the right location, not to run `/memory` harder.

---

## TS 3.2 — Custom Slash Commands and Skills

**Directory structure — mirrors CLAUDE.md hierarchy:**

| Type | Location | Shared? |
|---|---|---|
| **Project slash commands** | `.claude/commands/` | Yes — version-controlled |
| **Personal slash commands** | `~/.claude/commands/` | No |
| **Project skills** | `.claude/skills/<name>/SKILL.md` | Yes |
| **Personal skills** | `~/.claude/skills/<name>/SKILL.md` | No |

Each skill is its own directory containing a `SKILL.md` file. The directory name becomes the skill name.

**Skill frontmatter — the three options to memorise:**

```yaml
---
name: brainstorm
description: Explore design alternatives for a feature
context: fork
allowed-tools: [Read, Grep]
argument-hint: <feature-name>
---
```

| Option | Effect | Exam trigger phrase |
|---|---|---|
| **`context: fork`** | Runs in isolated sub-agent context; verbose output stays in fork; only summary returns to main conversation | "produces verbose output", "clutters main conversation", "keep main context clean" |
| **`allowed-tools`** | Restricts which tools the skill can call; prevents destructive actions | "analyse only, must not modify", "restrict to read-only" |
| **`argument-hint`** | Prompts the developer for required parameters when invoked without arguments | "prompt the user for the parameter" |

**THE critical distinction — Skills vs CLAUDE.md:**

| | Skills | CLAUDE.md |
|---|---|---|
| **When loaded** | On-demand, when invoked | Always loaded |
| **Purpose** | Task-specific workflows | Universal standards |
| **Example** | `/review`, `/migrate-component` | "Use British English in comments" |

**Category errors the exam tests:**
- "Needs to happen every time" → CLAUDE.md (NOT a skill — skills are opt-in)
- "Invoked when needed" → skill (NOT CLAUDE.md — would bloat every interaction)
- "Noisy output, keep main context clean" → skill with `context: fork`

**Personal customisation pattern:**

Scenario: Team has a `/review` skill; one developer wants a stricter variant for personal use.

**Wrong:** edit the project skill (affects teammates).
**Right:** create `~/.claude/skills/review-strict/SKILL.md` — different name, personal location, no impact on others.

**Two independent axes (don't conflate):**
1. *Who needs it?* → location (`.claude/` vs `~/.claude/`)
2. *Is the output noisy?* → `context: fork` or not

A skill can be project-scoped AND forked. Personal AND unforked. These are orthogonal.

---

## TS 3.3 — Path-Specific Rules

**Mechanism:** files in `.claude/rules/` with YAML frontmatter declaring a glob:

```yaml
---
paths: ["**/*.test.tsx", "**/*.test.ts"]
---

All test files must use describe/it blocks, not test().
Mock external services using shared fixtures in tests/fixtures/.
```

Loads **only** when Claude Code is about to work on a file matching the glob. Not loaded for non-matching files.

**Why this beats directory-level CLAUDE.md (the comparison the exam tests directly):**

| | Directory CLAUDE.md | Path-specific rules |
|---|---|---|
| **Scope** | One directory only | Anywhere the glob matches |
| **Spread across codebase** | Need one CLAUDE.md per directory | One rule file, globs everywhere |
| **Token cost** | Loaded whenever working in that directory | Loaded only on matching files |

**Glob patterns match files scattered anywhere.** `**/*.test.tsx` catches every test file regardless of directory. A directory-level CLAUDE.md cannot do this — you'd need N copies that drift.

**Token efficiency angle (commonly tested):**
- Path-scoped rules load **only when editing matching files** → reduces irrelevant context and token usage
- Project-level CLAUDE.md would burn tokens on every interaction
- Directory-level CLAUDE.md would require duplicates

**When to reach for which mechanism (the mental decision table):**

| Need | Tool |
|---|---|
| Universal standards (always relevant) | Project-level CLAUDE.md |
| Standards for one specific subdirectory | Directory-level CLAUDE.md |
| Standards for files spread across the codebase by **pattern** | **Path-specific rules** |
| Task-specific workflow, invoked on demand | Skill |

**The signature phrase to recognise:** *"pattern of files spread across the codebase"* → path-specific rules.

Examples that all fit the pattern: `**/*.test.*`, `**/migrations/**/*`, `**/*.tf`, `**/api/handlers/**/*.ts`.

**Trap distractor:** a slash command for the convention. Slash commands require developers to **remember to invoke** them. Standards must be ambient, not opt-in.

---

## TS 3.4 — Plan Mode vs Direct Execution

**This is a judgement task statement.** The exam hands you a task description; you classify.

**Plan mode when:**
- Large-scale changes (library migration affecting 45+ files, monolith restructure)
- Multiple valid approaches exist (need to evaluate before committing)
- Architectural decisions required (service boundaries, abstractions)
- Multi-file modifications where changes interact
- Unfamiliar codebase needs exploration before design

**Direct execution when:**
- Well-understood, narrow scope (single-file bug fix, clear stack trace)
- Mechanical changes (add a validation conditional, rename a variable)
- The correct approach is already known

**The dividing line:** *Are there decisions still to make, or just a known thing to do?* Decisions → plan mode. Known thing → direct execution.

**The Explore subagent:**

| Property | Effect |
|---|---|
| Isolates verbose discovery output | Search results, file listings stay in subagent |
| Returns **summaries** to main conversation | Preserves main context window |
| Used during multi-phase tasks | Prevents context window exhaustion |

**Exam phrasing:** "main conversation filling up with grep output and file listings" → Explore subagent.

Same principle as `context: fork` for skills — keep noise out of the main conversation. Different mechanism, same goal.

**The combination pattern (commonly tested):**

You don't have to pick one mode for the whole task. Common workflow:
1. Enter plan mode
2. Explore the codebase, evaluate approaches
3. Produce a concrete plan
4. Exit plan mode
5. Implement the planned steps directly

If a question describes *"exploring, then implementing the chosen approach"* — that's the hybrid, not a single-mode choice.

**Common task classifications:**

| Task | Mode |
|---|---|
| Restructure monolith into microservices (boundaries undecided) | Plan mode |
| Fix null pointer in `UserService.getById()`, stack trace clear | Direct execution |
| Migrate from `winston` to `pino` across 30 files (API differs) | Plan mode, then direct execution |
| Add a date-format validation conditional to one function | Direct execution |
| React v17 → v18 across 200+ components, concurrent features undecided | Plan mode (then direct execution per file) |

**Trap:** calling a multi-file migration "mechanical" when the libraries have different APIs. Scale + interface differences = plan mode.

---

## TS 3.5 — Iterative Refinement

**Technique hierarchy — what to reach for when prose isn't working:**

### 1. Concrete input/output examples (highest leverage)

When prose descriptions get interpreted inconsistently, show 2–3 concrete examples:

```
Input:  user_name → Output: userName
Input:  api_key   → Output: apiKey
Input:  http_url  → Output: httpUrl
```

**The model generalises from examples more reliably than from descriptions.** Three examples eliminate ambiguity that prose admits.

**Exam phrasing:** *"Claude Code interprets the instruction differently each iteration"* → concrete input/output examples.

### 2. Test-driven iteration

Write tests first. Share failures. Claude iterates against a machine-checkable specification rather than your prose. Strongest when behaviour is well-defined but implementation is open.

### 3. Interview pattern

Have Claude **ask questions first** before implementing. Surfaces considerations you would miss in unfamiliar domains.

**Exam phrasing:** *"developer working in unfamiliar domain, wants to avoid missing edge cases"* → interview pattern.

**Batch vs sequence feedback:**

| Issue type | How to feed back |
|---|---|
| Fixes **interact** (changing one affects others) | **Single message** — Claude sees all together, makes consistent decisions |
| Issues are **independent** (orthogonal) | **Sequential** — fix one, verify, move on |

**THE headline rule:** when prose fails repeatedly, **switch modalities, don't iterate on the prose**. Rewording prose tends to produce *different* misinterpretations, not fewer. Examples constrain the output shape directly.

---

## TS 3.6 — CI/CD Integration

**This is the most memorisation-heavy task statement in the domain. Drill the flags.**

**The `-p` flag (print mode) — single most-tested CI detail:**

- `claude -p "<prompt>"` runs in **non-interactive mode**
- Without `-p`, the CI job **hangs waiting for interactive input**
- Symptom in exam questions: *"the CI pipeline hangs"* / *"Claude is waiting for input"* / *"printed banner, then nothing"* → answer is `-p`

**Trap distractors to reject:**
- "Increase the timeout" — treats symptom, not cause
- "Redirect /dev/null to stdin" — wrong layer, not the documented fix
- "Run in detached shell" — same problem

**Structured CI output (for automation):**

| Flag | Effect |
|---|---|
| `--output-format json` | Produces JSON instead of prose |
| `--json-schema <schema>` | Constrains JSON to a specific shape for reliable downstream parsing |

**Exam phrasing:** *"automated system needs to post findings as inline PR comments"* → both flags together.

**Session context isolation — the counter-intuitive one:**

> **The same Claude session that generated code is LESS effective at reviewing its own changes.**

It retains the reasoning context that produced the code → less likely to question its own decisions (motivated reasoning).

**The exam-correct pattern:** use an **independent review instance** — a fresh Claude Code session with no prior context for the review step.

**Trap:** the "efficient" answer feels like "reuse the session, it already knows the code." Wrong. Review needs fresh eyes.

**Incremental review context:**

When a PR gets new commits and you re-run the review:
- **Include prior review findings in context**
- **Instruct Claude to report ONLY new or still-unaddressed issues**

Without this: developers get duplicate comments on every push → lose trust → start ignoring the bot.

**Exam framing:** *"developers ignoring the bot's comments"* / *"comment fatigue"* / *"duplicate findings on every push"* → incremental review context, NOT "remove the bot".

**CLAUDE.md for CI:**

CI-invoked Claude Code reads the same `.claude/CLAUDE.md`. For test generation especially, document:
- Testing standards (frameworks, patterns)
- What makes a **valuable** test in your codebase
- Available fixtures, mocks, helpers

**Without this:** CI-generated tests are low-value boilerplate (testing getters, trivial assertions).
**With it:** tests target meaningful behaviour.

**Exam phrasing:** *"CI generates low-quality tests"* → strengthen `.claude/CLAUDE.md` with testing standards.

---

## Distractor-Recognition Cribsheet

When you see these answer options, **reject them**:

- "Run `/memory reload` to pick up the project's instructions" (when the file doesn't exist on the new dev's machine — it's user-level config that was never shared)
- "Restart Claude Code to pick up new configuration" (same problem)
- "Move the file to `~/.claude/CLAUDE.md` to make it load on demand" (wrong direction — that *removes* it from sharing)
- "Edit the project skill to add stricter behaviour" (when the developer wants personal customisation — create a personal skill instead)
- "Create a `/migrate` slash command everyone invokes before writing a migration" (standards must be ambient, not opt-in — use path-specific rules)
- "Put one CLAUDE.md in every directory that contains test files" (maintenance disaster, drift inevitable — use path-specific rules with `**` globs)
- "Compress the single CLAUDE.md by removing examples and rationale" (compression hides the structural problem — split into `.claude/rules/` with `paths:` frontmatter)
- "Direct execution — start editing files, adjust as you go" (for unfamiliar 80-file refactor with open architectural questions — plan mode)
- "Plan mode — even small changes benefit from explicit planning" (for one-line null-check fix with clear stack trace — ceremony, not value)
- "Rewrite the prose description more carefully" (when prose has failed 3+ times — switch modalities to concrete examples)
- "Increase the model's temperature" (for prose-misinterpretation problems — examples, not randomness)
- "Increase the CI timeout to 90 minutes" (for the hanging-CI scenario — `-p` fixes the root cause)
- "Redirect /dev/null to stdin" (same)
- "Switch to `--output-format text` so output isn't buffered" (the hang isn't an output problem — `-p`)
- "Reuse the code-generating session for review — it already knows the context" (motivated reasoning — use a fresh session)
- "Just remove the review bot, developers find it noisy" (the real fix is incremental review context, not removal)

When you see these, **lean toward them**:

- "The conventions are stored in user-level `~/.claude/CLAUDE.md` and were never version-controlled"
- "Move team standards from `~/.claude/CLAUDE.md` to `.claude/CLAUDE.md` (or root `CLAUDE.md`) and commit"
- "Split the monolithic CLAUDE.md into topic files under `.claude/rules/` with `paths:` frontmatter"
- "Create a personal skill in `~/.claude/skills/` with a different name to avoid affecting teammates"
- "Add `context: fork` to the skill so verbose output stays out of the main conversation"
- "Restrict skill capability with `allowed-tools: [Read, Grep]` to prevent destructive actions"
- "Path-specific rule with `paths: [\"**/*.test.*\"]` to apply test conventions across the codebase"
- "Path-specific rule with `paths: [\"**/migrations/**/*\"]` for migration conventions everywhere"
- "Plan mode for investigation and design; direct execution for implementation"
- "Use the Explore subagent to isolate verbose discovery output from the main conversation"
- "Provide 2–3 concrete input/output examples of the desired transformation"
- "Have Claude ask clarifying questions first before implementing (interview pattern)"
- "Add the `-p` flag to run non-interactively"
- "Add `--output-format json` with `--json-schema` for machine-parseable findings"
- "Use a fresh, independent Claude Code session for PR review"
- "Pass prior review findings into the next review and instruct Claude to report only new or unaddressed issues"
- "Document testing standards and valuable-test criteria in `.claude/CLAUDE.md` for CI to read"

---

## One-Line Mnemonics

- **Path prefix is the answer.** `~/.claude/` = personal. `.claude/` = shared.
- **Two devs, same repo, divergent behaviour:** user-level config that was never committed.
- **`/memory` is for diagnosis, not for fix.** It tells you what's loaded; it can't conjure a missing file.
- **Directory CLAUDE.md scopes downward and local, never upward.**
- **"Or" / "depending on" in a CLAUDE.md section:** split into `.claude/rules/` with `paths:` frontmatter.
- **Standards must be ambient.** Slash command for a convention = wrong tool. Path-rules or CLAUDE.md.
- **Pattern of files spread across the codebase** → path-specific rules with globs.
- **Same conventions in three sibling directory trees** → one path-rule with multiple globs, not three CLAUDE.md files.
- **Skill vs CLAUDE.md:** on-demand workflow vs always-loaded standard. Don't confuse the categories.
- **Personal customisation of a team skill:** new name, `~/.claude/skills/`. Never edit the team skill.
- **`context: fork`** for noisy skills. **Explore subagent** for noisy investigations. Same goal, different mechanism.
- **Plan mode = decisions still to make.** Direct execution = known thing to do.
- **Plan mode → direct execution** is a valid hybrid, not a wrong answer.
- **Library migration with API differences across N files:** plan mode regardless of "mechanical" framing.
- **Prose failed three times:** switch to concrete input/output examples. Don't rewrite the prose.
- **Interacting fixes:** one message. **Independent fixes:** sequence them.
- **CI hangs after banner:** `-p` flag. Always.
- **Automated PR comments need machine-parseable output:** `--output-format json --json-schema`.
- **Same session reviewing its own code:** motivated reasoning. Use a fresh session.
- **Developers ignoring the review bot:** incremental review context (prior findings + "only new issues").
- **Low-quality CI-generated tests:** strengthen `.claude/CLAUDE.md` with test-value criteria and fixtures.

---

## Configuration-Layer Decision Tree

When a question describes a need, ask in order:

1. **Does it need to happen every interaction?** → Project-level CLAUDE.md
2. **Does it apply to a pattern of files scattered across the codebase?** → Path-specific rule in `.claude/rules/` with `paths:` glob
3. **Does it apply only to one specific subdirectory?** → Directory-level CLAUDE.md
4. **Is it a workflow invoked on demand?** → Skill in `.claude/skills/` (or command in `.claude/commands/`)
5. **Is it personal-only?** → Same answer as 1–4 but in `~/.claude/` instead of `.claude/`
6. **Is it for the CI to consume?** → `.claude/CLAUDE.md` (still the project-level file — CI reads the same hierarchy)

If the answer to all of 1–4 is "no," you probably don't need configuration — you need a one-off prompt.

---

## Layer-Awareness Rule (carry-forward from teaching)

Many wrong answers come from reaching for the right idea at the wrong layer:

| Layer | What lives here |
|---|---|
| **User-level config** (`~/.claude/`) | Personal CLAUDE.md, personal commands, personal skills |
| **Project-level config** (`.claude/`) | Shared CLAUDE.md, shared commands, shared skills, path-rules, `.mcp.json` |
| **Directory-level config** | Subdirectory CLAUDE.md (downward-scoped) |
| **Runtime flags** | `-p`, `--output-format json`, `--json-schema` |
| **Skill frontmatter** | `context: fork`, `allowed-tools`, `argument-hint` |
| **Workflow mode** | Plan mode, direct execution, Explore subagent |

**When you feel the pull toward a fix, ask which layer the question is testing.** A CI hang is a runtime-flag question (`-p`), not a CLAUDE.md question. A divergent-behaviour bug is a path-prefix question (user vs project), not a `/memory` question. Don't cross layers.

---

Print this. Read it the morning of the exam.
