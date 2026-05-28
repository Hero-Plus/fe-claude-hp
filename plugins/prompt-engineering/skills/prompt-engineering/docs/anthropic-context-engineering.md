# Anthropic: Effective Context Engineering for AI Agents

**Source:** https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
**Published:** September 29, 2025
**Authors:** Prithvi Rajasekaran, Ethan Dixon, Carly Ryan, Jeremy Hadfield (Anthropic Applied AI)
**Fetched:** 2026-03-03

---

## Core Definition

> **Context Engineering**: "the set of strategies for curating and maintaining the optimal set of tokens (information) during LLM inference, including all the other information that may land there outside of the prompts."

Context engineering evolves beyond prompt engineering. Prompt engineering = finding the right words. Context engineering = "what configuration of context is most likely to generate our model's desired behavior?"

---

## 1. Context Rot

> "Research on needle-in-a-haystack benchmarking uncovered 'context rot': as context windows grow, models' ability to accurately recall information decreases."

**Technical cause:** Transformers create n-squared pairwise token relationships. As context length increases:

- Attention is stretched thinner across more relationships
- Models have less training experience with longer sequences (training bias toward shorter texts)
- Performance degrades as a **gradient, not a cliff** — models remain capable but show reduced precision for information retrieval and long-range reasoning

**Implication:** Context is a finite resource with diminishing marginal returns. Adding more tokens past a point makes the model worse, not better.

---

## 2. Anatomy of Effective Context

### System Prompts: The Right Altitude

- **Too Brittle:** Hardcoded complex logic creates maintenance burden
- **Too Vague:** Insufficient concrete signals or false assumptions about shared context
- **Optimal:** Specific enough to guide behavior, flexible enough for strong heuristics

> "Striving for the minimal set of information that fully outlines your expected behavior."

Recommendation: Organize into distinct sections using XML tags or Markdown headers.

### Tools

Requirements for effective agent tools:

- Self-contained, robust to error, extremely clear purpose
- Descriptive, unambiguous input parameters
- Minimal overlap in functionality
- "If humans can't definitively choose which tool applies, neither can AI agents"

### Examples (Few-Shot)

> "Rather than exhaustive edge case lists, curate diverse, canonical examples that effectively portray the expected behavior of the agent. For an LLM, examples are the 'pictures' worth a thousand words."

---

## 3. Just-In-Time Context

Modern agents use "just in time" context rather than pre-computing all data:

- Maintain lightweight identifiers (file paths, queries, links)
- Dynamically load data at runtime using tools
- **Mirrors human cognition:** "We don't memorize corpuses but use indexing systems"

**Example:** Claude Code performs complex analysis over large databases using targeted queries and Bash commands (head, tail) without loading full data objects into context.

### Progressive Disclosure

Agents incrementally discover context through exploration:

- File sizes suggest complexity
- Naming conventions hint at purpose
- Timestamps indicate relevance
- This maintains focused working memory rather than exhaustive information

### Trade-offs

Runtime exploration is slower than pre-computed retrieval. Without proper tool guidance, agents waste context on misused tools and dead-ends.

### Hybrid Approach (Claude Code's Model)

- CLAUDE.md files preloaded into context (static)
- Glob and grep primitives enable just-in-time file retrieval (dynamic)
- Bypasses stale indexing and complex syntax issues

---

## 4. Long-Horizon Task Techniques

For tasks spanning hours with token counts exceeding context windows:

### 4a. Compaction

> Practice of summarizing conversations at context limits, reinitiating with compressed summary.

Process in Claude Code:

1. Pass message history to model for summarization
2. Preserve architectural decisions, unresolved bugs, implementation details
3. Discard redundant tool outputs
4. Continue with compressed context plus five most recent files

> "Balance between recall (capturing all relevant information) and precision (eliminating superfluous content)."

**Low-hanging optimization:** Clearing tool results — "once a tool has been called deep in the message history, why would the agent need to see the raw result again?"

### 4b. Structured Note-Taking

Agents regularly write notes persisted **outside** the context window, pulled back when needed.

Benefits:

- Persistent memory with minimal overhead
- Track progress across complex tasks
- Maintain critical context and dependencies

**Example:** Claude playing Pokemon maintains precise tallies across thousands of steps: "for the last 1,234 steps I've been training my Pokemon in Route 1, Pikachu has gained 8 levels toward the target of 10."

### 4c. Sub-Agent Architectures

Specialized sub-agents handle focused tasks with clean context windows:

- Main agent coordinates high-level plans
- Subagents perform deep work and return condensed summaries (1,000-2,000 tokens vs. tens of thousands explored)
- Achieves separation of concerns: detailed search context isolated within sub-agents

---

## 5. Strategic Selection Guide

| Technique   | Best For                                                       |
| ----------- | -------------------------------------------------------------- |
| Compaction  | Extensive back-and-forth tasks                                 |
| Note-Taking | Iterative development with clear milestones                    |
| Multi-Agent | Complex research/analysis benefiting from parallel exploration |

---

## 6. Key Recommendations

1. **Treat context as precious, finite resource** with diminishing marginal returns
2. **Find smallest set of high-signal tokens** maximizing desired outcomes
3. **Start with minimal prompts** on best available models, then add based on failure modes
4. **Employ hybrid strategies** — some data upfront, enable autonomous exploration for the rest
5. **"Do the simplest thing that works"** given rapid progress in model capabilities

---

## 7. Takeaway for CLAUDE.md / Skill Design

The article's principles directly inform instruction file design:

- Every token in always-loaded context (CLAUDE.md) must earn its place
- Detailed procedures should be loaded on-demand (skills = just-in-time context)
- Compaction-friendly context: structure content so summarization preserves the important parts
- Tools and examples are the highest-signal context types
