# Card Game Authoring Skill

This Codex skill helps agents create, review, validate, publish, and update CardPlay game packages.

Use it when converting party or card games into CardPlay declarative JSON packages, including:

- `game.json` metadata and rules text
- `decksets/*.json` card/content resources
- `runtime-state.schema.json` cross-round state
- `deal-rule.json` declarative deal behavior
- `tests/*.json` validation fixtures
- Supabase `game_packages` publish/update workflows

## Install

Clone this repository into your Codex skills directory:

```bash
git clone https://github.com/ocplease/card-game-authoring.git ~/.codex/skills/card-game-authoring
```

On Windows PowerShell:

```powershell
git clone https://github.com/ocplease/card-game-authoring.git "$env:USERPROFILE\.codex\skills\card-game-authoring"
```

Restart Codex or start a new session so the skill list refreshes.

## Use

Ask Codex to use the skill when authoring or reviewing CardPlay game packages:

```text
Use card-game-authoring to create a CardPlay package for Truth Bomb.
```

```text
Use card-game-authoring to review this deal-rule.json and fixtures.
```

The skill should also trigger for requests about CardPlay game package authoring, declarative deal rules, deck sets, player-count variants, runtime state, fixtures, or publishing packages to Supabase.

## Contents

- `SKILL.md`: agent-facing workflow and CardPlay authoring rules.
- `agents/openai.yaml`: UI metadata for Codex skill discovery.

## Validate

From this skill directory:

```bash
python ~/.codex/skills/.system/skill-creator/scripts/quick_validate.py .
```

On Windows PowerShell:

```powershell
python "$env:USERPROFILE\.codex\skills\.system\skill-creator\scripts\quick_validate.py" .
```
