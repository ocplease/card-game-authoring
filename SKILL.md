---
name: card-game-authoring
description: Create, review, validate, publish, or update CardPlay game packages. Use when converting a party/card game into CardPlay, defining declarative JSON deal rules, deck sets, player-count variants, round lifecycle behavior, runtime state, fixture tests, or Supabase game package publishing.
---

# Card Game Authoring

Author CardPlay games as data. Use declarative JSON packages that the trusted CardPlay engine can validate and execute; do not invent game-specific executable scripts unless the platform later adds an approved escape hatch.

## Workflow

1. Locate the CardPlay repo root. Run commands from there.
2. Pick the closest existing package template:

```text
packages/games/src/json/who-is-the-spy
packages/games/src/json/werewolf-role-deal
packages/games/src/json/king-game
packages/games/src/json/ito
```

3. Create or edit one package directory:

```text
my-game/
  game.json
  runtime-state.schema.json
  deal-rule.json
  decksets/
    main.json
  tests/
    4-players.json
    8-players.json
```

4. Validate the package before calling it done.
5. Publish only after validation passes and the Supabase status/version plan is clear.

## Required Files

- `game.json`: identity, name, description, player limits, `rulesText`, `aiContext`, deck set references, runtime schema reference, deal rule reference, and optional `coverImageUrl`.
- `decksets/*.json`: reusable or consumable card/content resources.
- `runtime-state.schema.json`: only persistent fields the engine must carry across rounds.
- `deal-rule.json`: declarative instructions for assigning cards to players.
- `tests/*.json`: fixture cases for supported player counts, asymmetric deals, default card faces, and cross-round behavior.

Minimal `game.json`:

```json
{
  "id": "my-game",
  "name": "My Game",
  "description": "Short picker description.",
  "minPlayers": 3,
  "maxPlayers": 8,
  "rulesText": "Player-facing rules text.",
  "aiContext": "Rules assistant boundaries for this game.",
  "deckSetIds": ["main"],
  "runtimeStateSchema": "runtime-state.schema.json",
  "deal": "deal-rule.json",
  "estimatedMinutes": 10,
  "ageRating": "8+"
}
```

Add `coverImageUrl` only when it is a real reachable image URL. The web game picker displays it first.

## Deal Rules

Every `deal-rule.json` must answer:

1. How many cards does each player receive?
2. Does the distribution or role mix change by player count?
3. Are cards reused, consumed, rotated, or excluded after use?

Supported `pattern` values:

- `one_card_each`: every player receives one card from the same deck.
- `multi_card_each`: every player receives multiple card types, such as one private number plus one shared visible topic.
- `role_distribution_by_player_count`: exact role/card mix changes by player count.
- `spy_word_pair`: most players receive one common word and spies receive a related word.
- `asymmetric_assignments`: different players receive different counts or card types.

Supported `roundLifecycle` values:

- `reusable`: cards can appear again next round.
- `consume_one_per_round`: one deck item is used per round and excluded from later rounds through runtime state.

## Deal Rule Requirements

- Include `defaultFace: "up" | "down"` on every dealt card assignment.
- Use `defaultFace: "down"` for hidden roles, words, numbers, identities, or private instructions.
- Use `defaultFace: "up"` only when a card should start visible to the table.
- Do not add `private`, `public`, or other visibility flags to card definitions. Visibility belongs to the dealt assignment.
- Use equal card counts only when every player truly receives the same number of cards.
- Use asymmetric assignments when players receive different card counts or different card types.
- Define player-count variants for every supported count where the role/card mix changes.
- Use `lifecycle: "reusable"` deck sets for repeatable roles, numbers, or cards.
- Use `lifecycle: "consumable"` deck sets for prompts, word pairs, questions, or content that should not repeat across rounds.
- Keep runtime state minimal. Add fields like `roundNumber`, `usedCardIds`, or `lastKingPlayerId` only when the deal rule needs them.
- Use `imageUrl` for visual cards when available; otherwise provide clear `name` and `description`.

## Pattern Examples

One hidden card each:

```json
{
  "pattern": "one_card_each",
  "deckSetId": "numbers",
  "selection": { "mode": "shuffle", "count": "player_count" },
  "defaultFace": "down",
  "roundLifecycle": "reusable"
}
```

Multiple cards per player with a shared visible card:

```json
{
  "pattern": "multi_card_each",
  "assignments": [
    {
      "deckSetId": "numbers",
      "selection": { "mode": "shuffle", "count": "player_count" },
      "defaultFace": "down"
    },
    {
      "deckSetId": "topics",
      "selection": { "mode": "shuffle", "count": 1 },
      "defaultFace": "up",
      "sharedAcrossPlayers": true
    }
  ],
  "roundLifecycle": "reusable"
}
```

Player-count role variants:

```json
{
  "pattern": "role_distribution_by_player_count",
  "deckSetId": "roles",
  "variants": {
    "8": [
      { "cardId": "werewolf", "count": 2, "defaultFace": "down" },
      { "cardId": "seer", "count": 1, "defaultFace": "down" },
      { "cardId": "doctor", "count": 1, "defaultFace": "down" },
      { "cardId": "villager", "count": 4, "defaultFace": "down" }
    ]
  },
  "roundLifecycle": "reusable"
}
```

Consumable word pair:

```json
{
  "pattern": "spy_word_pair",
  "deckSetId": "word-pairs",
  "spyCount": [
    { "minPlayers": 3, "maxPlayers": 6, "count": 1 },
    { "minPlayers": 7, "maxPlayers": 10, "count": 2 }
  ],
  "assignments": {
    "common_players": { "metadataField": "commonWord", "defaultFace": "down" },
    "spy_players": { "metadataField": "spyWord", "defaultFace": "down" }
  },
  "roundLifecycle": "consume_one_per_round",
  "runtimeState": ["usedCardIds", "roundNumber"]
}
```

Asymmetric card counts:

```json
{
  "pattern": "asymmetric_assignments",
  "selector": {
    "target": "king_player",
    "mode": "random_except_previous",
    "runtimeStateField": "lastKingPlayerId"
  },
  "assignments": [
    {
      "target": "king_player",
      "cards": [
        { "deckSetId": "king-cards", "count": 1, "defaultFace": "up" },
        { "deckSetId": "number-cards", "count": 1, "defaultFace": "down" }
      ]
    },
    {
      "target": "other_players",
      "cards": [
        { "deckSetId": "number-cards", "count": 1, "defaultFace": "down" }
      ]
    }
  ],
  "roundLifecycle": "reusable",
  "runtimeState": ["lastKingPlayerId", "roundNumber"]
}
```

## Fixture Coverage

Add fixtures that prove the real deal risks:

- Minimum supported player count.
- Maximum supported player count.
- Every player-count variant with a different role/card mix.
- Multi-card games with expected per-player counts and mixed visibility.
- Asymmetric games, including which player receives extra cards.
- Face-up and face-down expectations.
- Cross-round behavior for consumed content or rotating roles.

Fixture examples:

```json
{
  "name": "4 players",
  "playerCount": 4,
  "expected": {
    "cardsPerPlayer": [1, 1, 1, 1]
  }
}
```

```json
{
  "name": "king round",
  "playerCount": 4,
  "expected": {
    "cardsPerPlayer": [1, 1, 1, 2],
    "hasMixedVisibility": true
  }
}
```

## Commands

Validate built-ins:

```bash
npm run validate:games
```

Validate one authored package:

```bash
npx tsx packages/game-authoring/src/cli.ts <package-dir>
```

Expected valid-package output:

```text
validated 1 game package(s)
```

Publish draft or published rows:

```bash
npx tsx packages/game-authoring/src/publish-cli.ts <package-dir>
npx tsx packages/game-authoring/src/publish-cli.ts <package-dir> --publish
npx tsx packages/game-authoring/src/publish-cli.ts <package-dir> --upsert
npx tsx packages/game-authoring/src/publish-cli.ts <package-dir> --publish --upsert
```

Publishing requires:

```bash
NEXT_PUBLIC_SUPABASE_URL=...
SUPABASE_SERVICE_ROLE_KEY=...
```

Set `HTTP_PROXY` and `HTTPS_PROXY` when the network requires a proxy.

## Publishing Rules

- The publish CLI validates first, then writes into `game_packages`.
- Without `--upsert`, insert fails if the row already exists.
- With `--upsert`, update the row whose `id` matches `<slug>@1`.
- Current CLI limitation: publishing uses `version: 1`; there is no `--version` flag or automatic archive/publish rotation.
- Published packages appear in `/api/games` only when row `status` is `published`.
- For a published update, validate locally, run `publish-cli.ts <package-dir> --publish --upsert`, read the row back, and execute stored `deal_rule` against representative player counts.
- Do not hand-edit `game_packages.game`, `deck_sets`, `runtime_state_schema`, or `deal_rule` unless copying from a locally validated package.

## Completion Checklist

- `game.json` IDs match `deckSetIds`.
- Every deck set referenced by `deal-rule.json` exists.
- Every dealt card assignment has `defaultFace`.
- The chosen pattern preserves required randomness and card structure.
- Fixtures cover every supported player count or every distinct variant.
- Cross-round state is declared in `runtime-state.schema.json`.
- `npx tsx packages/game-authoring/src/cli.ts <package-dir>` passes.
- If publishing, Supabase env vars are set and the row status/version plan is clear.
- For published updates, database readback confirms stored rows execute as expected.

The package is not complete until schema validation and fixture tests pass.
