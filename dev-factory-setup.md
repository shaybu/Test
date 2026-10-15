# Dev Factory Setup: Step-by-Step Install Plan

> **How to run:** put this file anywhere on your Mac, open Claude Code and say:
> `Read dev-factory-setup.md and follow it step by step.`

## Instructions for Claude Code

You are setting up my personal Claude Code "dev factory" in `~/.claude/` from four GitHub repos. Follow these rules for the whole plan:

1. **Back up first.** Copy `~/.claude/` to `~/.claude-backup-<date>/` before changing anything.
2. **Clone into a scratch folder.** Shallow-clone each repo into `~/factory-src/`. Never copy a whole repo into `~/.claude/`.
3. **Take only what's listed.** Install only the items in the "What to take" table. Anything else in a repo stays in `~/factory-src/`.
4. **Check dependencies before copying.** Before you copy a skill or agent, read its file and list everything it references: other skills (`see skill: ...`), other agents, slash commands, scripts, hooks, MCP servers and CLI tools. If a referenced skill or agent is in this plan, copy it too. If it isn't, don't install it silently. Add it to the final report as "missing dependency" and say whether the item still works without it.
5. **Fix agent models.** ECC agents have `model: sonnet` or `model: opus` in their frontmatter. My models are served through a LiteLLM gateway, so change every copied agent to `model: inherit`.
6. **No hooks, no MCP servers, no auto-updaters.** Don't register anything in `~/.claude/settings.json` except the Superpowers plugin, and don't run any repo's `setup` or `install.sh` script.
7. **Don't duplicate.** If a skill or agent with the same name already exists in `~/.claude/`, stop and ask me before overwriting it.
8. **Report at the end:** a table of what you installed and where, every missing dependency, every CLI tool the installed items expect (`ruff`, `playwright`, etc.) and whether it's on my machine, and anything you skipped and why.

## What to take from each repo

This is the complete install list. Each item's dependencies are already in the list, except the CLI tools, which live in your projects rather than in `~/.claude/`.

| Repo | Item | Type | Install to | Depends on |
| --- | --- | --- | --- | --- |
| [multica-ai/andrej-karpathy-skills](https://github.com/multica-ai/andrej-karpathy-skills) | `CLAUDE.md` content | Always-on rules | Appended to `~/.claude/CLAUDE.md` | Nothing |
| [obra/superpowers](https://github.com/obra/superpowers) | Whole plugin (15 skills plus a session-start hook) | Plugin | Claude Code plugin system | Its skills call each other, so install it whole, never skill by skill |
| [affaan-m/ECC](https://github.com/affaan-m/ECC) | `skills/security-review` | Skill | `~/.claude/skills/security-review/` | Nothing |
| ECC | `agents/security-reviewer.md` | Agent | `~/.claude/agents/` | Skill `security-review`. CLIs in JS projects: `npm audit`, `eslint-plugin-security` |
| ECC | `skills/verification-loop` | Skill | `~/.claude/skills/verification-loop/` | The project's own build, lint, type and test tools (`npm`, `tsc`, `ruff`, `pyright`). Mentions a `/verify` command that isn't needed |
| ECC | `skills/e2e-testing` | Skill | `~/.claude/skills/e2e-testing/` | Playwright in the project being tested |
| ECC | `agents/e2e-runner.md` | Agent | `~/.claude/agents/` | Skill `e2e-testing`. Prefers the `agent-browser` CLI and falls back to Playwright. Playwright is enough |
| ECC | `skills/search-first` | Skill | `~/.claude/skills/search-first/` | Uses the built-in general-purpose subagent. Names a researcher, planner and architect agent and an `iterative-retrieval` skill as optional extras. Works without them |
| ECC | `skills/python-patterns` | Skill (Python) | `~/.claude/skills/python-patterns/` | Nothing. Needed by `python-reviewer` |
| ECC | `agents/python-reviewer.md` | Agent (Python) | `~/.claude/agents/` | Skill `python-patterns`. CLIs, used if present: `ruff`, `mypy`, `black`, `bandit`, `pytest` |
| ECC | `skills/coding-standards` | Skill (TS/JS) | `~/.claude/skills/coding-standards/` | Needed by `typescript-reviewer` |
| ECC | `skills/backend-patterns` | Skill (Node.js) | `~/.claude/skills/backend-patterns/` | Needed by `typescript-reviewer` for Node code |
| ECC | `skills/frontend-patterns` | Skill (React / Next.js) | `~/.claude/skills/frontend-patterns/` | Needed by `typescript-reviewer` for frontend code |
| ECC | `agents/typescript-reviewer.md` | Agent (TS/JS) | `~/.claude/agents/` | Skills `coding-standards`, `backend-patterns`, `frontend-patterns`. Hands `.tsx`/`.jsx` files to `react-reviewer` (next row). Tries `gh pr view` and falls back to `git diff` without it. CLIs: `tsc`, `eslint`, `prettier`, `npm audit`, `vitest` or `jest` |
| ECC | `agents/react-reviewer.md` | Agent (React) | `~/.claude/agents/` | Take only if your frontend is React. Claude checks its dependencies in Step 3 |
| ECC | `agents/build-error-resolver.md` | Agent (TS) | `~/.claude/agents/` | CLIs `tsc`, `npm`, `eslint`. Names other agents only as "use instead" pointers. Works without them |
| ECC | `rules/common/`, `rules/python/`, `rules/typescript/` | Always-on rules | `~/.claude/rules/ecc/` | Copy whole folders. Then remove `agents.md` and `hooks.md` from `common/`, since they describe ECC agents and hooks you aren't installing |
| [garrytan/gstack](https://github.com/garrytan/gstack) | `agents-digest/gstack-AGENTS.md` (optional) | Always-on rules | Appended to `~/.claude/CLAUDE.md` | Nothing |

This covers both Python and TypeScript / Node.js / frontend. To add Java later, take `java-reviewer` + `java-coding-standards` + `rules/java/` the same way.

**Don't take:**

- **ECC plugin, `install.sh`, hooks, `commands/`, `mcp-configs/`** — they load the full 300+ skill catalog and run scripts on every tool call.
- **ECC `tdd-workflow`, `code-reviewer`, `planner`, `tdd-guide`** — Superpowers already does TDD, planning and review. Two versions of the same workflow compete for the same trigger.
- **ECC `multi-*` commands** — they need the separate ccg-workflow runtime.
- **ECC `/astra-review`** — it sends code to OpenAI.
- **ECC `continuous-learning`** — it depends on ECC's hook runtime.
- **gstack `./setup`** — it builds a Chromium browser, adds a Stop hook and session auto-update checks, and rewrites `CLAUDE.md`. Its `/review`, `/qa` and `/ship` overlap with Superpowers.

## Step 1: Karpathy guidelines

```bash
mkdir -p ~/factory-src && cd ~/factory-src
git clone --depth 1 https://github.com/multica-ai/andrej-karpathy-skills.git
printf '\n\n' >> ~/.claude/CLAUDE.md
cat andrej-karpathy-skills/CLAUDE.md >> ~/.claude/CLAUDE.md
```

- Take only `CLAUDE.md`. Skip its plugin and `skills/karpathy-guidelines`, which hold the same text; installing both loads it twice.
- If `~/.claude/CLAUDE.md` already has rules, append; never replace what's there.

## Step 2: Superpowers plugin

Install the whole plugin through Claude Code's plugin system. Its skills call each other: brainstorming hands off to writing-plans, which hands off to subagent-driven-development, which uses test-driven-development and requesting-code-review. Cherry-picking would break that chain.

```bash
claude plugin marketplace add obra/superpowers-marketplace
claude plugin install superpowers@superpowers-marketplace
echo 'export SUPERPOWERS_DISABLE_TELEMETRY=1' >> ~/.zshrc
```

- If the `claude plugin` command isn't available, stop and ask me to run these inside Claude Code instead: `/plugin marketplace add obra/superpowers-marketplace`, then `/plugin install superpowers@superpowers-marketplace`.
- The telemetry variable stops the brainstorming visual companion from loading its logo from Prime Radiant's website.
- The plugin's session-start hook is the one hook we keep. It's what makes the skills trigger automatically.

What this installs: brainstorming, writing-plans, executing-plans, subagent-driven-development, dispatching-parallel-agents, test-driven-development, systematic-debugging, verification-before-completion, requesting-code-review, receiving-code-review, using-git-worktrees, finishing-a-development-branch, writing-skills, using-superpowers, diagnosing-superpowers.

## Step 3: ECC pieces

Copy files by hand. Don't use ECC's plugin or installer.

```bash
cd ~/factory-src
git clone --depth 1 https://github.com/affaan-m/ECC.git
cd ECC

# Skills: whole folders, so any files beside SKILL.md come along
mkdir -p ~/.claude/skills
cp -R skills/security-review skills/verification-loop skills/e2e-testing \
      skills/search-first ~/.claude/skills/
cp -R skills/python-patterns ~/.claude/skills/                       # Python
cp -R skills/coding-standards skills/backend-patterns \
      skills/frontend-patterns ~/.claude/skills/                     # TS / Node / frontend

# Agents
mkdir -p ~/.claude/agents
cp agents/security-reviewer.md agents/e2e-runner.md ~/.claude/agents/
cp agents/python-reviewer.md ~/.claude/agents/                       # Python
cp agents/typescript-reviewer.md agents/build-error-resolver.md ~/.claude/agents/   # TS / Node
cp agents/react-reviewer.md ~/.claude/agents/                        # only if React

# Rules: whole folders, so their relative links keep working
mkdir -p ~/.claude/rules/ecc
cp -R rules/common rules/python rules/typescript ~/.claude/rules/ecc/
rm ~/.claude/rules/ecc/common/agents.md ~/.claude/rules/ecc/common/hooks.md
```

Then:

1. In each copied agent, set `model: inherit` in the frontmatter.
2. Read every copied skill, agent and rule file. Search them for `skill:`, `agent`, `/` commands, `hooks`, `mcp` and `scripts/`, and check each reference against what's installed. Report any that's missing; don't fetch it automatically.
3. Read the remaining files in `rules/ecc/common/`, `rules/ecc/python/` and `rules/ecc/typescript/`. If any tells the agent to use an ECC command, agent or hook that isn't installed, remove that line and tell me.
4. Ask me whether my frontend is React. If not, remove `react-reviewer.md` and `frontend-patterns`.
5. Check which CLIs exist on my machine. Python: `ruff`, `mypy`, `black`, `bandit`, `pytest`. TS/Node: `node`, `npm`, `tsc`, `eslint`, `prettier`, `vitest` or `jest`, `npx playwright`. Don't install them. List the missing ones in the report.

## Step 4: gstack (optional)

Take only gstack's short rules digest, not the tool. Ask me before doing this step.

```bash
cd ~/factory-src
git clone --depth 1 https://github.com/garrytan/gstack.git
printf '\n\n' >> ~/.claude/CLAUDE.md
cat gstack/agents-digest/gstack-AGENTS.md >> ~/.claude/CLAUDE.md
```

- Don't run `./setup` and don't copy any gstack skill folders. Many of its skills rely on the browser and helper tools that `./setup` builds, so copied folders may not work on their own.
- If I later want the full gstack sprint (`/office-hours` → `/plan-eng-review` → `/review` → `/qa` → `/ship`), that's a separate install with its own hooks, and it would replace Superpowers rather than sit beside it.

## Step 5: Verify

1. Run `claude plugin list` and confirm `superpowers` is enabled.
2. List `~/.claude/skills/` and `~/.claude/agents/` and compare them with the "What to take" table: nothing missing, nothing extra.
3. Search `~/.claude/agents/` for `model:` and confirm every copied agent says `inherit`.
4. Show me the end of `~/.claude/CLAUDE.md` so I can check the appended rules.
5. Print the final report from the rules at the top.
6. Tell me to open a **new** Claude Code session and test with:
   - `Let's make a react todo list` — Superpowers should start brainstorming and ask questions before writing code.
   - `Review this repo for security issues` — it should use the `security-reviewer` agent.
   - `/context` — check how much of the context window the skills and rules use at startup.
7. Leave `~/factory-src/` in place, so updating later is a `git pull` and a re-copy.

## Sources

- [obra/superpowers](https://github.com/obra/superpowers)
- [affaan-m/ECC](https://github.com/affaan-m/ECC)
- [garrytan/gstack](https://github.com/garrytan/gstack)
- [multica-ai/andrej-karpathy-skills](https://github.com/multica-ai/andrej-karpathy-skills)
