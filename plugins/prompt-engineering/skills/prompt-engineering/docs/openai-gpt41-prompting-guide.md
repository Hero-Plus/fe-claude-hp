# OpenAI GPT-4.1 Prompting Guide

**Source:** https://cookbook.openai.com/examples/gpt4-1_prompting_guide
**Also:** https://github.com/openai/openai-cookbook/blob/main/examples/gpt4-1_prompting_guide.ipynb
**Fetched:** 2026-03-03

GPT-4.1 prompting guide from OpenAI. While written for GPT-4.1, many findings about agentic prompting, format preferences, and instruction following are model-agnostic.

---

## 1. Agentic Reminders: +20% SWE-bench

The single most impactful finding: three simple instructions in the system prompt increased internal SWE-bench Verified scores by **close to 20%**, achieving a **55% pass rate** (state-of-the-art for non-reasoning models).

### The Three Reminders

**1. Persistence** — Prevent premature yielding:

> "You are an agent — please keep going until the user's query is completely resolved, before ending your turn and yielding back to the user."

**2. Tool-calling** — Use the tools API field, not manual injection:

> "Use tools field to pass tools rather than manually injecting tool descriptions into prompts." Using API-parsed tool descriptions vs. manual injection yielded a **2% increase** in SWE-bench pass rate.

**3. Planning** — Induce explicit step-by-step planning:

> Inducing explicit planning increased SWE-bench Verified pass rates by **4%**.

> "These three instructions transform the model from a chatbot-like state into a much more 'eager' agent."

---

## 2. XML > JSON for Structured Context

| Format                              | Performance                | Notes                                                                 |
| ----------------------------------- | -------------------------- | --------------------------------------------------------------------- |
| **XML**                             | High                       | "Convenient to precisely wrap sections, add metadata, enable nesting" |
| **Pseudo-XML** (ID\|TITLE\|CONTENT) | High                       | Lee et al. format performed well                                      |
| **JSON**                            | Poor                       | "Highly structured but verbose; requires character escaping overhead" |
| **Markdown**                        | Recommended starting point | Use headers, inline/block code, lists                                 |

Example XML structure:

```xml
<doc id='1' title='Name'>Content</doc>
```

---

## 3. Instruction Following

GPT-4.1 follows instructions **"more literally than predecessors"**, requiring explicit specification rather than implicit intent inference.

> "The model is highly steerable and responsive to well-specified prompts."

### Recommended Workflow

1. Start with "Response Rules" or high-level instructions
2. Add specific sections (e.g., "Sample Phrases") for behavior modification
3. Include ordered step lists for workflows
4. Check for conflicting/underspecified instructions (instructions closer to prompt end take precedence)
5. Add examples demonstrating desired behavior

### Common Failure Modes

- Instructing absolute compliance can cause **tool hallucination**; mitigate with "if you lack information, ask the user"
- Sample phrases risk **verbatim repetition**; instruct variation
- Without specificity, models produce **excessive prose**; provide explicit instructions and examples

---

## 4. Long Context (1M Token Window)

### Instruction Placement

> "Place instructions at both beginning AND end of provided context for better performance."

If single placement: **above context outperforms below**.

### Context Reliance Tuning

- For external-only knowledge: "Only use provided External Context. If unknowable, respond 'I don't have the information needed.'"
- For blended: "By default use external context; use internal knowledge if needed for basic connections."

---

## 5. Recommended Prompt Structure

```
# Role and Objective

# Instructions
## Sub-categories for detailed instructions

# Reasoning Steps

# Output Format

# Examples
## Example 1

# Context

# Final instructions and step-by-step thinking prompt
```

---

## 6. Tool Use Best Practices

- Use the tools field exclusively (avoid manual injection into prompts)
- Clear tool names indicating purpose
- Detailed descriptions in the description field
- Comprehensive parameter naming and descriptions
- For complex tools: create `# Examples` section in system prompt rather than cluttering the description field
- The model has "substantially improved diff capabilities" and handles various diff formats with clear instructions

---

## 7. Chain of Thought (Non-Reasoning Model)

GPT-4.1 is **not a reasoning model** but benefits from induced step-by-step planning.

### Basic CoT Instruction:

> "First, think carefully step by step about what documents are needed. Then, print out TITLE and ID of each document. Then, format the IDs into a list."

### Advanced CoT Pattern (Query/Context Analysis):

1. **Query Analysis:** Break down intent; use context for clarification
2. **Context Analysis:** Select potentially relevant documents; optimize for recall; rate relevance (high/medium/low/none)
3. **Synthesis:** Summarize most relevant documents with ratings of medium or higher

---

## 8. Key Quantitative Findings

| Technique                                                | Impact                                              |
| -------------------------------------------------------- | --------------------------------------------------- |
| Three agentic reminders (persistence + tools + planning) | **+20% SWE-bench Verified**                         |
| Explicit planning prompt                                 | **+4% SWE-bench Verified**                          |
| API tool field vs. manual injection                      | **+2% SWE-bench Verified**                          |
| Total agentic harness                                    | **55% SWE-bench Verified** (SOTA for non-reasoning) |
| Queries at end of long context                           | **Up to 30% quality improvement**                   |

---

## 9. Additional Caveats

- Rare resistance to very long, repetitive outputs (hundreds of one-by-one analyses)
- Isolated cases of parallel tool call incorrectness; consider `parallel_tool_calls=false` if issues observed
- All guidance is empirical; LLMs are nondeterministic; build evals and iterate frequently

---

## 10. Cross-Model Applicability

Several findings generalize beyond GPT-4.1:

1. **Agentic persistence reminders** — applicable to any model used as an agent
2. **XML > JSON for context** — aligns with Anthropic's own XML tag recommendations
3. **Literal instruction following** — Claude 4.x models show the same pattern
4. **Instructions at both ends of long context** — confirmed across multiple model families
5. **Planning prompts improve agentic performance** — universal finding
6. **Tool API field > manual injection** — applies to any model with native tool support
