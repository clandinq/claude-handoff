# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

`claude-handoff` is a Claude Code skill plugin that delegates implementation work to the OpenAI Codex CLI. Claude handles planning and review; Codex executes file writes and shell commands. The skill auto-triggers at every implementation decision point to conserve Claude tokens.

## Repository structure

```
claude-handoff/
├── SKILL.md          # The skill definition installed into Claude Code's plugin system
├── settings.json     # Example ~/.claude/settings.json with allow/deny command lists
└── README.md         # Installation guide and architecture docs
```

No build system, test runner, or package manager. This is documentation and configuration only.

## Installation

```bash
mkdir -p ~/.claude/skills/codex-delegate
cp SKILL.md ~/.claude/skills/codex-delegate/SKILL.md
```

Restart Claude Code, then run `/skills` to confirm it appears. Skills must be in a named subdirectory — `~/.claude/skills/<skill-name>/SKILL.md` — not a flat file.

## Codex binary path

`SKILL.md` Step 5 hardcodes a Codex binary path:

```
/Users/cesarlandin/.nvm/versions/node/v24.13.0/bin/codex
```

Before installing, replace this with the output of `which codex` on the target machine.

## Skill architecture

The skill defines a 6-step workflow in `SKILL.md`:

1. **Ask** — One confirmation before any file writes
2. **Assemble permissions** — Read `~/.claude/settings.json`; extract `allow` (auto-approved for Codex) and `deny` (HARD CONSTRAINTS); surface any unlisted commands once for session approval
3. **Plan** — Glob/Grep to find 3–5 relevant files; read only, no writes
4. **Craft prompt** — Build a structured self-contained prompt with SECURITY directive, PROJECT/LANGUAGE/CONVENTIONS, relevant file paths, STEPS, APPROVED COMMANDS, and HARD CONSTRAINTS
5. **Execute** — `codex exec "<prompt>" --sandbox workspace-write --ask-for-approval on-request --cd <project> --output-last-message /tmp/codex-last-msg.md`
6. **Review** — `git diff`, read 1–2 modified files, read `/tmp/codex-last-msg.md`, grep for deny-list violations, report

## Security model

- `--sandbox workspace-write` bounds Codex file access to the project directory
- Deny list enforced twice: in the prompt's HARD CONSTRAINTS and in Claude's post-execution grep
- Codex prompt opens with a SECURITY directive to prevent prompt injection from file contents
- `--ask-for-approval on-request` is a fallback safety layer for commands not in the approved list

**Never use `--sandbox danger-full-access` unless explicitly requested.**

## Editing guidance

Changes to `SKILL.md` are the primary development activity. When modifying the workflow steps, verify:
- Step 2 correctly surfaces only commands absent from both `allow` and `deny` lists
- Step 4 prompt template includes the SECURITY directive at the top
- Step 5 uses the correct sandbox flags
- Step 6 greps for deny-list violations in addition to reviewing `git diff`

`settings.json` is an example file users copy to `~/.claude/settings.json`. The deny list there should reflect the minimum safe baseline that will travel into every Codex prompt as HARD CONSTRAINTS.
