# Card Game Authoring Skill

This agent skill helps AI coding assistants create, review, and validate portable card and party game packages. It is packaged as a standard `SKILL.md` directory and can be installed in tools that support local agent skills, including Codex and Claude Code.

Use it when converting party or card games into declarative JSON packages, including:

- `game.json` metadata and rules text
- `decksets/*.json` card/content resources
- `runtime-state.schema.json` cross-round state
- `deal-rule.json` declarative deal behavior
- `tests/*.json` validation fixtures
- fixture-based structural validation

The skill is intentionally self-contained. It does not require agents to read any private app repository or copy internal source files.

## Install For Codex

Clone this repository into your Codex skills directory:

```bash
git clone https://github.com/ocplease/card-game-authoring.git ~/.codex/skills/card-game-authoring
```

On Windows PowerShell:

```powershell
git clone https://github.com/ocplease/card-game-authoring.git "$env:USERPROFILE\.codex\skills\card-game-authoring"
```

Restart Codex or start a new session so the skill list refreshes.

## Install For Claude Code

Clone this repository into your Claude Code skills directory:

```bash
git clone https://github.com/ocplease/card-game-authoring.git ~/.claude/skills/card-game-authoring
```

On Windows PowerShell:

```powershell
git clone https://github.com/ocplease/card-game-authoring.git "$env:USERPROFILE\.claude\skills\card-game-authoring"
```

Restart Claude Code or start a new session so the skill list refreshes.

## Use

Ask your agent to use the skill when authoring or reviewing card game packages:

```text
Use card-game-authoring to create a portable JSON game package for Truth Bomb.
```

```text
Use card-game-authoring to review this deal-rule.json and fixtures.
```

The skill should also trigger for requests about card game package authoring, declarative deal rules, deck sets, player-count variants, runtime state, fixtures, or adapting a package to a target game engine.

## Contents

- `SKILL.md`: agent-facing workflow and portable card game package contract.
- `agents/openai.yaml`: optional UI metadata for agents that read OpenAI-style skill metadata.

## Validate

From this skill directory:

```bash
python ~/.codex/skills/.system/skill-creator/scripts/quick_validate.py .
```

If you only have Claude Code installed, use the validator from a local skill-creator skill if available, or validate manually by checking that `SKILL.md` has YAML frontmatter with `name` and `description`.

On Windows PowerShell:

```powershell
python "$env:USERPROFILE\.codex\skills\.system\skill-creator\scripts\quick_validate.py" .
```
