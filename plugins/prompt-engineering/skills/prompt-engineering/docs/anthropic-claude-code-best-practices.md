# Anthropic: Claude Code Best Practices

**Source:** https://docs.anthropic.com/en/docs/claude-code/best-practices (redirects to code.claude.com/docs/en/best-practices)
**Fetched:** 2026-03-03

Official guide from Anthropic covering patterns proven effective across internal teams and external engineers.

---

## Central Constraint

> "Most best practices are based on one constraint: Claude's context window fills up fast, and performance degrades as it fills."

> "The context window is the most important resource to manage."

---

## 1. CLAUDE.md Design Guidance

### What to Include vs. Exclude

| Include                                              | Exclude                                            |
| ---------------------------------------------------- | -------------------------------------------------- |
| Bash commands Claude can't guess                     | Anything Claude can figure out by reading code     |
| Code style rules that differ from defaults           | Standard language conventions Claude already knows |
| Testing instructions and preferred test runners      | Detailed API documentation (link to docs instead)  |
| Repository etiquette (branch naming, PR conventions) | Information that changes frequently                |
| Architectural decisions specific to your project     | Long explanations or tutorials                     |
| Developer environment quirks (required env vars)     | File-by-file descriptions of the codebase          |
| Common gotchas or non-obvious behaviors              | Self-evident practices like "write clean code"     |

### Key Design Principles

> "CLAUDE.md is loaded every session, so only include things that apply broadly. For domain knowledge or workflows that are only relevant sometimes, use skills instead."

> "Keep it concise. For each line, ask: 'Would removing this cause Claude to make mistakes?' If not, cut it."

> **"Bloated CLAUDE.md files cause Claude to ignore your actual instructions!"**

### Emphasis Markers

> "You can tune instructions by adding emphasis (e.g., 'IMPORTANT' or 'YOU MUST') to improve adherence."

### Maintenance as Code

> "Treat CLAUDE.md like code: review it when things go wrong, prune it regularly, and test changes by observing whether Claude's behavior actually shifts."

Diagnostic signals:

- If Claude keeps doing something wrong despite a rule: the file is probably too long and the rule is getting lost
- If Claude asks questions answered in CLAUDE.md: the phrasing might be ambiguous

### File Locations

| Location              | Scope                                           |
| --------------------- | ----------------------------------------------- |
| `~/.claude/CLAUDE.md` | All Claude sessions                             |
| `./CLAUDE.md`         | Project (check into git to share with team)     |
| `./CLAUDE.local.md`   | Project (gitignored, personal)                  |
| Parent directories    | Monorepos (root + child both loaded)            |
| Child directories     | Loaded on-demand when working in that directory |

### Import Syntax

CLAUDE.md files can import additional files using `@path/to/import`:

```markdown
See @README.md for project overview and @package.json for available npm commands.

# Additional Instructions

- Git workflow: @docs/git-instructions.md
```

---

## 2. Give Claude a Way to Verify Its Work

> "This is the single highest-leverage thing you can do."

> "Claude performs dramatically better when it can verify its own work — run tests, compare screenshots, and validate outputs."

Strategies:

- Provide test cases with expected outputs
- Use screenshot comparison for UI changes
- Give root causes, not symptoms, in error reports

---

## 3. Explore First, Then Plan, Then Code

Four-phase workflow:

1. **Explore** (Plan Mode) — read files, answer questions, no changes
2. **Plan** — create detailed implementation plan
3. **Implement** (Normal Mode) — code against the plan, run tests
4. **Commit** — descriptive message, open PR

> "Planning is most useful when you're uncertain about the approach, when the change modifies multiple files, or when you're unfamiliar with the code being modified. If you could describe the diff in one sentence, skip the plan."

---

## 4. Context Management

### Clear Between Tasks

> "Use `/clear` frequently between tasks to reset the context window entirely."

### Compaction

> "When auto compaction triggers, Claude summarizes what matters most, including code patterns, file states, and key decisions."

Custom compaction: add instructions like `"When compacting, always preserve the full list of modified files and any test commands"` to CLAUDE.md.

### Subagents for Investigation

> "Since context is your fundamental constraint, subagents are one of the most powerful tools available."

Subagents explore in separate context windows, report summaries back, keeping main conversation clean.

---

## 5. Course-Correct Early

> "If you've corrected Claude more than twice on the same issue in one session, the context is cluttered with failed approaches. Run `/clear` and start fresh with a more specific prompt."

> "A clean session with a better prompt almost always outperforms a long session with accumulated corrections."

---

## 6. Common Failure Patterns

| Pattern                                                                | Fix                                                            |
| ---------------------------------------------------------------------- | -------------------------------------------------------------- |
| **Kitchen sink session** — unrelated tasks in one session              | `/clear` between unrelated tasks                               |
| **Correcting over and over** — context polluted with failed approaches | After two failures, `/clear` and write a better initial prompt |
| **Over-specified CLAUDE.md** — too long, important rules lost in noise | Ruthlessly prune; delete rules Claude already follows          |
| **Trust-then-verify gap** — plausible but untested output              | Always provide verification (tests, scripts, screenshots)      |
| **Infinite exploration** — unscoped investigation fills context        | Scope investigations narrowly or use subagents                 |

---

## 7. Skills (Brief Mention)

> "Skills extend Claude's knowledge with information specific to your project, team, or domain. Claude applies them automatically when relevant, or you can invoke them directly with `/skill-name`."

Create skills in `.claude/skills/` with a `SKILL.md` file. Use `disable-model-invocation: true` for workflows with side effects.

---

## 8. Hooks

> "Unlike CLAUDE.md instructions which are advisory, hooks are deterministic and guarantee the action happens."

Use hooks for actions that must happen every time with zero exceptions (linting, blocking writes to certain folders).

---

## 9. Scaling: Parallel Sessions and Fan-Out

- Run multiple Claude sessions for parallel work
- Use `claude -p "prompt"` for non-interactive CI/scripting
- Fan out across files with loops calling `claude -p` per file
- Writer/Reviewer pattern: one session implements, another reviews with fresh context

---

## 10. Develop Your Intuition

> "Pay attention to what works. When Claude produces great output, notice what you did: the prompt structure, the context you provided, the mode you were in."

> "Over time, you'll develop intuition that no guide can capture."
