# Anthropic Claude 4.x Prompting Best Practices

**Source:** https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/claude-4-best-practices
**Fetched:** 2026-03-03

Single reference for prompt engineering with Claude Opus 4.6, Sonnet 4.6, and Haiku 4.5.

---

## 1. Opus 4.6 Overtriggering and Over-Prompting

Claude Opus 4.5 and 4.6 are **more responsive to the system prompt than previous models**. Prompts designed to reduce undertriggering on tools or skills may now **overtrigger**.

> "Where you might have said 'CRITICAL: You MUST use this tool when...', you can use more normal prompting like 'Use this tool when...'."

### Specific Guidance for Dialing Back

- **Replace blanket defaults with targeted instructions.** Instead of "Default to using [tool]," use "Use [tool] when it would enhance your understanding of the problem."
- **Remove over-prompting.** "Instructions like 'If in doubt, use [tool]' will cause overtriggering."
- **Use `effort` as a fallback.** If Claude continues to be overly aggressive, use a lower `effort` setting.

### Overthinking / Excessive Thoroughness

> "Claude Opus 4.6 does significantly more upfront exploration than previous models, especially at higher effort settings."

Mitigation prompt:

> "When you're deciding how to approach a problem, choose an approach and commit to it. Avoid revisiting decisions unless you encounter new information that directly contradicts your reasoning."

### Overeagerness / Overengineering

> "Claude Opus 4.5 and Claude Opus 4.6 have a tendency to overengineer by creating extra files, adding unnecessary abstractions, or building in flexibility that wasn't requested."

Mitigation: Provide explicit "Avoid over-engineering" instructions with scoped rules for documentation, defensive coding, and abstractions.

---

## 2. Emphasis Markers Guidance

The document implicitly addresses emphasis through its overtriggering advice:

- **Old pattern (too aggressive):** `CRITICAL: You MUST use this tool when...`
- **New pattern (appropriate):** `Use this tool when...`

Emphasis markers (IMPORTANT, MUST, CRITICAL) are still valid for genuinely critical rules, but **Opus 4.6 is already highly steerable** — reserve emphasis for rules where non-compliance would be a real problem.

> "You can tune instructions by adding emphasis (e.g., 'IMPORTANT' or 'YOU MUST') to improve adherence." (from Claude Code best practices)

---

## 3. Positive Framing

> "Tell Claude what to do instead of what not to do."

| Instead of                             | Use                                                                                                                                                       |
| -------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| "Do not use markdown in your response" | "Your response should be composed of smoothly flowing prose paragraphs."                                                                                  |
| "NEVER use ellipses"                   | "Your response will be read aloud by a text-to-speech engine, so never use ellipses since the text-to-speech engine will not know how to pronounce them." |

The second example shows that when negative framing is needed, **adding the WHY makes it effective** — Claude generalizes from the explanation.

---

## 4. Explain WHY (Context Improves Performance)

> "Providing context or motivation behind your instructions, such as explaining to Claude why such behavior is important, can help Claude better understand your goals and deliver more targeted responses."

> "Claude is smart enough to generalize from the explanation."

This principle applies broadly: instead of just stating a rule, explain the consequence or purpose so Claude can apply the rule appropriately in novel situations.

---

## 5. Examples (Few-Shot Prompting)

> "Examples are one of the most reliable ways to steer Claude's output format, tone, and structure."

**Recommended count: 3-5 examples for best results.**

When adding examples, make them:

- **Relevant:** Mirror your actual use case closely.
- **Diverse:** Cover edge cases and vary enough that Claude doesn't pick up unintended patterns.
- **Structured:** Wrap examples in `<example>` tags so Claude can distinguish them from instructions.

> "You can also ask Claude to evaluate your examples for relevance and diversity, or to generate additional ones based on your initial set."

---

## 6. Be Clear and Direct (Literal Interpretation)

> "Think of Claude as a brilliant but new employee who lacks context on your norms and workflows."

Claude 4.x models follow instructions **more literally** than predecessors. If you want "above and beyond" behavior, explicitly request it:

| Less effective                  | More effective                                                                                                                                                   |
| ------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| "Create an analytics dashboard" | "Create an analytics dashboard. Include as many relevant features and interactions as possible. Go beyond the basics to create a fully-featured implementation." |
| "Can you suggest some changes?" | "Change this function to improve its performance."                                                                                                               |

> **Golden rule:** "Show your prompt to a colleague with minimal context on the task and ask them to follow it. If they'd be confused, Claude will be too."

---

## 7. XML Tags for Structure

> "XML tags help Claude parse complex prompts unambiguously, especially when your prompt mixes instructions, context, examples, and variable inputs."

Best practices:

- Use consistent, descriptive tag names across prompts.
- Nest tags when content has a natural hierarchy.
- Wrap sections in tags like `<instructions>`, `<context>`, `<input>`.

---

## 8. Long Context Prompting (20K+ tokens)

- **Put longform data at the top**, above your query, instructions, and examples.
- **Queries at the end improve response quality by up to 30%** in tests, especially with complex, multi-document inputs.
- **Ground responses in quotes**: Ask Claude to quote relevant parts of documents first before completing the task.

---

## 9. Autonomy and Safety Balance

> "Without guidance, Claude Opus 4.6 may take actions that are difficult to reverse or affect shared systems."

Recommended: Explicitly distinguish local/reversible actions (encouraged) from destructive/shared-system actions (require confirmation).

---

## 10. Subagent Orchestration

> "Claude Opus 4.6 has a strong predilection for subagents and may spawn them in situations where a simpler, direct approach would suffice."

Guidance: Use subagents for parallel work, isolated context, or independent workstreams. For simple tasks, single-file edits, or sequential operations, work directly.

---

## 11. Adaptive Thinking

Claude Opus 4.6 uses adaptive thinking by default. Key findings:

- **General instructions outperform prescriptive steps**: "A prompt like 'think thoroughly' often produces better reasoning than a hand-written step-by-step plan."
- **Multishot examples work with thinking**: Use `<thinking>` tags inside few-shot examples.
- **Self-checking is reliable**: "Before you finish, verify your answer against [test criteria]."

---

## 12. Migration Checklist (from older models to 4.6)

1. Be specific about desired behavior
2. Frame instructions with quality modifiers
3. Request features explicitly
4. Update thinking configuration to adaptive
5. Migrate away from prefilled responses
6. **Tune anti-laziness prompting** — dial back aggressive encouragement
