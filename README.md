# fe-claude-hp

FE-managed Claude Code marketplace for HeroPlus. Carries cross-repo skills, hooks, and atlases that span the FE-owned HeroPlus codebases (`heroplus-terminal`, `heroplus-a8`), plus general FE engineering tooling (e.g. `prompt-engineering`).

BE Claude Code tooling is deliberately out of scope so BE can structure their own marketplace independently.

## Plugins shipped

| Plugin | Purpose | Triggers on |
|---|---|---|
| `rki` | Cross-repo atlas for HeroPlus Remote Key Injection. Carries Sunmi `key_type` enum, kif-bridge bug status, P3 slot map, A8 binding status, KCV-only hygiene rule, per-repo file index. | RKI / BDK / IPEK / KCV / DUKPT / kif-bridge / Sunmi portal / KPay-KDP / MYHSM queries |
| `prompt-engineering` | Research-backed authoring rules for Claude Code skills, CLAUDE.md, rules files, hooks, and subagents. Backed by 5 official guidance docs + 18 empirical studies. | skill / CLAUDE.md / hook / subagent / instruction-file authoring intents |

## Install (per developer machine)

Once per machine, from any Claude Code session:

```
/plugin marketplace add github:Hero-Plus/fe-claude-hp
/plugin install rki@fe-claude-hp
```

After install, the plugin is available in **every Claude Code session on this machine** — in any HeroPlus repo (heroplus-terminal, heroplus-a8, be-heroplus, heroplus-kif-bridge) or anywhere else.

For teammates pulling `heroplus-terminal`, the marketplace is auto-registered via that repo's `.claude/settings.json` `extraKnownMarketplaces` entry — they only need the second command (`/plugin install rki@fe-claude-hp`), or if `enabledPlugins` is also set there, no commands at all.

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
