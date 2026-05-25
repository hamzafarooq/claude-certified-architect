# Domain 4 — Prompt Engineering & Structured Output: Cheatsheet

**Exam weight: 20%. Pass cut: 720/1000.**
**Appears primarily in: Claude Code for CI/CD, Structured Data Extraction scenarios.**

---

## The Three Global Rules That Decide Most Questions

1. **Specific categorical criteria beat vague confidence-based instructions.** "Flag X, report Y, skip Z" with concrete categories beats "be conservative" every time. If an option contains the words *conservative*, *careful*, or *high-confidence* without defining them in categorical terms, it is almost always a distractor.
2. **Match the mechanism to the failure mode.** Few-shot for inconsistent judgement. Schema design for fabrication. Validation-retry for output errors. Multi-instance for self-review limitations. Wrong-tool answers are the most common distractor shape in this domain.
3. **Retry fixes output errors. Schema design fixes source gaps.** Information genuinely absent from the source document is *not* recoverable through retry — only through nullable fields, "unclear" enums, or "other" categories. This boundary is tested in nearly every form.

---

## TS 4.1 — Explicit Criteria

**The core principle:** Specific categorical criteria obliterate vague confidence-based instructions.

| Wrong (vague) | Right (categorical) |
|---|---|
| "Be conservative." | "Flag comments only when claimed behaviour contradicts actual code behaviour." |
| "Only report high-confidence findings." | "Report bugs and security vulnerabilities. Skip minor style preferences and local patterns." |

**Structure to memorise:** *What to flag. What to report. What to skip.* Three categories, no adjectives about confidence.

**The false-positive trust problem — heavily tested:**

When one category has a high false-positive rate, developers stop trusting **all** categories, including the accurate ones.

**The counterintuitive fix:** temporarily **disable** the noisy category entirely while improving its prompt. Don't tighten in place — pull it from output, restore trust in what remains, then re-enable.

**Trap distractors:**
- "Lower the confidence threshold for that category" — keeps noise flowing
- "Add more instructions to the prompt for that category" — keeps noise flowing
- "Lower the temperature" — deterministic bad criteria is still bad criteria

**Severity calibration — the prose-vs-code distinction:**

Prose definitions of severity drift across runs. Anchor severity levels with **actual code examples** for each level.

```
Critical: SQL injection from unvalidated input
  Example: db.query("SELECT * FROM users WHERE id=" + req.params.id)

Medium: Missing error handling on non-critical path
  Example: fs.readFile(path, cb) where cb ignores err parameter
```

**Trap:** a "comprehensive rubric" of five severity levels described in prose. Looks thorough — fails because it lacks code anchors.

---

## TS 4.2 — Few-Shot Prompting

**The headline rule:** Few-shot examples are the most effective technique for **consistency**. Not more instructions. Not confidence thresholds. Not temperature changes. Examples.

**Three deployment triggers — memorise verbatim:**

1. Detailed instructions alone produce inconsistent formatting
2. Model makes inconsistent judgement calls on ambiguous cases
3. Extraction tasks produce empty/null fields for information that exists in the document

**Construction rules:**

| Property | Requirement |
|---|---|
| **Count** | 2–4 targeted examples for ambiguous scenarios (not 10, not 1) |
| **Content** | Each example shows the **reasoning** for why one action was chosen over plausible alternatives |
| **Purpose** | Generalisation to novel patterns, not pattern-matching pre-specified cases |

**The hallucination reduction effect — exam-favourite:**

Documents vary in structure: inline citations vs bibliographies, narrative prose vs structured tables. Few-shot examples covering varied structures **dramatically reduce hallucination** because the model learns the *shape* of valid extractions across formats, rather than inventing data to fit the schema.

**Trap distractors:**
- "Add more detailed step-by-step instructions covering all edge cases" — instructions grow linearly, edge cases grow combinatorially
- "Add 10 diverse examples" — wrong number; 2–4 targeted on ambiguous cases beats 10 on solved cases
- "Lower temperature to 0 and take the union of two runs" — determinism isn't a judgement problem

**Signature phrase to recognise:** *"inconsistent output across runs despite detailed instructions"* → few-shot examples.

---

## TS 4.3 — Structured Output with tool_use

**The reliability hierarchy:**

| Approach | Guarantees |
|---|---|
| Prompt-based JSON ("return JSON like...") | **Nothing.** Model can produce malformed JSON. |
| **tool_use with JSON schema** | **Eliminates syntax errors entirely.** |

That is the **only** thing tool_use guarantees. Syntactic validity. Nothing else.

**What tool_use does NOT prevent — the three persistent failure modes:**

1. **Semantic errors** — line items don't sum to stated total. Schema doesn't check arithmetic.
2. **Field placement errors** — right value, wrong field. Schema validates types, not meaning.
3. **Fabrication** — model invents values for required fields when source lacks the information.

**The fabrication factory:** a strict schema with all-required fields is a fabrication factory. *Required* doesn't guarantee correctness — it guarantees something will be present, even if invented.

**`tool_choice` — three modes the exam tests:**

| Value | Behaviour | When to use |
|---|---|---|
| `"auto"` (default) | Model may return text instead of calling a tool | When the model legitimately might have nothing to extract |
| `"any"` | **Must** call a tool, model picks which | Guaranteed structured output, unknown document type, multiple tools available |
| `{"type": "tool", "name": "..."}` | **Must** call this specific tool | Force a mandatory first step or single-tool workflow |

**Mental shortcut:** named-tool forces a *specific* tool; `"any"` forces *some* tool from the menu.

**Trap:** named-tool offered as a distractor when document type varies. Named-tool routes heterogeneous documents through one tool that doesn't fit most of them.

**Schema design — the anti-fabrication toolkit:**

| Technique | Failure mode it addresses |
|---|---|
| **Optional / nullable fields** | Source legitimately lacks the information → prevents fabrication |
| **`"unclear"` enum value** | Ambiguous cases → lets the model express uncertainty within the schema |
| **`"other"` + freeform `detail` string** | Categories outside the enum → captures reality instead of forcing wrong-bucket |
| **Format normalisation rules in the prompt** | Schemas validate types; prompts specify formats (e.g., ISO-8601 dates) |

**Trap distractors:**
- "Use tool_use with all fields required to guarantee complete extractions" — fabrication factory
- "Use tool_use to ensure line items sum to the total" — schemas don't do arithmetic
- "Add 'do not fabricate' to the prompt while leaving the field required" — prompt cannot override structural pressure

---

## TS 4.4 — Validation-Retry Loops

**The mechanism — three things sent back on validation failure:**

1. The original document
2. The failed extraction output
3. The specific validation error message

The model uses the error to **self-correct**. Dramatically more effective than naive retry, because the model now has signal about *what specifically went wrong*.

**The retry effectiveness boundary — most-tested concept in 4.4:**

| EFFECTIVE for | INEFFECTIVE for |
|---|---|
| Format mismatches (date format wrong) | **Information genuinely absent from the source** |
| Structural output errors (object vs array) | |
| Misplaced values (value in wrong field) | |

**Memorise as a sentence:** *Retry handles output errors. Schema design handles source gaps.*

**Trap:** a scenario where 15% of documents lack a field and the option says "add validation-retry to recover the missing data." Wrong — the data isn't missing from the extraction, it's missing from the source. Retry will produce fabrications, not corrections. The correct fix is **nullable field**.

**`detected_pattern` fields — systematic improvement:**

Add a `detected_pattern` field to each structured finding (e.g., `"unvalidated_user_input_in_sql_concatenation"`).

| Capability unlocked | Why it matters |
|---|---|
| Analyse dismissal patterns | If pattern X is dismissed 80% of the time, you have data to improve its prompt |
| Convert black-box tool to measurable system | Improvement based on evidence, not anecdote |

**Exam framing:** *"how do you improve a code review bot's accuracy over time"* → systematic data path via `detected_pattern`, not "iterate on the prompt".

**Self-correction flows — two patterns:**

1. **Calculated vs stated totals.** Extract `stated_total` AND `calculated_total` (sum the line items yourself). Disagreement flags OCR errors, fraud, or model arithmetic mistakes — three things schemas can't catch.
2. **`conflict_detected` boolean.** When source data is internally inconsistent (header date ≠ footer date), surface the conflict rather than silently picking one.

**Shared principle:** extract redundantly and let inconsistencies bubble up.

**Trap distractors:**
- "Add retry-with-feedback to handle all extraction failures" — conflates output errors with source gaps
- "Increase the retry limit from 3 to 10" — if 3 retries didn't fix it, more produce more fabrications, not more accuracy

---

## TS 4.5 — Batch Processing

**Message Batches API — five facts to memorise:**

1. **50% cost savings** vs synchronous
2. **Up to 24-hour processing window** — no faster, no slower-bounded
3. **No guaranteed latency SLA** — could return in 10 minutes or 23 hours
4. **Does NOT support multi-turn tool calling within a single request** — single-shot completions only
5. **Uses `custom_id`** for correlating request/response pairs

**The fourth point is the sleeper trap.** Agentic workflows (model calls tool, sees result, calls another tool) cannot run on batch.

**The matching rule — central exam test:**

| Workflow type | API choice |
|---|---|
| **Blocking** (someone or something is waiting) | **Synchronous** |
| **Latency-tolerant** (no human waiting) | **Batch** |

**Blocking examples:** pre-merge CI checks, IDE inline suggestions, customer-facing chat.
**Latency-tolerant examples:** overnight test generation, weekly compliance audits, monthly competitive intelligence.

**THE Q11-style trap — manager proposes "switch everything to batch to save 50%":**

The correct answer is **never** "migrate everything to batch." It is **always** the mixed strategy: batch the latency-tolerant workloads, keep blocking workloads synchronous. 50% saving on API cost is dwarfed by the cost of developers blocked waiting for a 6-hour batch return.

**Special case — multi-turn tool calls:** even if the workflow looks latency-tolerant, multi-turn agentic loops **cannot** run on batch. Must be synchronous.

**Batch failure handling:**

| Pattern | What it does |
|---|---|
| Identify failures by `custom_id`, resubmit only failures with modifications | Don't re-batch the whole 100,000; chunk oversized docs and resubmit just the failures |
| **Refine prompts on a sample set BEFORE batch processing** | Iteration loops in batch are ≤24h each; validate synchronously first |

**Trap distractors:**
- "Use batch for the CI/CD code review bot to save costs" — CI is blocking
- "Iterate on the prompt during batch runs" — iteration loops are 24h+, validate upfront
- "Resubmit the entire batch" when only 200 of 50,000 failed — wastes savings

---

## TS 4.6 — Multi-Instance Review

**The self-review limitation — foundational principle:**

> A model reviewing its own output in the same session retains the reasoning context that produced the output. It is biased toward its prior conclusions and **less** likely to question its own decisions.

**The fix:** spawn an **independent instance** — fresh session, no prior context — for the review step. The cold instance catches subtle issues the original misses.

**Trap distractor:** "ask the same model to review its work before returning." Self-review within session. Wrong. Use a fresh instance.

This is the same principle as the CI/CD session-isolation rule in Domain 3.6 — different scenario, same architecture.

**Multi-pass architecture for codebase-wide tasks:**

| Pass | Purpose |
|---|---|
| **Per-file local analysis** (one pass per file, focused context) | Consistent depth per file |
| **Separate cross-file integration pass** | Catches data flow issues, contract mismatches across files |

**Why single-pass fails on large repos:**

- **Attention dilution** — early files get more attention than late files (or vice versa) unpredictably
- **Contradictory findings** — model loses track of conclusions on file 3 by the time it reaches file 47
- **Misses cross-file issues** — cross-file reasoning competes with per-file detail

**Trap distractor:** "use a larger context window so the whole codebase fits in one pass." Context size doesn't fix attention dilution. Bigger window ≠ more attention per file.

**Confidence-based routing:**

| Step | What to do |
|---|---|
| Model self-reports confidence per finding | Numeric or categorical |
| Route low-confidence findings to human review | Auto-apply high-confidence findings |
| **Calibrate thresholds using a labelled validation set** | Don't pick by intuition |

**Trap:** "set confidence threshold to 0.9 to ensure quality" without mention of calibration. 0.9 in one system is permissive; in another it's prohibitive. Always calibrate empirically.

---

## Distractor-Recognition Cribsheet

When you see these answer options, **reject them**:

- "Be more conservative" / "only report high-confidence findings" (vague, unmeasurable — need categorical criteria)
- "Lower temperature to fix inconsistent judgement" (judgement isn't a randomness problem)
- "Lower the confidence threshold on the noisy category" (keeps noise flowing — disable and iterate)
- "Add more detailed instructions to fix inconsistent output" (instructions have ceilings — switch to few-shot)
- "Add 10 diverse few-shot examples to cover all cases" (wrong number — 2–4 targeted on ambiguous cases)
- "Use tool_use with all fields required to guarantee complete extractions" (fabrication factory)
- "Add 'do not fabricate' to the prompt while keeping fields required" (prompt can't override structural pressure)
- "Use tool_use to ensure line items sum to the stated total" (schemas don't do arithmetic)
- "Use `tool_choice: 'auto'` when structured output is required" (lets the model escape into prose)
- "Force `{type: 'tool', name: 'extract_invoice'}` for documents of unknown type" (named-tool ≠ heterogeneous routing)
- "Add validation-retry to recover fields missing from the source" (retry can't conjure absent data)
- "Increase the retry limit from 3 to 10 to recover difficult cases" (more retries ≠ more accuracy)
- "Switch everything to the Batch API to save 50%" (breaks blocking workflows)
- "Batch the CI code review bot" (CI is blocking)
- "Use batch for the multi-turn customer support agent" (batch doesn't support multi-turn tool calling)
- "Iterate on the prompt while batch is running" (24h iteration loops)
- "Resubmit the entire batch when 0.4% fail" (wastes the savings — resubmit failures by `custom_id`)
- "Ask the model to review its own findings before returning" (self-review in session — motivated reasoning)
- "Use a larger context window so the whole codebase fits in one pass" (attention dilution isn't a token problem)
- "Set confidence threshold to 0.9 to guarantee quality" (without calibration, the number is meaningless)
- "Increase temperature to encourage broader exploration of issues" (consistency problem, not exploration problem)

When you see these, **lean toward them**:

- "Flag X when condition Y; report Z; skip W" (categorical criteria, not adjectives)
- "Disable the noisy category temporarily while improving its prompt; keep other categories active" (restores trust)
- "Add severity-level definitions with concrete code examples for each level" (prose drifts, code anchors)
- "Add 2–4 few-shot examples covering ambiguous cases, each showing reasoning" (judgement consistency)
- "Add few-shot examples covering varied document structures (inline citations, bibliographies, narrative, tables)" (hallucination reduction)
- "Make the field nullable in the schema and instruct the model to return null when not present" (anti-fabrication)
- "Add an `'unclear'` enum value for ambiguous cases" (express uncertainty within the schema)
- "Add an `'other'` enum value plus a freeform detail string" (extensible categorisation)
- "Switch `tool_choice` to `'any'` so the model must call one of the available tools" (guaranteed structured output across document types)
- "Force a specific tool with `{type: 'tool', name: '...'}` to mandate a first step" (single-tool or mandatory-step workflows)
- "Add validation-retry with error feedback: original document + failed extraction + validation error" (output-error recovery)
- "Add a `detected_pattern` field to enable systematic analysis of dismissal patterns" (improvement over time)
- "Extract `calculated_total` alongside `stated_total` and flag discrepancies" (self-correction)
- "Add a `conflict_detected` boolean for internally inconsistent source data" (surface ambiguity)
- "Keep blocking workflows synchronous; batch only the latency-tolerant overnight jobs" (mixed strategy)
- "Identify failures by `custom_id` and resubmit only those with modifications" (efficient batch recovery)
- "Refine prompts on a sample synchronously before batch processing" (avoid 24h iteration loops)
- "Run extraction in a fresh, independent Claude instance for the review pass" (avoids motivated reasoning)
- "Run a per-file analysis pass, then a separate cross-file integration pass" (consistent depth + cross-file detection)
- "Route low-confidence findings to human review, with thresholds calibrated on a labelled validation set" (calibrated, not arbitrary)

---

## One-Line Mnemonics

- **Categorical criteria beat confidence adjectives.** "Flag X, report Y, skip Z" wins.
- **High false-positives in one category destroy trust in all categories.** Disable, iterate, re-enable.
- **Prose severity drifts. Code-example severity anchors.**
- **Inconsistent output despite detailed prompts → few-shot, not more instructions.**
- **2–4 targeted examples on ambiguous cases.** Not 10. Not 1.
- **Few-shot examples reduce hallucination by showing varied document structures.**
- **tool_use eliminates syntax errors. That's it.** Not semantics, not placement, not fabrication.
- **All-required schemas are fabrication factories.**
- **`"auto"` for optional. `"any"` for must-call-something. Named for must-call-this.**
- **Nullable fields, `"unclear"` enums, `"other"` categories — the anti-fabrication toolkit.**
- **Retry fixes output errors. Schema design fixes source gaps.**
- **Retry-with-feedback = original + failed output + specific error message.**
- **`detected_pattern` turns a black-box bot into a measurable system.**
- **Calculated vs stated total = self-correction without trusting either alone.**
- **Batch: 50% cost, ≤24h, no SLA, no multi-turn tool calling, `custom_id` correlates.**
- **Blocking → synchronous. Latency-tolerant → batch.** Never "everything to batch."
- **Multi-turn tool calls → synchronous, regardless of latency tolerance.**
- **Refine prompts synchronously before batching.** Don't iterate inside a 24h loop.
- **Resubmit batch failures by `custom_id`, not the whole batch.**
- **Same session reviewing its own work = motivated reasoning. Use a fresh instance.**
- **Per-file pass + cross-file integration pass beats one giant pass.**
- **Larger context window doesn't fix attention dilution.**
- **Calibrate confidence thresholds on a labelled validation set.** Intuition picks the wrong number.

---

## Mechanism-to-Failure Decision Tree

When a question describes a failure, ask in order:

1. **Inconsistent judgement across runs / inconsistent formatting?** → Few-shot examples (2–4, with reasoning)
2. **Malformed JSON?** → tool_use with JSON schema
3. **Required field fabricated when source lacks data?** → Nullable field in schema + prompt instruction to return null
4. **Ambiguous categorisation?** → `"unclear"` enum value or `"other"` + freeform detail
5. **Format mismatch in output (e.g., date format wrong)?** → Validation-retry with error feedback
6. **Line items don't sum to total / semantic inconsistency?** → Self-correction (extract both, compare, flag)
7. **Internally inconsistent source data?** → `conflict_detected` boolean, surface ambiguity
8. **Blocking workflow (someone waiting)?** → Synchronous API
9. **Latency-tolerant single-shot workflow?** → Batch API
10. **Multi-turn agentic workflow?** → Synchronous, always (batch doesn't support it)
11. **Inconsistent depth across files in a large repo?** → Per-file pass + cross-file integration pass
12. **Need a second opinion on Claude's work?** → Independent instance, fresh session
13. **Want to improve a review bot over time?** → `detected_pattern` field + analysis of dismissal patterns
14. **Bot output dismissed by developers because of noisy category?** → Disable that category; iterate; re-enable

---

## Layer-Awareness Rule (carry-forward from teaching)

Many wrong answers come from reaching for the right idea at the wrong layer:

| Layer | What lives here |
|---|---|
| **Prompt content** | Criteria, severity definitions with code examples, format normalisation rules |
| **Few-shot examples** | Judgement consistency, varied-structure generalisation, hallucination reduction |
| **Schema design** | Anti-fabrication (nullable, unclear, other), required vs optional |
| **tool_choice** | Whether and which tool gets called |
| **Validation-retry** | Recovering output errors |
| **Pipeline architecture** | Multi-pass review, calculated vs stated, conflict booleans |
| **API selection** | Synchronous vs batch based on blocking vs latency-tolerant |
| **Multi-instance** | Fresh sessions for independent review |

**When you feel the pull toward a fix, ask which layer the question is testing.**
A fabrication problem is a schema-design question (nullable), not a prompt-content question ("don't fabricate"). A batch-failure question is a `custom_id` question, not a retry question. A self-review limitation is a multi-instance question, not a prompt-content question. Don't cross layers.

---

Print this. Read it the morning of the exam.
