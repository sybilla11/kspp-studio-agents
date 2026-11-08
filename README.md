# KSPP Studio Agents

[![Validate](https://github.com/sybilla11/kspp-studio-agents/actions/workflows/validate.yml/badge.svg)](https://github.com/sybilla11/kspp-studio-agents/actions/workflows/validate.yml)

Claude Code plugins for everyday engineering workflows, packaged as a plugin marketplace you can add to any workspace, including private ones. Each plugin runs inside your own Claude Code session, against your own code. There is no server, no API key to set up, and nothing to host.

| Plugin | What it does |
|---|---|
| [**kspp-analysis**](plugins/kspp-analysis) — KSPP Studio Analysis Agent | Turns a user story or ticket into gaps and open questions, Given/When/Then acceptance criteria, a codebase impact map with file and line evidence, and an ordered TDD test plan, before anyone writes code. |

## KSPP Studio Analysis Agent

Give it a story:

```
/kspp-analysis:analyze-story As a wallet holder, I want to add money from my linked bank account to any of my wallets, including a foreign-currency wallet...
```

It reads your codebase (read-only), then returns a report in seven sections:

1. **Story summary** with a verdict: Ready, Ready with assumptions, or Needs answers before starting
2. **Gaps and open questions**, marked blocking or not, plus the assumptions it used for the rest
3. **Acceptance criteria** in Given/When/Then: happy path, validation and errors, edge cases, permissions, non-functional
4. **Codebase impact map**: which files change and the evidence for each (`client.ts:14`), patterns to follow, and tests to extend
5. **TDD test plan**: failing tests in red→green order, in your project's test framework and naming style, each mapped to a criterion
6. **Risks and dependencies**
7. **Size and slicing**: a T-shirt size, and vertical slices if the story is too big

**See a real run:** [examples/multi-currency-wallet-add-money.md](examples/multi-currency-wallet-add-money.md). It's the unedited report for an "Add money" story on [multi-currency-wallet](https://github.com/sybilla11/multi-currency-wallet). The agent caught that the story depends on a bank-linking feature planned for a later phase, found that the API client can't send writes yet, and built on decisions already recorded in the project's architecture doc.

### How it's built

- **A subagent, [`story-analyst`](plugins/kspp-analysis/agents/story-analyst.md),** does the analysis in its own context window. Your main session gets the report without the dozens of file reads behind it.
- **Read-only by design.** It can only use `Read`, `Glob`, and `Grep`. It can't edit files, run commands, or reach the network.
- **Grounded, not guessed.** It must cite only files it actually opened or matched, and label proposed new files as new. In the example run, every cited path exists and every line number is within the file.
- **Story text is treated as data.** Instructions inside a ticket ("ignore the above…") are analyzed, not followed.
- **Uses your model.** `model: inherit` runs it on whatever model your session uses.
- **A thin command, [`/kspp-analysis:analyze-story`](plugins/kspp-analysis/skills/analyze-story/SKILL.md),** accepts pasted text or a file path, delegates to the agent, and can save the report with `--save`. You can also skip the command and just ask Claude to "analyze this story" — the agent's description lets Claude pick it automatically.

## Install

Requires [Claude Code](https://code.claude.com).

### Just for you

In a Claude Code session:

```
/plugin marketplace add sybilla11/kspp-studio-agents
/plugin install kspp-analysis@kspp-studio-agents
```

Or from a terminal:

```bash
claude plugin marketplace add sybilla11/kspp-studio-agents
claude plugin install kspp-analysis@kspp-studio-agents
```

> **Using the VS Code or JetBrains extension?** The `/plugin` command only works in Claude Code in a terminal; the IDE extensions reply "/plugin isn't available in this environment". Run the two `claude plugin` commands above in your IDE's integrated terminal instead, then start a new Claude Code conversation so it picks up the plugin.

### For a whole team, in a private repo

Commit this to your project's `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "kspp-studio-agents": {
      "source": { "source": "github", "repo": "sybilla11/kspp-studio-agents" }
    }
  },
  "enabledPlugins": {
    "kspp-analysis@kspp-studio-agents": true
  }
}
```

When teammates open the repo in Claude Code and accept the workspace trust prompt, the marketplace is registered and the plugin is enabled. Nobody has to run install commands. Claude Code ignores these entries in folders that haven't been trusted.

To pin a version instead of following `main`, fork this repo or point `source` at a tag. Plugins only see what Claude Code already sees in your session. This repo has no telemetry and makes no network calls.

## Usage

```
/kspp-analysis:analyze-story <story text>
/kspp-analysis:analyze-story docs/tickets/PROJ-142.md
/kspp-analysis:analyze-story docs/tickets/PROJ-142.md --save   # also writes docs/stories/<slug>.md
```

Run it from your project's root so the agent can map the story onto your code. It still works without a codebase (for example, refining a story before a repo exists) and says so in the impact section.

## Repository layout

```
.claude-plugin/marketplace.json      # the marketplace: lists every plugin in this repo
plugins/
  kspp-analysis/
    .claude-plugin/plugin.json       # plugin manifest
    agents/story-analyst.md          # the subagent: tools, model, and system prompt
    skills/analyze-story/SKILL.md    # the /kspp-analysis:analyze-story command
examples/                            # real runs, with their input stories
.github/workflows/validate.yml       # CI: validates every manifest on push and PR
```

## Adding a plugin

1. Create `plugins/<name>/.claude-plugin/plugin.json` and add agents under `agents/` and skills under `skills/<skill>/SKILL.md`.
2. Add an entry to `.claude-plugin/marketplace.json` with `"source": "./plugins/<name>"`. Its `name` must match the plugin's `plugin.json`.
3. Test locally without installing:
   ```bash
   claude --plugin-dir ./plugins/<name>
   claude plugin validate . --strict && claude plugin validate ./plugins/<name> --strict
   ```
4. Add a real run under `examples/`.

## License

[MIT](LICENSE)
