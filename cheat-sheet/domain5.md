# Domain 5 — Context Management & Reliability: Cheatsheet

**Exam weight: 15% (smallest, but cascades into Domains 1, 2, and 4). Pass cut: 720/1000.**
**Appears primarily in: Customer Support Resolution Agent, Multi-Agent Research System, Structured Data Extraction, Codebase Exploration scenarios.**

---

## The Three Global Rules That Decide Most Questions

1. **Structure beats summarisation.** Persistent fact blocks, structured errors, claim-source bundles — anything you need to survive compression must live in structured form, never in summarised prose. The moment a high-value fact becomes free-text, it is at risk of being lost in summarisation, attention dilution, or downstream synthesis.
2. **Explicit signals beat inferred signals.** Explicit human request → escalate. Explicit access failure type → distinguish from empty result. Explicit publication date → distinguish trend from contradiction. Answers that rely on the model **inferring** intent, failure cause, or recency are almost always wrong.
3. **Calibrate, don't trust raw self-reports.** Raw model confidence, sentiment, "the model seems sure" — unreliable. Calibrated confidence against a labelled validation set, stratified by document type and field — reliable. This boundary is tested in 5.2 and 5.5.

---

## TS 5.1 — Context Preservation

**The headline rule:** Summarisation is lossy compression that disproportionately destroys high-value tokens.

**What summarisation compresses away first:**

| Token type | Example loss |
|---|---|
| Numerical values | "$247.83 refund" → "a refund" |
| Dates | "March 3rd" → "recently" |
| Order/case IDs | "#8891" → omitted entirely |
| Percentages / quantities | "32% increase" → "an increase" |
| Customer-stated expectations | "by Friday" → vanished |

**The fix — persistent "case facts" block:**

Extract transactional facts into a structured block. Inject **verbatim** into every prompt. Summarisation never touches it. Conversation history above and below can be compressed; the facts block is sacred.

```
<case_facts>
- Customer ID: 4471
- Order: #8891, placed 2026-03-03
- Refund requested: $247.83
- Customer deadline: by Friday 2026-05-22
- Channel preference: email
</case_facts>
```

**Lost in the middle — structural attention bias:**

Transformer attention is empirically biased toward the **beginning and end** of long inputs. Information buried in the middle is measurably less likely to be retrieved correctly. This is not fixable with a better model — it is structural.

| Mitigation | Why |
|---|---|
| Place key findings summaries at the **beginning** | Beginning gets reliable attention |
| Use **explicit section headers** (`## Findings`, `## Sources`) | Structural anchors aid navigation |
| Surface relevant slice of long tool results at the top, full payload below | Critical content escapes the middle |

**Tool result trimming — pre-context filtering:**

A 40-field order lookup when 5 fields are needed = 35 fields of noise. Trim at the integration layer, **before** the result hits the context window. Untrimmed results accumulate across turns — turn 8 still carries the noise from turns 1–7.

Define a per-tool **response shape** with only the fields downstream reasoning needs. Treat tool responses like API contracts, not data dumps.

**Full history requirements — stateless API:**

Claude API calls are stateless. Every request must include the **complete** conversation history. Drop earlier messages → break coherence. Use prompt caching on the stable prefix (system prompt + tools + early conversation) to make this economical.

**Upstream agent optimisation — structured output across agent boundaries:**

In multi-agent systems, the cheapest place to fix downstream context bloat is **upstream**. Modify subagents to return:

- Key facts
- Citations
- Relevance scores
- Confidence values

**Not** verbose prose, **not** reasoning chains, **not** narrative. Reasoning happens inside the subagent's context; only distilled conclusions cross the boundary.

**Trap distractors:**
- "Apply progressive summarisation uniformly across the conversation" — destroys transactional spine
- "Drop the oldest turns to stay under the limit" — breaks coherence; API is stateless
- "Trust the summarisation prompt to preserve important facts" — it can't; that's the trap
- "Pass full tool responses to maintain fidelity" — accumulates noise across turns
- "Return verbose reasoning from subagents so the orchestrator has full context" — fabrication of context bloat

---

## TS 5.2 — Escalation and Ambiguity Resolution

**The headline rule:** Three triggers warrant escalation. Two superficially plausible triggers do not.

**Three valid escalation triggers:**

| Trigger | Required action |
|---|---|
| Customer **explicitly** requests a human | Escalate immediately. **Do not attempt resolution first.** |
| Policy exception or gap (request outside documented policy) | Escalate — the agent cannot invent policy |
| Inability to make **meaningful** progress | Escalate after legitimate avenues tried |

**Two unreliable triggers — wrong answers:**

| Bad trigger | Why it fails |
|---|---|
| Sentiment-based (customer is frustrated) | Frustration does not correlate with case complexity |
| Self-reported model confidence | Model is often incorrectly confident on hard cases and uncertain on easy ones |

**The frustration nuance — heavily tested:**

| Situation | Correct action |
|---|---|
| Straightforward issue + frustrated customer (no human request) | Acknowledge frustration, **offer resolution**, do not escalate yet |
| Customer **reiterates** preference for a human after you offer help | Now escalate |
| Customer **explicitly** says "I want a human" up front | Escalate immediately — even if the issue looks trivially resolvable |

**Memorise the distinction:** Explicit human request = immediate escalation. Frustration alone = offer help first.

**Ambiguous customer matching:**

Search returns multiple matches for "John Smith." The correct move is **always**:

> Ask for an additional identifier — email, phone, order number, postcode.

**Wrong heuristics — exam distractors:**
- Pick the most recent account
- Pick the most active account
- Pick the one whose order details "match" the description
- Pick based on geographical proximity

The cost of asking is one extra turn. The cost of acting on the wrong account is enormous (wrong refund, wrong PII exposure, wrong cancellation). **Never resolve ambiguity with heuristics when the customer can disambiguate it in one turn.**

**Trap distractors:**
- "Resolve the duplicate charge first to demonstrate value, then escalate" — overrides explicit human request
- "Ask the customer to confirm they really want a human" — patronising; explicit means explicit
- "Escalate because the customer sounds frustrated" — sentiment is unreliable
- "Auto-escalate when model confidence drops below 0.6" — uncalibrated self-reported confidence
- "Pick the most recently active account that matches the search" — heuristic on ambiguity

---

## TS 5.3 — Error Propagation

**The headline rule:** How a failure is reported matters as much as that it happened. Silent suppression and workflow termination are both anti-patterns.

**Structured error context — four required fields:**

| Field | Purpose |
|---|---|
| **Failure type** | `transient` / `validation` / `business` / `permission` |
| **What was attempted** | Specific query, parameters, endpoint, request shape |
| **Partial results** | Anything gathered before the failure |
| **Potential alternatives** | Suggested retry, fallback path, narrower query, cached endpoint |

Together these give the model enough context to **recover**. Unstructured errors force the model to give up or hallucinate.

**The two anti-patterns:**

| Anti-pattern | Why it fails |
|---|---|
| **Silent suppression** — return `{"results": [], "status": "ok"}` on tool failure | Orchestrator believes no data exists. **Zero recovery path** — this is the worst answer in any 5.3 question. |
| **Workflow termination** — kill the entire pipeline on a single tool failure | Throws away partial results from other subagents that succeeded |

**Access failure vs valid empty result — exam favourite:**

Two superficially identical responses, opposite correct behaviours.

| Type | Meaning | Correct response |
|---|---|---|
| **Access failure** | Tool could not reach the data source (network, auth, timeout) | **Consider retry** or alternative path |
| **Valid empty result** | Tool reached the source, ran the query, found no matches | **No retry.** The empty result *is the answer.* |

**The structured error type distinguishes these.** Treat all empty results the same and you either retry valid-empties forever or accept access failures as truth. Both wrong.

**Coverage annotations — synthesis output discipline:**

> "Section on geothermal energy is limited due to unavailable journal access (3 of 7 expected sources unreachable)."

Better than silently omitting the section. The consumer of the synthesis can now decide whether to act on partial data or wait for the gap to be filled. Silent omission destroys this signal.

**Trap distractors:**
- "Return empty results with status=ok so the orchestrator can continue gracefully" — silent suppression, worst option
- "Kill the entire pipeline on a single subagent timeout" — workflow termination, loses partial results
- "Retry the tool call indefinitely until it succeeds" — wastes budget on permanent failures; doesn't distinguish failure types
- "Always retry empty results in case data appears" — conflates valid empty with access failure
- "Catch the exception silently and log it for later review" — model has no signal during execution

---

## TS 5.4 — Codebase Exploration

**The headline rule:** Extended agentic sessions degrade. Verbose discovery output dilutes early findings; the model reverts to generic "typical patterns" instead of session-specific facts.

**Two symptoms of context degradation:**

1. Model references **"typical patterns"** instead of the specific classes/functions/files it discovered earlier — generic knowledge crowds out session-specific findings.
2. Context fills with **verbose discovery output** (file dumps, search results) and earlier conclusions get buried.

Both symptoms have the same root: high-signal early findings get diluted by high-volume later exploration.

**Four mitigation strategies — match the mitigation to the symptom:**

| Strategy | Best for |
|---|---|
| **Scratchpad files** — write key findings to a file, reference for subsequent questions | Findings you'll need later; survives compaction and crashes |
| **Subagent delegation** — spawn a subagent for a specific investigation; coordinator receives distilled result only | Parallel deep dives; keeps coordinator context focused on coordination |
| **Summary injection** — summarise findings from one phase before spawning the next phase's subagents | Phase boundaries in multi-phase workflows |
| **`/compact`** — reduce context usage when verbose output fills the window | Recovery valve when nearing the limit; trade-off: loses fidelity |

**Crash recovery — manifest pattern:**

Each agent exports structured state to a known file location (a **manifest**) at meaningful checkpoints. On resume, the coordinator loads the manifest and injects it into the resumed agent's prompts.

The **manifest is the source of truth, not the conversation history.** Robust to:
- Process crashes
- Context window exhaustion mid-task
- Operator interruptions
- Long-running jobs spanning multiple sessions

Without a manifest, a crash means starting over. With a manifest, a crash means resuming from the last checkpoint.

**Trap distractors:**
- "Increase model temperature to encourage more specific responses" — temperature affects sampling, not retention of specific facts
- "Re-run the entire exploration from scratch with a fresh context" — wastes work; hits the same problem at the same turn count
- "Use a larger context window so degradation doesn't occur" — attention dilution isn't a token-count problem (mirrors Domain 4.6 attention rule)
- "Trust the model to remember earlier findings if asked directly" — that *is* the failure mode

---

## TS 5.5 — Human Review and Confidence Calibration

**The headline rule:** Aggregate metrics hide stratum-level failures. Calibration requires ground truth, not intuition.

**The aggregate metrics trap — exam favourite:**

> "Our extraction system achieves 97% accuracy. We can automate."

The trap: 97% **overall** accuracy can hide a 40% error rate on a specific document type that is rare in the validation set. If that document type is the high-value commercial contracts, your automation is dangerous despite the headline number.

**Discipline:** Always validate accuracy by document type **and** by field segment before automating. A single aggregate number is never sufficient evidence to remove humans from the loop.

**Stratified random sampling — ongoing audit:**

Even after calibration and deployment, continue sampling **high-confidence** extractions for ongoing verification.

| Reason | Why it matters |
|---|---|
| Novel error patterns emerge over time | New document templates, edge cases, input distribution drift |
| Failures slip through silently if not sampled | By definition, they don't trigger existing review thresholds |
| Stratified across types, fields, confidence bands | Ensures coverage of all segments, not just the loud failures |

The discipline: a small, ongoing audit of "the cases we think are fine" is what catches the failures you didn't anticipate.

**Field-level confidence calibration — the full pattern:**

| Step | Detail |
|---|---|
| 1. **Per-field confidence** (not per-document) | Fields within a document vary wildly in difficulty |
| 2. **Calibrate thresholds on a labelled validation set** | Ground truth. Raw model confidence values are not directly meaningful — "0.85" might mean 70% correct or 95% correct depending on field and document type |
| 3. **Route low-confidence fields to human review** | The rest auto-process |
| 4. **Prioritise scarce reviewer capacity on highest-uncertainty items** | Reviewer time is the limiting resource |

**Connection to 5.2:** This is what makes confidence-based decisions reliable. Naive self-reported confidence is unreliable. **Calibrated, field-level, validation-set-grounded confidence is reliable.** The exam tests this distinction.

**Trap distractors:**
- "97% accuracy meets industry standard, automate confidently" — aggregate metric, no stratum check
- "Sample only low-confidence extractions for review" — misses novel error patterns in the high-confidence band
- "Set confidence threshold to 0.9 to ensure quality" — uncalibrated number; meaningless without ground truth
- "Use document-level confidence to decide review routing" — field variance within documents is the point
- "Human review is always required regardless of metrics" — too absolute; calibrated automation is valid

---

## TS 5.6 — Information Provenance

**The headline rule:** Attribution must travel with every claim as structured data. Conflicts are surfaced, not resolved. Dates distinguish trends from contradictions.

**Structured claim-source mappings — five required fields per finding:**

| Field | Purpose |
|---|---|
| **Claim** | The assertion itself |
| **Source URL** | Direct link |
| **Document name** | Human-readable identifier |
| **Relevant excerpt** | The actual passage supporting the claim |
| **Publication date** | Temporal context |

This bundle travels with the claim through every downstream agent. The synthesis agent **preserves and merges** these mappings — it does not flatten them into prose.

**Without this structure, attribution dies the moment any agent summarises.** The final report says "studies show X" with no path back to which study, by whom, when. That is not a research output; that is a plausible-sounding hallucination risk.

**Conflict handling:**

Two credible sources report different numbers for the same statistic. The wrong moves and the right move:

| Wrong | Why it fails |
|---|---|
| Arbitrarily pick one | Erases the disagreement |
| Average the values | Fabricates a number that neither source reported — worst kind of synthesis hallucination |
| Pick the more recent without flagging | Loses the signal that disagreement existed |
| Pick the source with the "better" reputation silently | Erases information the consumer needs |

| Right |
|---|
| **Annotate with both values and full source attribution. Let the consumer decide.** |

```
Renewable energy share of global generation in 2024:
- 30% (IEA, World Energy Outlook 2025, published 2025-10)
- 32% (IRENA, Renewable Capacity Statistics 2025, published 2025-03)
```

**Temporal awareness — date-aware reconciliation:**

Many apparent contradictions are actually different reporting periods. A 28% figure from 2022 and a 32% figure from 2024 are not conflicting — they show change over time.

| Required in structured output | Why |
|---|---|
| Publication date | Reframes "conflict" as "trend" |
| Data collection date | Distinguishes when data was gathered vs when it was published |

Same logic for: market share, headcount, regulatory status, version numbers, anything that changes.

**Content-appropriate rendering — match format to content:**

| Content type | Rendering |
|---|---|
| Financial data | **Tables** (numbers in prose are hard to scan and compare) |
| News / narrative | **Prose** (forcing bullets loses causality) |
| Technical findings | **Structured lists** (multiple discrete items benefit from visual separation) |

**Flattening everything into one uniform format loses information.** The exam may ask which format suits a given content type — match the content, do not impose a house style.

**Trap distractors:**
- "Select the more conservative figure when sources conflict" — arbitrary heuristic, erases disagreement
- "Average the conflicting values to balance the sources" — fabricates a number neither source reported
- "Omit the statistic when sources disagree" — destroys information
- "Render everything in a uniform bullet-point format for consistency" — content-format mismatch
- "Cite sources in a separate appendix rather than inline with claims" — attribution dies during synthesis

---

## Distractor-Recognition Cribsheet

When you see these answer options, **reject them**:

- "Apply progressive summarisation across the conversation" (destroys transactional spine — use a persistent case-facts block)
- "Drop the oldest messages to stay under the limit" (breaks coherence; API is stateless)
- "Pass full tool responses to maintain fidelity" (accumulates noise across turns)
- "Return verbose subagent reasoning to the orchestrator" (downstream context bloat)
- "Place the summary at the end of the document" (lost-in-the-middle risk — put it at the beginning)
- "Resolve the issue first, then escalate if the customer is still unhappy" (when customer explicitly requested a human — overrides explicit request)
- "Escalate because the customer sounds frustrated" (sentiment ≠ complexity)
- "Auto-escalate when model confidence drops below threshold X" (uncalibrated self-reported confidence)
- "Pick the most recently active account when multiple match" (heuristic on ambiguity — ask for an identifier)
- "Return empty results with status=ok on tool failure" (silent suppression — worst pattern)
- "Kill the entire pipeline on a single subagent failure" (workflow termination — loses partial results)
- "Retry empty results in case data appears later" (conflates access failure with valid empty)
- "Increase temperature to fix context degradation" (temperature isn't the lever)
- "Use a larger context window to prevent attention dilution" (attention isn't a token-count problem — same as Domain 4.6)
- "Re-run from scratch with a fresh context window" (hits the same problem at the same turn count — use scratchpad + subagents)
- "97% aggregate accuracy is sufficient to automate" (mask stratum-level failures)
- "Sample only low-confidence extractions for review" (misses novel patterns in the high-confidence band)
- "Set confidence threshold to 0.9 without calibration" (meaningless without ground truth)
- "Use document-level confidence to route review" (field variance within documents is the point)
- "Select the more conservative figure when sources conflict" (arbitrary; erases disagreement)
- "Average conflicting figures to balance the sources" (fabricates a number neither source reported)
- "Omit the statistic when sources disagree" (destroys information)
- "Render everything in one uniform format for consistency" (content-format mismatch)

When you see these, **lean toward them**:

- "Extract transactional facts into a persistent case-facts block injected verbatim into every prompt" (anti-summarisation)
- "Place key findings summaries at the beginning with explicit section headers" (counters lost-in-the-middle)
- "Trim tool results to relevant fields before appending to context" (prevents noise accumulation)
- "Include the complete conversation history in every API request" (stateless API)
- "Modify upstream subagents to return structured key facts and citations, not verbose reasoning" (downstream optimisation)
- "Escalate immediately when the customer explicitly requests a human, without attempting resolution first" (explicit signal honoured)
- "Acknowledge frustration, offer resolution; escalate only if the customer reiterates the human request" (frustration nuance)
- "Ask the customer for an additional identifier (email, phone, order number) to disambiguate" (no heuristics on ambiguity)
- "Return a structured error: failure type, query attempted, partial results, suggested alternative" (recovery-capable error)
- "Preserve partial results from succeeded subagents when one fails; annotate the synthesis with a coverage note" (graceful degradation)
- "Distinguish access failure (consider retry) from valid empty result (the answer is 'none')" (exam favourite)
- "Write key findings to a scratchpad file and reference it for subsequent questions" (context-degradation fix)
- "Delegate specific investigations to subagents; coordinator receives distilled result only" (context isolation)
- "Inject phase summaries before spawning next-phase subagents" (continuity without full history)
- "Export structured state to a manifest file at checkpoints for crash recovery" (resume from manifest, not history)
- "Validate accuracy by document type and field segment before automating" (defeats aggregate-metrics trap)
- "Continue stratified sampling of high-confidence extractions post-deployment to catch novel error patterns" (ongoing audit)
- "Output per-field confidence, calibrate thresholds on a labelled validation set, route low-confidence to human review" (full calibration pattern)
- "Prioritise scarce reviewer capacity on highest-uncertainty items" (resource allocation)
- "Attach structured claim-source mappings (claim + URL + doc + excerpt + date) to every finding" (provenance survives synthesis)
- "Surface conflicting sources with both values and full attribution; let the consumer decide" (conflict handling)
- "Include publication and data collection dates in structured outputs to distinguish trends from contradictions" (temporal awareness)
- "Match rendering to content: tables for financial data, prose for narrative, structured lists for technical findings" (content-format match)

---

## One-Line Mnemonics

- **Summarisation compresses numbers, dates, IDs, percentages, expectations first.** Extract them into a persistent block that summarisation never touches.
- **Lost in the middle is structural, not a model bug.** Put summaries at the beginning, use section headers, never bury the lede.
- **Trim tool results before they hit context.** Noise accumulates across turns.
- **The API is stateless.** You manage history. Drop messages → break coherence.
- **Upstream subagents return structured key facts.** Not narrative. Not reasoning chains.
- **Three valid escalation triggers:** explicit human request, policy gap, no meaningful progress.
- **Two unreliable triggers:** sentiment, self-reported confidence.
- **Explicit "I want a human" = escalate now, do not investigate.**
- **Frustration alone = offer help first; escalate if reiterated.**
- **Ambiguous customer match = ask for an identifier, never heuristic.**
- **Structured error = type + attempt + partial results + alternatives.**
- **Silent suppression is the worst pattern.** Empty results with status=ok = no recovery path.
- **Workflow termination throws away partial results.** Graceful degradation preserves them.
- **Access failure ≠ valid empty.** Retry only the former.
- **Coverage annotations beat silent omission.**
- **Long sessions degrade: scratchpad, subagents, summary injection, `/compact`.**
- **Manifest files are the source of truth across crashes.** Not conversation history.
- **97% accuracy can hide 40% on a stratum.** Validate by document type AND field segment.
- **Stratified sampling continues post-deployment.** Sample high-confidence cases to find novel errors.
- **Per-field confidence, calibrated on a labelled validation set.** Raw 0.9 means nothing without ground truth.
- **Every finding carries: claim + URL + doc + excerpt + date.** Attribution dies otherwise.
- **Surface conflicts with both values and attribution. Never average. Never silently pick.**
- **Dates distinguish trends from contradictions.** 28% (2022) vs 32% (2024) is not a conflict.
- **Match rendering to content.** Tables for numbers, prose for narrative, lists for technical.

---

## Mechanism-to-Failure Decision Tree

When a question describes a failure, ask in order:

1. **Conversation history hitting the limit and you fear losing transactional facts?** → Persistent case-facts block injected verbatim, summarise surrounding turns only
2. **Key findings buried in long inputs and being missed?** → Place summaries at the beginning + explicit section headers
3. **Tool result has 40 fields when you need 5?** → Trim before appending to context
4. **Downstream agents drowning in upstream verbosity?** → Modify upstream agents to return structured key facts only
5. **Customer explicitly asks for a human?** → Escalate immediately, no investigation
6. **Customer is frustrated but issue is simple and no human requested?** → Acknowledge, offer resolution, escalate only if reiterated
7. **Multiple customers match the search query?** → Ask for an additional identifier (email/phone/order ID)
8. **Tool call failed and you're deciding how to report it?** → Structured error: type + attempt + partial results + alternatives
9. **Tool returned an empty array — should you retry?** → Distinguish access failure (retry) from valid empty (do not retry; that is the answer)
10. **One subagent fails in a pipeline of four?** → Preserve the three successes, annotate the gap, do not terminate the pipeline
11. **Long codebase exploration session: model referencing "typical patterns" instead of specific findings?** → Scratchpad file with key findings, referenced for subsequent questions
12. **Need parallel deep dives without bloating the coordinator's context?** → Subagent delegation, coordinator receives distilled output only
13. **Long-running job risks a mid-task crash?** → Manifest file checkpoint pattern, resume from manifest
14. **97% accuracy claimed and team wants to automate?** → Demand stratified validation by document type and field segment first
15. **Need to catch novel error patterns after deployment?** → Stratified sampling of high-confidence extractions, ongoing
16. **Routing extractions to human review?** → Per-field confidence, calibrated on labelled validation set, route low-confidence
17. **Two credible sources report different numbers?** → Annotate both with full source attribution; let the consumer decide
18. **Same statistic with different values from different years?** → Include publication dates; this is a trend, not a contradiction
19. **Rendering the synthesis output?** → Match format to content type (table/prose/list)

---

## Layer-Awareness Rule (carry-forward from teaching)

Many wrong answers come from reaching for the right idea at the wrong layer:

| Layer | What lives here |
|---|---|
| **Prompt content** | Case-facts block, structured section headers |
| **History management** | Summarisation strategy, what to drop, what to keep verbatim |
| **Tool integration** | Response-shape trimming, structured error envelopes |
| **Agent boundaries** | Structured key facts and citations crossing agent boundaries — not narrative |
| **Escalation policy** | Three valid triggers, two unreliable triggers, the frustration nuance |
| **Error semantics** | Failure type taxonomy, access failure vs valid empty, coverage annotations |
| **Session memory** | Scratchpad files, subagent delegation, summary injection, `/compact`, manifests |
| **Validation discipline** | Stratified sampling, per-field confidence, ground-truth calibration |
| **Provenance** | Claim-source bundles, conflict annotation, temporal metadata, content-aware rendering |

**When you feel the pull toward a fix, ask which layer the question is testing.**
A summarisation problem is a history-management question (case-facts block), not a prompt-content question ("tell the model to remember the refund amount"). An ambiguous-customer problem is an escalation-policy question (ask for identifier), not a tool-integration question. A novel-error-pattern problem is a validation-discipline question (stratified sampling of high-confidence cases), not a confidence-threshold question. Don't cross layers.

---

## Cross-Domain Connections

Domain 5's smallest weight is misleading — these concepts surface across other scenarios:

- **TS 5.1 → Domain 1** (Customer Support Resolution): persistent case-facts block is the canonical fix for multi-turn refund/support cases
- **TS 5.2 → Domain 1**: escalation logic owns half of the support-agent question space
- **TS 5.3 → Domain 2** (Multi-Agent Research): structured error propagation is what keeps research pipelines recoverable
- **TS 5.4 → Domain 3** (Codebase Exploration): scratchpad/subagent/manifest patterns are the codebase-agent toolkit
- **TS 5.5 → Domain 4** (Structured Data Extraction): calibration + stratified sampling overlap with Domain 4.6 confidence-based routing
- **TS 5.6 → Domain 2**: provenance is the difference between a research synthesis and a plausible-sounding hallucination

The **calibrated-confidence** pattern (TS 5.5) and the **multi-instance review** pattern (Domain 4.6) are the same architectural idea expressed at different layers. Recognise the family resemblance.

---

Print this. Read it the morning of the exam.
