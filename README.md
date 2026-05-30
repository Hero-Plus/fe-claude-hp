# fe-claude-hp

FE-managed Claude Code marketplace for HeroPlus. Carries cross-repo skills, hooks, and atlases that span the FE-owned HeroPlus codebases (`heroplus-terminal`, `heroplus-a8`, `heroplus-web`, `heroplus-merchant`, `hero-plus-merchant-react-native`), plus general FE engineering tooling (`prompt-engineering`, `commit-message`).

BE Claude Code tooling is deliberately out of scope so BE can structure their own marketplace independently.

## Plugins shipped

| Plugin | Purpose | Triggers on |
|---|---|---|
| `rki` | Cross-repo atlas for HeroPlus Remote Key Injection. Carries Sunmi `key_type` enum, kif-bridge bug status, P3 slot map, A8 binding status, KCV-only hygiene rule, per-repo file index. | RKI / BDK / IPEK / KCV / DUKPT / kif-bridge / Sunmi portal / KPay-KDP / MYHSM queries |
| `prompt-engineering` | Research-backed authoring rules for Claude Code skills, CLAUDE.md, rules files, hooks, and subagents. Backed by 5 official guidance docs + 18 empirical studies. | skill / CLAUDE.md / hook / subagent / instruction-file authoring intents |
| `commit-message` | Intent-first commit message style — each body point names the problem the change solves, not the diff. Scope vocabulary defers to each repo's CLAUDE.md. | generating any commit message, staging changes, running `git commit` |

## Per-repo subset

Each FE repo commits the plugin subset it wants in its own `.claude/settings.json` (`extraKnownMarketplaces` + `enabledPlugins`). The subset is the repo owner's explicit, version-controlled choice:

| Plugin | Repos |
|---|---|
| `commit-message` | all FE repos |
| `prompt-engineering` | all FE repos |
| `rki` | `heroplus-terminal`, `heroplus-a8` (POS only — key injection) |

## Install

**Fresh clone (new engineer).** On the first Claude Code session in a freshly-cloned repo, after you trust the folder, Claude Code auto-discovers this marketplace from the committed `extraKnownMarketplaces` entry and offers the repo's declared `enabledPlugins` for install — accept once. No marketplace URL to remember, no per-plugin commands to type.

**Already-cloned repo that just gained a new plugin** (e.g. a plugin newly added to `enabledPlugins` by a pull): with `autoUpdate: true` the marketplace clone refreshes on session start, then install the one new plugin once per machine:

```
/plugin install commit-message@fe-claude-hp
```

> The floor is a single accept-prompt (or one `/plugin install`) **per machine** — a one-time step, not per-session. Truly silent auto-install from `enabledPlugins` is a pending Claude Code feature ([#23737](https://github.com/anthropics/claude-code/issues/23737)); until it ships, expect this one-time confirmation.

**Manual / machine-wide.** From any Claude Code session, to add the marketplace and a plugin at user scope (available in every repo on the machine):

```
/plugin marketplace add github:Hero-Plus/fe-claude-hp
/plugin install <plugin>@fe-claude-hp
```

## Adding a new plugin to this marketplace

1. Create the plugin under `plugins/<plugin-name>/`:

   ```
   plugins/<plugin-name>/
   ├── .claude-plugin/
   │   └── plugin.json          # { "name": "<plugin-name>", "description": "..." }
   ├── README.md (optional)
   └── skills/
       └── <skill-name>/
           └── SKILL.md
   ```

2. Add an entry to `.claude-plugin/marketplace.json`'s `plugins` array:

   ```json
   {
     "name": "<plugin-name>",
     "description": "...",
     "category": "internal",
     "source": "./plugins/<plugin-name>"
   }
   ```

3. Commit + push. Teammates pull and run `/plugin install <plugin-name>@fe-claude-hp` (or add to `enabledPlugins` in committed settings for auto-enable).

## Why this marketplace exists

Skills that need to fire across multiple HeroPlus repos can't live as project-scoped skills in `.claude/skills/` of any one repo — they'd only fire in that repo's sessions. Putting them in a plugin under a Git-hosted marketplace lets them auto-discover (via `extraKnownMarketplaces` in committed settings) and auto-fire in every Claude Code session on the developer's machine, regardless of working directory.

## KCV-only hygiene

For anything RKI-adjacent: record only KCVs (3-byte check values, `TDES_ECB(0…0, K)[:3]`) and public resource identifiers (AWS `KeyArn`, certificate KCVs, KSN, key-name labels). Never commit raw key material or TR-31 / TR-34 / OAEP wrapped key blocks into this marketplace's content — sandbox or not. Git history retains anything ever committed; rotation is the only clean remedy.
