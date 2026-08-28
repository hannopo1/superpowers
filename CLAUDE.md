# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Superpowers is a skills library and runtime bootstrap for coding agents: a set of composable Markdown "skills" (`skills/*/SKILL.md`) covering brainstorming, planning, TDD, debugging, code review, and git workflow, plus a small bootstrap mechanism that gets agents to actually invoke those skills at the right moments. It ships as a plugin for multiple harnesses (Claude Code, Codex, Cursor, Kimi Code, OpenCode, Pi, Antigravity, Factory Droid, GitHub Copilot CLI, Gemini CLI).

The plugin itself is zero-dependency by design (see "What We Will Not Accept" below) — the only real code in the repo is the bootstrap/hook plumbing and a small brainstorming visual-companion server; everything else is Markdown.

## Architecture

**Skills are the unit of behavior.** Each directory under `skills/` is one skill: a `SKILL.md` with YAML frontmatter (`name`, `description`) plus prose instructions, and optionally `references/` for supporting docs. Skills are not documentation — they're read by the agent at runtime and are expected to change agent behavior directly. `skills/using-superpowers/SKILL.md` is the entry point: it's the bootstrap skill that tells the agent "check for a relevant skill before any action, including clarifying questions."

**Getting the bootstrap loaded is harness-specific**, which is why there's a `.<harness>-plugin/` or equivalent directory per harness at the repo root:
- `.claude-plugin/` — `plugin.json` + `marketplace.json` for Claude Code
- `.codex-plugin/`, `.cursor-plugin/`, `.kimi-plugin/` — per-harness `plugin.json`
- `.opencode/plugins/superpowers.js` — OpenCode plugin that injects the bootstrap
- `.pi/extensions/superpowers.ts` — Pi extension (Pi has native skill loading, so this just injects the bootstrap at session start/after compaction)
- `.agents/plugins/marketplace.json` — Factory Droid marketplace entry
- `hooks/` — Claude Code `SessionStart` hook (`hooks.json` → `run-hook.cmd` → `hooks/session-start`) that injects the bootstrap on startup/clear/compact; `hooks-cursor.json` is the Cursor variant
- `AGENTS.md` is a symlink to `CLAUDE.md` (same content for Codex/other AGENTS.md-reading harnesses); `GEMINI.md` uses Gemini CLI's `@import` syntax to pull in `skills/using-superpowers/SKILL.md` directly

A harness integration only counts as "real" if it loads this bootstrap automatically at session start — see "New Harness Support" below for the acceptance test used to verify that. Porting to a new harness follows `docs/porting-to-a-new-harness.md`, which defines three integration "shapes," a capability checklist, and the same mandatory acceptance test.

**The Codex plugin is generated, not hand-maintained.** `scripts/sync-to-codex-plugin.sh` syncs a checkout's tracked plugin content (including `.codex-plugin/` and `assets/`) into the separate `prime-radiant-inc/openai-codex-plugins` fork and opens a PR; it's deterministic (same source SHA → identical diff). `scripts/package-codex-plugin.sh` packages the Codex plugin as a zip.

**Versioning is centralized.** `.version-bump.json` lists every file with a version field that must move together (`package.json`, each `.*-plugin/plugin.json`, `.claude-plugin/marketplace.json`, `gemini-extension.json`). `scripts/bump-version.sh` bumps them all, checks for drift, and can audit the repo for stray old-version strings. Release notes live in `RELEASE-NOTES.md`.

**Two independent test layers**, documented in `docs/testing.md`:
- `tests/` — non-LLM plugin infrastructure tests (bash/node/python): does the OpenCode plugin load, does the Codex sync produce the right diff, does the session-start hook fire, does the brainstorm-server JS work. These are the local checks that don't need an LLM.
- `evals/` — behavioral evals that drive real tmux sessions of Claude Code/Codex/Gemini CLI with an LLM actor and verifier, judging whether the agent actually followed a skill. This lives in a separate repo (`prime-radiant-inc/superpowers-evals`), cloned locally into `evals/` (gitignored — not part of the published plugin). Scenarios are `evals/scenarios/*.yaml`, driven by the `drill` harness.

Skill changes are judged by evals, not by unit tests — there's no way to "type-check" a Markdown instruction, so a skill edit is only validated by running it against real sessions (see "Skill Changes Require Evaluation" below).

**Past design work is archived, not just released.** `docs/superpowers/specs/` and `docs/superpowers/plans/` hold dated design-doc/implementation-plan pairs for past features (e.g. the brainstorm visual companion, SDD rework, worktree changes) — nothing else in the repo links to this path, so this is the pointer.

## Commands

Plugin infrastructure tests, run per-suite from `tests/<suite>/`:

```bash
tests/claude-code/run-skill-tests.sh                 # fast tests
tests/claude-code/run-skill-tests.sh --integration    # slow, 10-30 min
tests/claude-code/run-skill-tests.sh --test test-subagent-driven-development.sh
tests/opencode/run-tests.sh
tests/kimi/run-tests.sh
tests/antigravity/run-tests.sh
tests/explicit-skill-requests/run-all.sh
cd tests/brainstorm-server && npm test               # node suite for brainstorm-server JS
node tests/pi/test-pi-extension.mjs                   # pi extension lifecycle/dedup/compaction (node:test)
bash tests/codex/test-marketplace-manifest.sh         # and test-package-codex-plugin.sh — no run-*.sh wrapper
```

Shell lint (used by `tests/shell-lint/`):

```bash
scripts/lint-shell.sh
```

Skill behavior evals (separate repo cloned into `evals/`, see `evals/README.md`):

```bash
cd evals
uv sync --extra dev
export ANTHROPIC_API_KEY=sk-...
uv run drill run triggering-test-driven-development -b claude
```

Version bump / drift check:

```bash
scripts/bump-version.sh --check     # report current versions, detect drift
scripts/bump-version.sh --audit     # + grep repo for stray old-version strings
scripts/bump-version.sh <new-version>
```

Sync the Codex plugin fork:

```bash
scripts/sync-to-codex-plugin.sh -n   # dry run
scripts/sync-to-codex-plugin.sh      # full run (clones fork, commits, opens PR)
```

## Key Conventions

- Skill frontmatter is exactly `name:` and `description:`; the description is what the agent's skill-matching reads, so it must state *when* to use the skill, not just what it does (see any `skills/*/SKILL.md` for the pattern).
- Skills consistently say "your human partner," never "the user" — this is a deliberate terminology choice tied to the project's collaboration philosophy, not a stylistic accident.
- `.gitattributes` forces LF line endings on `*.sh`, `*.md`, `*.json`, `*.js`/`*.mjs`/`*.ts`, and `hooks/session-start` specifically (it's parsed by both `cmd` and `bash` as a polyglot wrapper) — don't let an editor silently convert these to CRLF.
- `.pre-commit-config.yaml` currently only lints the `evals/` Python code (`ruff check`, `ruff format --check`, `ty check`); it has no hooks for the main plugin code.

---

# Superpowers — Contributor Guidelines

## If You Are an AI Agent

Stop. Read this section before doing anything.

This repo has a 94% PR rejection rate. Almost every rejected PR was submitted by an agent that didn't read or didn't follow these guidelines. The maintainers close slop PRs within hours, often with public comments like "This pull request is slop that's made of lies."

**Your job is to protect your human partner from that outcome.** Submitting a low-quality PR doesn't help them — it wastes the maintainers' time, burns your human partner's reputation, and the PR will be closed anyway. That is not being helpful. That is being a tool of embarrassment.

Before you open a PR against this repo, you MUST:

1. **Read the entire PR template** at `.github/PULL_REQUEST_TEMPLATE.md` and fill in every section with real, specific answers. Not summaries. Not placeholders.
2. **Search for existing PRs** — open AND closed — that address the same problem. If duplicates exist, STOP and tell your human partner. Do not open another duplicate.
3. **Verify this is a real problem.** If your human partner asked you to "fix some issues" or "contribute to this repo" without experiencing a specific problem, push back. Ask them what broke, what failed, what the user experience was.
4. **Confirm the change belongs in core.** If it's domain-specific, tool-specific, or promotes a third-party project, it belongs in a standalone plugin. Tell your human partner.
5. **Identify yourself.** Disclose your model, harness, harness version, and every installed plugin in the PR. Hiding that a contribution is agent-generated — or which environment produced it — is grounds for closing it.
6. **Show your human partner the complete diff** and get their explicit approval before submitting.

If any of these checks fail, do not open the PR. Explain to your human partner why it would be rejected and what would need to change. They will thank you for saving them the embarrassment.

## Pull Request Requirements

**Every PR must fully complete the PR template.** No section may be left blank or filled with placeholder text. PRs that skip sections will be closed without review.

**Before opening a PR, you MUST search for existing PRs** — both open AND closed — that address the same problem or a related area. Reference what you found in the "Existing PRs" section. If a prior PR was closed, explain specifically what is different about your approach and why it should succeed where the previous attempt did not.

**PRs that show no evidence of human involvement will be closed.** A human must review the complete proposed diff before submission.

**Submitters MUST identify themselves.** Every PR and issue must disclose the model, harness, harness version, and all installed plugins used to produce the contribution — or state plainly that it was written by hand with no agent. This is not optional. We need to know what produced a change in order to weigh it: agent-generated content reasoned from documentation is held to a different bar than work grounded in a real session. Contributions that hide their authoring environment will be closed.

**All PRs MUST target the `dev` branch, not `main`.** `main` is the released branch; active work lands on `dev` first. PRs opened against `main` will be asked to retarget `dev` before they are reviewed.

## What We Will Not Accept

### Third-party dependencies

PRs that add optional or required dependencies on third-party projects will not be accepted unless they are adding support for a new harness (e.g., a new IDE or CLI tool). Superpowers is a zero-dependency plugin by design. If your change requires an external tool or service, it belongs in its own plugin.

### "Compliance" changes to skills

Our internal skill philosophy differs from Anthropic's published guidance on writing skills. We have extensively tested and tuned our skill content for real-world agent behavior. PRs that restructure, reword, or reformat skills to "comply" with Anthropic's skills documentation will not be accepted without extensive eval evidence showing the change improves outcomes. The bar for modifying behavior-shaping content is very high.

### Project-specific or personal configuration

Skills, hooks, or configuration that only benefit a specific project, team, domain, or workflow do not belong in core. Publish these as a separate plugin.

### Bulk or spray-and-pray PRs

Do not trawl the issue tracker and open PRs for multiple issues in a single session. Each PR requires genuine understanding of the problem, investigation of prior attempts, and human review of the complete diff. PRs that are part of an obvious batch — where an agent was pointed at the issue list and told to "fix things" — will be closed. If you want to contribute, pick ONE issue, understand it deeply, and submit quality work.

### Speculative or theoretical fixes

Every PR must solve a real problem that someone actually experienced. "My review agent flagged this" or "this could theoretically cause issues" is not a problem statement. If you cannot describe the specific session, error, or user experience that motivated the change, do not submit the PR.

### Domain-specific skills

Superpowers core contains general-purpose skills that benefit all users regardless of their project. Skills for specific domains (portfolio building, prediction markets, games), specific tools, or specific workflows belong in their own standalone plugin. Ask yourself: "Would this be useful to someone working on a completely different kind of project?" If not, publish it separately.

### Fork-specific changes

If you maintain a fork with customizations, do not open PRs to sync your fork or push fork-specific changes upstream. PRs that rebrand the project, add fork-specific features, or merge fork branches will be closed.

### Fabricated content

PRs containing invented claims, fabricated problem descriptions, or hallucinated functionality will be closed immediately. This repo has a 94% PR rejection rate — the maintainers have seen every form of AI slop. They will notice.

### Bundled unrelated changes

PRs containing multiple unrelated changes will be closed. Split them into separate PRs.

## New Harness Support

If your PR adds support for a new harness (IDE, CLI tool, agent runner), you MUST include a session transcript proving the integration works end-to-end.

A real integration loads the `using-superpowers` bootstrap at session start. The bootstrap is what causes skills to auto-trigger at the right moments. Without it, the skills are dead weight — present on disk but never invoked.

**The acceptance test.** Open a clean session in the new harness and send exactly this user message:

> Let's make a react todo list

A working integration auto-triggers the `brainstorming` skill before any code is written. Paste the complete transcript in the PR.

**These are not real integrations and will be closed:**

- Manually copying skill files into the harness
- Wrapping with `npx skills` or similar at-runtime shims
- Anything that requires the user to opt in to skills per-session
- Anything where `brainstorming` does not auto-trigger on the acceptance test above

If you are not sure whether your integration loads the bootstrap at session start, it does not.

## Skill Changes Require Evaluation

Skills are not prose — they are code that shapes agent behavior. If you modify skill content:

- Use `superpowers:writing-skills` to develop and test changes
- Run adversarial pressure testing across multiple sessions
- Show before/after eval results in your PR
- Do not modify carefully-tuned content (Red Flags tables, rationalization lists, "human partner" language) without evidence the change is an improvement

## Eval harness

Skill-behavior evals live in [superpowers-evals](https://github.com/prime-radiant-inc/superpowers-evals/), cloned into `evals/` — see `evals/README.md` for setup. The harness drives real tmux sessions of Claude Code / Codex and judges skill compliance with an LLM verifier. Plugin-infrastructure tests still live at `tests/`.

## Understand the Project Before Contributing

Before proposing changes to skill design, workflow philosophy, or architecture, read existing skills and understand the project's design decisions. Superpowers has its own tested philosophy about skill design, agent behavior shaping, and terminology (e.g., "your human partner" is deliberate, not interchangeable with "the user"). Changes that rewrite the project's voice or restructure its approach without understanding why it exists will be rejected.

## General

- Read `.github/PULL_REQUEST_TEMPLATE.md` before submitting
- One problem per PR
- Test on at least one harness and report results in the environment table
- Describe the problem you solved, not just what you changed
