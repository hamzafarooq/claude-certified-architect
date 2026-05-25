# Contributing

Contributions that improve this resource for people studying the Claude Certified Architect exam are welcome.

## What to contribute

**Questions** — New practice questions following the same scenario-based format. Each question needs:
- A scenario label (e.g. "Multi-Agent Research System")
- Question text
- Four options (A–D)
- The correct answer index (0–3)
- An explanation of why the correct answer is right and why the distractors fail

Only submit questions you're allowed to share publicly. Don't reproduce verbatim exam questions under NDA.

**Explanations** — If an existing explanation is unclear or incomplete, open a PR with the improved version.

**Cheat sheet additions** — New mnemonics, trap patterns, or concept clarifications that would help someone under exam pressure.

## How to add a question

In `practice-exam.html`, find the relevant domain's `qs` array and add an object following this shape:

```js
{
  scenario: "Multi-Agent Research System",
  q: "Your question text here. Use <code>inline code</code> where relevant.",
  opts: [
    "Option A text",
    "Option B text",
    "Option C text",
    "Option D text"
  ],
  correct: 1,  // 0-indexed
  exp: "<strong>Why B is correct.</strong> Explanation here. Why A, C, D are wrong."
}
```

## PR checklist

- [ ] Question has a scenario label matching one of the five existing scenarios
- [ ] All four options are plausible (good distractors, not obviously wrong)
- [ ] Explanation addresses both the correct answer AND why each distractor fails
- [ ] No verbatim reproduction of NDA-protected exam content
