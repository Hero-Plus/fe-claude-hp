# Anthropic: Claude Code Skills Documentation

**Source:** https://docs.anthropic.com/en/docs/claude-code/skills (redirects to code.claude.com/docs/en/skills)
**Fetched:** 2026-03-03

Official documentation for authoring, configuring, and sharing Claude Code skills.

---

## Core Concept

> "Skills extend what Claude can do. Create a SKILL.md file with instructions, and Claude adds it to its toolkit. Claude uses skills when relevant, or you can invoke one directly with `/skill-name`."

Skills follow the [Agent Skills](https://agentskills.io) open standard.

---

## 1. Context Window as a Public Good

The skills system is designed around a fundamental constraint: **skill descriptions are loaded into context, but full skill content only loads when invoked.**

> "In a regular session, skill descriptions are loaded into context so Claude knows what's available, but full skill content only loads when invoked."

**Character budget:** Scales dynamically at **2% of the context window**, with a fallback of 16,000 characters.

> "If you have many skills, they may exceed the character budget. The budget scales dynamically at 2% of the context window."

This means every skill's `description` field competes for a shared, limited resource. Keep descriptions concise but distinctive enough for Claude to match accurately.

---

## 2. Skill File Structure

```
my-skill/
  SKILL.md           # Main instructions (required)
  template.md        # Template for Claude to fill in
  examples/
    sample.md        # Example output showing expected format
  scripts/
    validate.sh      # Script Claude can execute
```

**Keep SKILL.md under 500 lines.** Move detailed reference material to separate files.

---

## 3. Frontmatter Reference

```yaml
---
name: my-skill # Display name, becomes /slash-command
description: What this does # Helps Claude decide when to use it
argument-hint: [issue-number] # Shown during autocomplete
disable-model-invocation: true # Only user can trigger
user-invocable: false # Only Claude can trigger (background knowledge)
allowed-tools: Read, Grep # Tools available without permission prompts
model: opus # Model override
context: fork # Run in subagent
agent: Explore # Subagent type
---
```

All fields are optional. Only `description` is recommended.

---

## 4. Two Types of Skill Content

### Reference Content (Knowledge)

Conventions, patterns, style guides, domain knowledge. Runs inline alongside conversation context.

```yaml
---
name: api-conventions
description: API design patterns for this codebase
---
When writing API endpoints:
  - Use RESTful naming conventions
  - Return consistent error formats
```

### Task Content (Workflows)

Step-by-step instructions for specific actions. Often invoked manually with `/skill-name`. Add `disable-model-invocation: true` to prevent automatic triggering.

```yaml
---
name: deploy
description: Deploy the application to production
context: fork
disable-model-invocation: true
---
Deploy the application:
1. Run the test suite
2. Build the application
3. Push to the deployment target
```

---

## 5. Invocation Control

| Frontmatter                      | User can invoke | Claude can invoke | Context loading                                       |
| -------------------------------- | --------------- | ----------------- | ----------------------------------------------------- |
| (default)                        | Yes             | Yes               | Description always loaded; full content on invocation |
| `disable-model-invocation: true` | Yes             | No                | Description NOT in context; loads when user invokes   |
| `user-invocable: false`          | No              | Yes               | Description always loaded; loads when Claude invokes  |

Key insight: `disable-model-invocation: true` **removes the skill description from context entirely**, freeing up the character budget.

---

## 6. Where Skills Live

| Location   | Path                               | Applies to                |
| ---------- | ---------------------------------- | ------------------------- |
| Enterprise | Managed settings                   | All users in organization |
| Personal   | `~/.claude/skills/<name>/SKILL.md` | All your projects         |
| Project    | `.claude/skills/<name>/SKILL.md`   | This project only         |
| Plugin     | `<plugin>/skills/<name>/SKILL.md`  | Where plugin is enabled   |

Priority: enterprise > personal > project. Plugin skills use namespace (`plugin-name:skill-name`) and cannot conflict.

### Automatic Discovery

When working with files in subdirectories, Claude Code discovers skills from nested `.claude/skills/` directories (supports monorepos).

---

## 7. Supporting Files

Reference supporting files from SKILL.md so Claude knows what they contain and when to load them:

```markdown
## Additional resources

- For complete API details, see [reference.md](reference.md)
- For usage examples, see [examples.md](examples.md)
```

This enables **progressive disclosure** — SKILL.md stays focused, detailed references load on-demand.

---

## 8. Dynamic Context Injection

The `` !`command` `` syntax runs shell commands before skill content is sent to Claude:

```yaml
---
name: pr-summary
description: Summarize changes in a pull request
context: fork
---
## Pull request context
- PR diff: !`gh pr diff`
- PR comments: !`gh pr view --comments`
```

Commands execute immediately; output replaces the placeholder. Claude only sees the final rendered prompt.

---

## 9. String Substitutions

| Variable                | Description                        |
| ----------------------- | ---------------------------------- |
| `$ARGUMENTS`            | All arguments passed when invoking |
| `$ARGUMENTS[N]` or `$N` | Specific argument by 0-based index |
| `${CLAUDE_SESSION_ID}`  | Current session ID                 |

If `$ARGUMENTS` is not present in content, arguments are appended as `ARGUMENTS: <value>`.

---

## 10. Running Skills in Subagents

Add `context: fork` for isolated execution. The skill content becomes the subagent's prompt (no access to conversation history).

> "context: fork only makes sense for skills with explicit instructions. If your skill contains guidelines without a task, the subagent receives the guidelines but no actionable prompt."

Agent types: `Explore` (read-only), `Plan`, `general-purpose`, or any custom subagent from `.claude/agents/`.

---

## 11. Bundled Skills

Ships with Claude Code:

- **`/simplify`**: Reviews recently changed files for code reuse, quality, efficiency. Spawns 3 review agents in parallel.
- **`/batch <instruction>`**: Orchestrates large-scale changes across codebase in parallel. Spawns one background agent per unit in isolated git worktrees.
- **`/debug [description]`**: Troubleshoots current session by reading debug log.

---

## 12. Troubleshooting

| Problem                       | Solution                                                                                                                           |
| ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| Skill not triggering          | Check description keywords match user's natural language                                                                           |
| Skill triggers too often      | Make description more specific, or add `disable-model-invocation: true`                                                            |
| Claude doesn't see all skills | Character budget exceeded (2% of context window). Run `/context` to check. Override with `SLASH_COMMAND_TOOL_CHAR_BUDGET` env var. |

---

## 13. Design Principles for Skill Authors

From the documentation's patterns:

1. **Description is the critical field** — it determines both automatic invocation accuracy and context budget usage
2. **Separate reference from task content** — know whether your skill adds knowledge or defines a workflow
3. **Use supporting files** for anything over 500 lines
4. **Use `disable-model-invocation: true`** for side-effect workflows AND to save context budget
5. **Use `context: fork`** when the skill performs deep work that would pollute the main conversation
6. **Dynamic injection (`` !`command` ``)** enables fresh, live data without stale context
