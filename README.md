# robust-plan

A [SKILL.md](https://code.claude.com/docs/en/skills)-standard planning skill for coding agents: design **deep, evidence-based implementation and testing plans** for complex features and fixes — invariants, route selection, staged rollout, acceptance criteria, fallback tables, and isolated testing.

Works with **Claude Code, Codex, Cursor, Kilo, OpenCode, Gemini CLI, Copilot, Amp**, and any other harness that reads the open `SKILL.md` agent-skills format.

## What it does

`robust-plan` turns "how should we implement X?" into a rigorous, written `<FEATURE>_IMPLEMENTATION_PLAN.md` that another agent (or a human) can execute without re-deciding anything. It forces a five-phase method:

1. **Gather evidence** — read every touched file plus callers/config; every code claim must cite `file:line`. No planning from memory.
2. **Resolve ambiguity with the user** — asks targeted questions (with recommended defaults) for scope, irreversible, or default-choice decisions; never silently invents an answer.
3. **Choose the route** — enumerates 2–3 genuinely different architectures, compares them in a trade-off table, records why the winner wins and why the rest lose.
4. **Write the plan** — exact section order: Scope and evidence → Goals → Decisions → Route selection → Target architecture + Invariants → per-concern sections (Files / Findings / Changes / Why / Acceptance criteria) → Failure modes and fallbacks → Smaller but definite fixes → Non-goals and shortcomings → Implementation sequence → Required verification → Post-implementation additions → Rollout → Final acceptance criterion.
5. **Quality checklist** — verifies the plan before presenting it, then stays the living source of truth during implementation.

## Why this approach

Most planning skills and "plan first" prompts produce an optimistic bullet list. This skill encodes the habits of real incident/reliability implementation plans:

- **Evidence before design.** Every claim about current code carries a `file:line` citation, so the plan is grounded in the actual codebase instead of the model's memory or plausible-sounding guesses.
- **Invariants before implementation.** Numbered absolute rules (ownership, cleanup, state consistency, bounded terminal states, untouched-by-default behavior) stated up front — each later change must tie back to them.
- **Honesty throughout.** The plan explicitly records what the review did *not* establish, and verification sections read "the following are required tests, not tests already executed." No fake checkmarks.
- **Staged, reversible delivery.** Risk-ordered stages: protect data first, concurrency/scaling last, everything risky behind default-off toggles, each stage with a rollback criterion.
- **Nothing destructive.** Failure handling escalates graceful → bounded retry → quarantine-and-report; the failure-mode table must cover crash, cancel, timeout, partial success, stale state, and ownership-uncertain cases.
- **Testing isolation rules built in.** Temp dirs, mocks, injected config; snapshot-and-restore for anything that must touch the environment; never point tests at production state.
- **Zero dependencies.** One self-contained `SKILL.md` file — no scripts, no runtime, no config. Copy it in, and it works on every agent that supports the standard.

## Install

### Option 1 — skills CLI (recommended, works with 70+ agents)

The open [skills](https://github.com/vercel-labs/skills) CLI installs the skill into whichever agents it detects (Claude Code, Codex, Cursor, Kilo, OpenCode, Gemini CLI, and more):

```bash
npx skills add Xexxxed/robust-plan-skill
```

Useful flags: `-g` for global (user-level) install, `-a <agents...>` to target specific agents (e.g. `-a claude-code codex kilo`), `-y` to skip prompts, `--all` to install everywhere without asking.

### Option 2 — Claude Code plugin marketplace

```text
/plugin marketplace add Xexxxed/robust-plan-skill
/plugin install robust-plan@robust-plan-skill
```

### Option 3 — manual install

Clone and copy the skill folder into your agent's skills directory:

```bash
git clone https://github.com/Xexxxed/robust-plan-skill.git
mkdir -p ~/.claude/skills
cp -r robust-plan-skill/skills/robust-plan ~/.claude/skills/
```

PowerShell (Windows):

```powershell
git clone https://github.com/Xexxxed/robust-plan-skill.git
New-Item -ItemType Directory -Force "$HOME\.claude\skills" | Out-Null
Copy-Item -Recurse robust-plan-skill\skills\robust-plan "$HOME\.claude\skills\"
```

**Where to put it** — project scope (committed with the repo) or personal scope (all your projects):

| Agent | Project scope | Personal scope |
| --- | --- | --- |
| Claude Code | `.claude/skills/` | `~/.claude/skills/` |
| OpenAI Codex | `.agents/skills/` | `~/.codex/skills/` |
| Cursor | `.agents/skills/` or `.cursor/skills/` | `~/.cursor/skills/` |
| Kilo | `.kilo/skills/` (older installs: `.kilocode/skills/`) | `~/.kilo/skills/` |
| OpenCode | `.opencode/skills/` | `~/.config/opencode/skills/` |
| Gemini CLI | `.gemini/skills/` | `~/.gemini/skills/` |
| GitHub Copilot | `.github/skills/` | `~/.copilot/skills/` |
| Amp / anything else | `.agents/skills/` | `~/.config/agents/skills/` |

`.agents/skills/` is the vendor-neutral path most agents check as a fallback — a good choice for mixed-tool teams.

### Updating

Re-run the install command, or for manual installs `git pull` in the clone and re-copy. If you installed via symlink instead of copy, a pull is enough:

```bash
ln -s /path/to/robust-plan-skill/skills/robust-plan ~/.claude/skills/robust-plan
```

## Usage

The skill is model-invoked: once installed, it triggers when you ask for a plan, e.g.

- "Write an implementation plan for adding API-key auth"
- "Plan how to make this pipeline restart-safe"
- "How should we implement multi-tenant uploads?"

In agents that expose skills as slash commands (Claude Code, Kilo, Codex), you can also invoke it directly:

```text
/robust-plan Add resumable downloads to the export service
```

Say "implement the plan" afterwards and the skill keeps the plan file as the living source of truth while executing it stage by stage. Small fixes and one-liners deliberately bypass it — see "When not to use" in the skill.

## Repository layout

```
.claude-plugin/marketplace.json   # Claude Code plugin marketplace manifest
skills/robust-plan/SKILL.md       # the skill (the only file agents load)
```

## License

[MIT](LICENSE)
