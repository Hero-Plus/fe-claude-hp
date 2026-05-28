---
description: Research-backed rules for creating, auditing, modifying, or designing Claude Code skills, CLAUDE.md files, rules files, hooks, subagents, or any system-prompt instruction file. Use whenever the user wants to write a new skill or extension, modify or improve an existing skill, audit a skill's triggering behavior, decide whether a skill should be created vs deferred, optimize a description for better auto-firing, or design a hook/agent that interacts with skills. Use even for adjacent framings like "what's the best solution for X" if X involves any of skill design, hook design, instruction-file authoring, or Claude Code extension architecture. Load BEFORE making decisions or edits.
user-invocable: true
---

# Prompt Engineering for Claude Code System Prompts

## Evidence Base

These rules derive from 5 official guidance docs (Anthropic, OpenAI) supported by 18 empirical studies. Optimized for thinking models (Opus 4.7 adaptive-thinking-only mode; Opus 4.6 also supported).

### Tier 1: Anthropic / Claude Code Official Guidance

- Context is a finite resource with diminishing marginal returns; bloated instruction files cause Claude to ignore actual instructions. Every token in always-loaded context must earn its place. Quantified: 68% accuracy at 500 instructions with linear decay (IFScale, 2507.11538); 20%+ cost increase from unnecessary context (AGENTS.md, 2602.11988). (Anthropic Context Engineering; Anthropic Claude Code)
- Specify only what the model won't infer — ruthlessly prune rules Claude already follows. Both over-specification (-19%) and underspecification (-23%) hurt. (Anthropic Claude 4.x; Anthropic Claude Code; quantified: Underspecification, 2505.13360)
- General instructions outperform prescriptive step-by-step plans **with thinking enabled** — model reasons by default; explicit CoT adds 35-600% latency for marginal gains. With thinking OFF / low effort, manual CoT inside `<thinking>` tags remains a valid fallback (Anthropic Opus 4.7 guidance). (Anthropic Claude 4.x; quantified: CoT Value, 2506.07142)
- Format/convention examples (2-3) remain effective for steering output structure. Reasoning shown in an example's *visible output* is counterproductive with thinking enabled (-6 to -35% on RL-trained reasoners) — the model imitates example patterns instead of deploying its RL-trained strategies. Reasoning shown inside `<thinking>` tags in an example IS fine — Claude generalizes that pattern to its own thinking blocks. Use `<thinking>` tags to shape reasoning style; never put prescriptive steps in the example's visible output. (Anthropic Opus 4.7 guidance; DeepSeek R1; From Harm to Help, 2509.23196; Over-prompting, 2509.13196)
- Persona/role assignments ("You are an expert X") provide zero improvement on *objective accuracy* (math, factual QA — confirmed across 162 personas) but DO usefully steer tone, voice, and stylistic behavior (Anthropic Opus 4.7 guidance: "Even a single sentence makes a difference."). Use a role when consistent voice matters; skip it when only correctness matters.
- Format consistency matters — Markdown-formatted prompts elicit Markdown responses. (Anthropic guidance)
- For very long instruction files, lead with the highest-leverage triggers. The "lost-in-middle" U-shape was strongest in 2023-era / non-thinking models with limited context; Opus 4.7's 1M-token context with context-awareness shows much weaker positional bias. Do NOT duplicate rules at top and bottom of concise files — duplication wastes context and Opus 4.7's literal interpretation can produce inconsistent behavior across copies. (Anthropic Claude 4.x; quantified-but-likely-attenuated: Lost in Middle, 2307.03172)
- Skill descriptions decide both whether a skill auto-fires AND its always-loaded context cost. Claude tends to *under*trigger, so descriptions should explicitly state when to use the skill, including paraphrased intents. Filesystem rules are non-negotiable: kebab-case folder, exact `SKILL.md` (case-sensitive), no `README.md` inside the skill folder. (Anthropic Skills Complete Guide, April 2026)

### Tier 2: OpenAI / Industry Guidance (Cross-Model Validated)

- Agentic reminders (persistence + tool-calling + planning) improve SWE-bench scores +20%. Include behavioral triggers for agent workflows. (OpenAI GPT-4.1)
- Modern models follow instructions more literally than predecessors — test exact wording carefully. Opus 4.7 is more literal than Opus 4.6: it will not silently generalize an instruction from one item to another, and will not infer requests you didn't make. State scope explicitly when you want a rule applied broadly. (OpenAI GPT-4.1; Anthropic Claude 4.x; Anthropic Opus 4.7 guidance)
- XML outperforms JSON for structured context; Markdown recommended for instruction files. (OpenAI GPT-4.1)

### Tier 3: Empirical Research (Unique Quantitative Findings)

These findings have no equivalent in official guidance and provide unique value:

- Human-curated procedural knowledge improves performance +16.2pp. Self-generated instructions provide **zero benefit on average**. 2-3 focused modules per scope outperform comprehensive docs. (SkillsBench, 2602.12670)
- Behavioral triggers ("when X, do Y") are the highest-value content type. Claude Code was the **only** agent that performed worse with developer-provided context (Fig 3). (AGENTS.md, 2602.11988)
- Context files reduce runtime 28.64% and tokens 16.58% even when they don't improve correctness — they improve **efficiency**. (AGENTS.md Efficiency, 2601.20404)
- Agents asking clarifying questions improve 74% over non-interactive. 3-4 targeted questions optimal. (AMBIG-SWE, 2502.13069)
- System/user prompt separation fails: constraint followed only ~30% in conflict. Explicit labeling + priority order is far more effective. (Control Illusion, 2502.15851)
- Repeating critical instructions is neutral with reasoning enabled (improves non-reasoning: 47/70 wins; Repetition, 2512.14982). *(validated with reasoning toggled on/off)*
- Few-shot reasoning demonstrations in an example's *visible output* degrade thinking-model accuracy by 6-35% (From Harm to Help, 2509.23196; DeepSeek R1, 2501.12948). Reasoning inside `<thinking>` tags is the supported pattern (Anthropic Opus 4.7 guidance) — Claude generalizes the style to its own thinking blocks. Format/convention examples remain effective. *(tested on multiple reasoning models)*
- LLMs infer unstated requirements 41.1% of the time; inferred behaviors regress 2x across model updates. Format conventions 70.7% inferred (skip); conditional behaviors 22.9% (always specify). (Underspecification, 2505.13360) *(tested on standard inference; o3-mini partial)*
- Stable prompts achieve 41-80% cost savings via prompt caching. Session-varying content destroys cache. (Prompt Caching, 2601.06007)
- Repeated rewrites erode domain knowledge ("context collapse"). Prefer incremental edits. (ACE, 2510.04618)
- Skills files are an attack surface — payloads can hide in auxiliary files. (SkillJect, 2602.14211)

Primary sources: `docs/*.md` (official guidance, sibling to SKILL.md) and `anthropic-skills-complete-guide.pdf` (Anthropic's April 2026 skills guide — Tier 1, not arxiv). Supporting research: the remaining `*.pdf` files in the same directory (arxiv papers).

**Caveat on quantified percentages.** Many of the specific percentages above (e.g., 68% accuracy at 500 instructions, ±19/23%, 35-600% latency, 16.2pp procedural-knowledge lift, 70.7%/22.9% inference rates, 41-80% caching savings) come from studies on non-thinking or older models. Treat them as directional indicators of which way the effect goes, not as exact predictions for Opus 4.7 with adaptive thinking. The directional findings still hold: over- and under-specification both hurt; format conventions are highly inferable; conditional behaviors aren't; caching saves cost. The exact magnitudes likely differ on Opus 4.7 max-thinking.

## Core Principles

- **Thinking models reason via RL-trained strategies.** Prescriptive process instructions ("first do X, then Y, then Z") can constrain the model's internal search and degrade output. Prefer goals, constraints, and behavioral triggers over step-by-step recipes. (Anthropic Claude 4.x; OpenAI reasoning guidance; DeepSeek R1; CoT Value, 2506.07142)
- **Complex instruction files cause overthinking.** At higher effort settings, Opus 4.6 does significantly more upfront exploration. Instruction files with many conditional branches or verbose rules trigger excessive thinking tokens and slower responses. Minimalism is not just about context cost — it directly reduces latency. (Anthropic Claude 4.x)
- Never describe the codebase — the model reads source code, package.json, and can ls/glob.
- Write only rules the model **cannot infer** from reading code. Minimal means precise, not vague — each rule should be specific enough that a new engineer would know exactly what to do. "Follow our coding conventions" is underspecified; "Use snake_case for all variable names" is specified. Both over-specification (-19%) and underspecification (-23%) hurt. (Anthropic guidance; Underspecification, 2505.13360)
- For Opus 4.6 and 4.7, use normal language instead of emphasis markers (CRITICAL/MUST/IMPORTANT). These cause overtriggering on current frontier models — reserve emphasis only as a last resort. (Anthropic guidance)
- When a rule has a non-obvious rationale, include the WHY. Claude generalizes from motivations — one explained rule outperforms five unexplained ones. (Anthropic guidance)
- Frame rules positively. "Use named exports" is more reliably followed than "Don't use default exports." (Anthropic guidance)
- 2-3 focused modules per scope outperform comprehensive documentation.
- Among context file content types, behavioral triggers ("when X, do Y") have the highest value — but only when scoped precisely. Blanket triggers ("always run tests") cause over-compliance on trivial tasks. Prefer conditional triggers: "when modifying API endpoints, run integration tests."
- Every instruction creates an obligation and consumes cognitive bandwidth. If that obligation doesn't improve output, it degrades it — costing more tokens, more exploration steps, and potentially more failures. Over-specification (up to -19%) can hurt as much as underspecification (-23%).
- Every line must pass: "Would removing this cause Claude to make mistakes?"
- **Structural placement**: For long instruction files, lead with the highest-leverage triggers. Do NOT duplicate rules at top and bottom — the lost-in-middle effect is much weaker for thinking models with large contexts, and duplication wastes tokens. Concise files don't need positional engineering.
- **Format consistency**: Maintain consistent Markdown formatting across all instruction files. Your prompt's formatting style influences Claude's response style — Markdown-formatted prompts elicit Markdown responses. Don't mix plain text, Markdown, JSON, and YAML within a single file. (Anthropic guidance)
- **Claude Code-specific**: Claude Code was empirically the most sensitive agent to context noise (AGENTS.md study, Fig 3). Its sub-agents, glob, grep, and code graph mean it discovers structure autonomously. Be especially aggressive about minimalism for Claude Code users.

## CLAUDE.md vs Skills: Different Constraints

CLAUDE.md and skills serve fundamentally different roles. Do NOT apply CLAUDE.md rules to skills.

**CLAUDE.md (always-loaded)** — Every line costs context in EVERY session, even when irrelevant. Must be ruthlessly minimal. Line budgets are hard constraints.

**Skills (on-demand)** — Load only when relevant. The cost is amortized over only sessions that need them. **Signal density matters more than line count.** A 425-line skill that is 90% high-signal behavioral rules outperforms a 100-line skill that is 50% noise.

**Rules files (path-scoped)** — Loaded when working in matching file paths. Rules are the tightest instruction type: every line must be a behavioral trigger or constraint, no exposition. Pure "when X, do Y" patterns. Rules activate contextually, so they should contain only what differs from project-wide conventions.

**Sub-project rules files (path-scoped, high-density exception)** — When a single rules file serves as the de facto `CLAUDE.md` for an entire independent sub-project sharing a repo with other independent projects, evaluate by signal density, not line count — same principle as skills (see "When NOT to Trim Skills" below). These files legitimately carry regression-proven triggers and non-inferable domain knowledge across dozens or hundreds of files; the 30/50-line budget does not apply. Identifying signal: a scope note at the top of the file declaring it a sub-project `CLAUDE.md`, OR an entry in the project `CLAUDE.md` referencing it as such. Do not propose trimming these files to meet line budgets; instead apply the quality checklist (inferability, duplication, rule-vs-description) to individual additions.

## Skill Description Writing

The `description` frontmatter field decides whether a skill auto-triggers when the user's intent matches it. Treat it as the most important line in the skill — a poor description means the skill never fires when needed. (Anthropic Skills guidance.)

- **Third person.** "Processes Excel files." Not "I help…" or "You can use this to…" Mixed POV breaks discovery.
- **Description formula:** `[What it does] + [When to use it] + [Key capabilities]`. Example: "Reviews changed code for reuse, quality, and efficiency. Use when the user requests a code review or after substantive code changes. Identifies dead code, duplication, and convention violations." The "When to use it" half is what Claude matches against intent — never omit it. (Anthropic Skills Complete Guide.)
- **Lean toward triggering, not against it.** Claude tends to *under*trigger skills. Make the "Use when…" half slightly inclusive: enumerate paraphrased intents and adjacent phrasings, not just literal keywords. If the skill has a clear scope boundary, name it explicitly ("Use when … — not for general Y questions"). Pushy descriptions are correct under Opus's literal-following tendency. (Anthropic Skills Complete Guide; skill-creator guidance.)
- **Specific triggers, not vague themes.** Bad: "Helps with documents." Good: "Extracts text from PDFs. Use when the user mentions PDFs, forms, or document extraction."
- **Hard limits (Anthropic):** name ≤ 64 chars (lowercase letters, numbers, hyphens only; no XML tags; no reserved words `anthropic` or `claude`); description ≤ 1024 chars, non-empty, no XML tags.
- **Filesystem hard limits.** Skill folder must be kebab-case (no spaces, no underscores, no capitals). The main file must be exactly `SKILL.md` (case-sensitive — `skill.md`/`Skill.md` won't load). Do NOT place `README.md` inside the skill folder (frontmatter parser conflicts); put all instructions in `SKILL.md` or `references/`. Violations cause silent upload failures. (Anthropic Skills Complete Guide.)
- **For empirical description optimization, use `skill-creator`.** Anthropic's `skill-creator` plugin includes an optimization loop (60/40 train/test split, 5 iterations, picks best by held-out test score to avoid overfitting). Reach for it when a skill's trigger rate matters enough to formally evaluate; for everyday skills, the rules above are sufficient.
- Audit existing project skills against these rules before adding new content — if a skill isn't triggering reliably, the description is usually the cause, not the body.

## Content Classification

### ALWAYS-LOADED (CLAUDE.md) — include only

- Domain terminology not inferable from code
- Behavioral triggers and critical scaffolds
- Tool preference overrides
- Naming conventions that deviate from standard patterns
- Commit/PR conventions

### NEVER in CLAUDE.md

- Tech stack lists (model reads dependency/config files)
- Project structure trees (overviews don't help agents find files faster — AGENTS.md study, Fig 4)
- API surfaces or type definitions (model reads source)
- Tool reference tables (MCP manifests provide these)
- Algorithm explanations (model reads source)
- Code examples from the codebase
- Persona/role assignments ("You are an expert X") for *accuracy boost* — zero improvement on objective tasks. (Roles for tone/voice steering DO help and may live in CLAUDE.md if the project has a consistent voice requirement.)
- CoT directives ("think step by step") **with thinking enabled** — modern models reason by default; explicit CoT adds latency for marginal gain. (With thinking OFF, manual CoT inside `<thinking>` tags is still a valid fallback.)
- Format conventions the model already follows by default (70.7% inference rate) — only specify format rules that deviate from standard behavior

### DESCENDANT (nested CLAUDE.md files) — include only

- Domain knowledge spanning multiple modules or layers that no single file reveals
- Module-specific conventions that differ from project-wide patterns
- Error handling or state machine overviews scattered across many files
- Debugging tips for module-specific issues (hardware, third-party SDKs, etc.)

### ON-DEMAND (skills) — include

- Step-by-step procedural workflows (>5 steps)
- Architecture patterns and conventions
- Domain-specific decision trees
- Multi-step verification processes
- **Convention-establishing code examples** — HIGH VALUE in skills because they prescribe format/structure for new code (a behavioral rule expressed as code, not a codebase description). Use 2-3 examples showing the exact pattern to follow. Reasoning steps in an example's *visible output* are counterproductive with thinking enabled — the model's RL-trained reasoning outperforms example-mimicry. To shape reasoning style, place the reasoning inside `<thinking>` tags within the example (Anthropic Opus 4.7 guidance) — Claude generalizes that pattern to its own thinking blocks rather than copying it into output. (Anthropic Opus 4.7 guidance; SkillsBench, 2602.12670; 2602.15228)
- **Scoped behavioral triggers** — triggers should specify conditions, not blanket directives. Agents follow instructions even when irrelevant to the task, causing unnecessary exploration and tool use (AGENTS.md study, Fig 9). Pattern: "When X happens AND condition Y holds, do Z" — not "Always do Z."
- **Iterate on a single task before expanding.** When drafting a new skill, pick one challenging realistic task and iterate the draft until Claude succeeds, then extract the winning approach into the skill. This leverages in-context learning and gives faster signal than broad multi-task testing upfront. Once the foundation works, add coverage tasks. (Anthropic Skills Complete Guide.)

### Match prescriptiveness to fragility (degrees of freedom)

Match a skill's prescriptiveness to its task's fragility — wrong altitude is a real quality bug. (Anthropic Skills guidance.)

- **High freedom** (judgment text, "review the code structure") — when multiple paths are valid and decisions depend on context. Example: code review.
- **Medium freedom** (parameterized templates, pseudocode) — when a preferred pattern exists with acceptable variation. Example: report generation with format/include-charts knobs.
- **Low freedom** (exact commands, specific scripts) — when operations are fragile and consistency is critical. Example: DB migrations, release sequences, build-and-publish flows.

Low-freedom on judgment tasks produces brittle output (rigid checklists miss nuanced issues); high-freedom on fragile tasks produces inconsistent execution (ambiguity diverges across sessions).

## Quality Checklist (Primary Gate)

This checklist is the PRIMARY constraint for all edits. Line budgets are secondary guidelines.

Before every edit to a system prompt file, answer all seven:

1. **Inferable?** Can the model infer this by reading 1-3 files? → Don't add. But if inferring it requires reading 10+ files to notice a pattern across them → ADD it. Format conventions are 70.7% inferred by default — skip unless they deviate from standard behavior. Conditional/edge-case behaviors are only 22.9% inferred — prioritize specifying these.
2. **Duplicate?** Does this duplicate an MCP tool description? → Don't add.
3. **Budget?** Will this push the file over its line guideline? → For CLAUDE.md: condense or move to skill. For skills and sub-project rules files (see "CLAUDE.md vs Skills"): evaluate by signal density — only trim if density is low.
4. **Rule or description?** Is this a behavioral rule or a codebase description? → Only rules belong. But convention-establishing code examples ARE rules expressed as code (see above).
5. **Already covered?** Does an existing instruction already cover this? → Update, don't duplicate.
6. **Position?** For files >150 lines, are highest-leverage triggers near the top? → Long files reward leading with the most-important rules. Do NOT duplicate rules at the file's tail just to fight middle-attention — that effect is weak for thinking models with large contexts, and duplication wastes tokens.
7. **Cache-stable?** Does this content vary between sessions? → Session-varying data destroys prompt caching (41-80% cost savings lost). Dynamic state belongs in memory files or tool results, not always-loaded files. (2601.06007)

## Line Guidelines

These are soft guidelines, not hard limits. The quality checklist above is the real gate.

| File type | Guideline | Hard ceiling |
| --- | --- | --- |
| Global `~/.claude/CLAUDE.md` | ~40 lines | 50 |
| Project `./CLAUDE.md` | ~60 lines | 80 |
| Descendant CLAUDE.md | ~80 lines | 120 |
| Skills (on-demand) | Varies by signal density | 500 |
| Rules files | ~30 lines | 50 |
| Sub-project rules files | Signal-density-based | No fixed ceiling |

**For skills**: A 400-line skill with high signal density is BETTER than a 100-line skill that omits critical conventions. Do not trim a skill to meet an arbitrary line target. Instead, audit each section: does every line pass the quality checklist? If yes, the skill is the right length regardless of line count.

**For sub-project rules files**: Same signal-density principle as skills. A rules file that covers an entire independent sub-project will often exceed 100 lines legitimately. Apply the quality checklist to individual additions, not the cumulative line count.

## When NOT to Trim Skills

NEVER trim a skill section if:

- It documents a **convention that requires reading 10+ files to discover** (e.g., "all data-fetching modules use a two-file pattern with an options factory")
- It contains **code examples that prescribe patterns for new code** (e.g., a service layer template, a validation factory pattern)
- It captures **architectural decisions** that are not self-evident from any single file (e.g., "only 2 global stores exist", "modals use a custom hook, NOT route-based navigation")
- It defines **ordering conventions** across file internals (e.g., "imports → types → constants → logic → exports")
- Removing it would cause Claude to write code that **works but violates project conventions**

ALSO when authoring multi-file skills:

- **Keep file references one level deep from SKILL.md.** Chained references (SKILL.md → A.md → B.md) cause Claude to partially read B.md (e.g., `head -100`), losing information silently. All reference files should link directly from SKILL.md. (Anthropic Skills guidance.)

TRIM a skill section if:

- It restates standard language/framework knowledge Claude already has (e.g., "use built-in memoization primitives")
- It describes something visible from reading a single file (e.g., "the main entry point is in the root directory")
- It duplicates information in config files (package.json, pyproject.toml, Cargo.toml, etc.) or MCP tool descriptions
- It assigns personas or roles ("You are an expert X") *purely to boost accuracy* — no measurable benefit. (Keep role assignments that steer tone/voice when the skill produces voice-sensitive output.)
- It includes CoT directives ("think step by step") for modern models with thinking enabled — they reason by default. (With thinking off, manual CoT in `<thinking>` tags remains useful.)

## New MCP Server Pattern

1. Add 1-2 line trigger rule to appropriate CLAUDE.md (global if cross-project, project if specific).
2. Only create a skill if the server requires a detailed procedural workflow (>5 steps).
3. Never add tool reference tables — MCP manifests already describe tools.

## When to Update System Prompts

**UPDATE when:**

- Prefer incremental edits to wholesale rewrites of instruction files — repeated rewrites erode domain knowledge ("context collapse"). (ACE, 2510.04618)
- A new non-obvious architectural pattern is established
- A new domain term is introduced that can't be inferred from code
- A new behavioral trigger is discovered through experience
- A critical scaffold prevents a recurring failure mode
- The repo is poorly documented and context files fill a genuine gap (LLM-generated files actually help +2.7% when existing docs are absent — AGENTS.md study, Fig 5). Well-documented repos need less context; undocumented repos justify more.

**DO NOT UPDATE when:**

- You just added a new file or function (model can read it)
- You refactored code structure (model can ls/glob)
- You added a new dependency (model reads package.json)
- The information is already documented in source code comments
- The convention follows standard framework patterns the model already knows (70.7% of format conventions are inferred without explicit specification)

**DIAGNOSE when an instruction file isn't working as expected:**

- Rule keeps being ignored → File is probably too long. Prune other rules rather than adding emphasis — emphasis markers cause overtriggering on Opus 4.6+ (see Core Principles) and mask the real problem.
- Model asks questions answered in the file → Phrasing is ambiguous. Rewrite the rule for clarity, not volume.
- Adding a rule doesn't change behavior → Model already follows it naturally. Remove the rule (see "Every line must pass" test above).
- Skill never auto-loads when it should → description **undertriggers**. Add detail and trigger keywords (technical terms, file types, paraphrased intents) and lean the "Use when…" half slightly pushy. (Anthropic Skills Complete Guide.)
- Skill loads on irrelevant queries → description **overtriggers**. Add explicit negative triggers ("Do NOT use for…"), tighten scope to specific contexts, or be more specific about the conditions in the "Use when…" half. (Anthropic Skills Complete Guide.)

## Merge Arbiter Pattern

When multiple autonomous instances produce conflicting edits to the same instruction file:

1. A separate Claude Opus instance loads this skill as the decision framework.
2. The arbiter evaluates BOTH versions against the quality checklist.
3. The arbiter picks the version that is more concise, more behavioral (not descriptive), and within line guidelines.
4. If both versions add value, the arbiter merges them — but NEVER by concatenation. The merged result must itself pass the quality checklist.
5. The arbiter must verify the merged file stays within the line guideline for its file type.

## Self-Protection

This skill is a curated human artifact backed by 5 official guidance docs + 18 empirical studies. Do NOT modify this skill or any file in this skill's directory unless a human explicitly requests it. Self-generated procedural knowledge provides zero benefit on average (SkillsBench).

Skills that reference external scripts should be scrutinized — payloads can hide in auxiliary files. (SkillJect, 2602.14211)

**When a human requests changes to this skill**: Before applying any edit, read the relevant sections of the research papers in this skill's `docs/*.pdf` and verify the proposed changes align with the research foundations. Do not weaken, remove, or contradict findings from the source papers without the human explicitly acknowledging the deviation.
