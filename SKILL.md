---
name: card-game-authoring
description: Create, review, or validate portable card and party game packages using declarative JSON. Use when converting tabletop, social deduction, role, prompt, number, or custom card games into a self-contained game package with metadata, deck sets, runtime state, deal rules, and fixture tests. Avoids repository-specific assumptions and does not require reading source code from any private app.
---

# Card Game Authoring

Create self-contained card game packages that an app or game engine can validate and deal without custom game-specific code. Treat each game as data: metadata, decks, runtime state, deal rules, and fixtures.

Do not assume access to a private repository. Do not tell the user to copy internal source files. If a target app has its own CLI or schema, adapt this portable contract to that app only after the user provides those public details.

## Output Layout

Create one directory per game:

```text
my-game/
  game.json
  runtime-state.schema.json
  deal-rule.json
  decksets/
    main.json
  tests/
    4-players.json
```

Use stable lowercase URL-safe IDs such as `truth-bomb`, `spy-words`, or `number-ranking`.

## Authoring Workflow

1. Explain the real-world rules in plain language.
2. Identify what each player receives at deal time.
3. Decide whether every player receives the same number of cards.
4. Decide whether role/card mixes change by player count.
5. Decide whether decks are reusable or consumable across rounds.
6. Define the smallest runtime state needed for cross-round memory.
7. Write the JSON files.
8. Add fixtures for minimum, maximum, and high-risk player counts.
9. Validate by checking references, counts, visibility, and fixture expectations.

## `game.json`

Purpose: game identity, picker metadata, player limits, player-facing rules, and references to package files.

Required fields:

- `id`: stable URL-safe identifier.
- `name`: display name.
- `description`: short picker/summary text.
- `minPlayers`: minimum player count.
- `maxPlayers`: maximum player count.
- `rulesText`: concise player-facing rules.
- `aiContext`: boundaries and rules for an assistant explaining the game.
- `deckSetIds`: IDs of deck sets used by the game.
- `runtimeStateSchema`: usually `"runtime-state.schema.json"`.
- `deal`: usually `"deal-rule.json"`.

Optional fields:

- `estimatedMinutes`: approximate round/game length.
- `ageRating`: simple label such as `"8+"` or `"13+"`.
- `coverImageUrl`: real reachable image URL for game pickers.

Example:

```json
{
  "id": "truth-bomb",
  "name": "Truth Bomb",
  "description": "Players receive secret prompts and reveal answers together.",
  "minPlayers": 3,
  "maxPlayers": 10,
  "rulesText": "Each player receives one hidden prompt. Take turns answering honestly or passing. Start a new round for fresh prompts.",
  "aiContext": "Explain only the rules and prompt flow. Do not invent scoring unless the user asks for a variant.",
  "deckSetIds": ["prompts"],
  "runtimeStateSchema": "runtime-state.schema.json",
  "deal": "deal-rule.json",
  "estimatedMinutes": 15,
  "ageRating": "13+"
}
```

## Deck Sets

Purpose: define cards, roles, words, numbers, prompts, or other dealable content.

Required deck set fields:

- `id`: must match `game.json.deckSetIds` and deal-rule references.
- `name`: display name.
- `lifecycle`: `"reusable"` or `"consumable"`.
- `cards`: array of card/content objects.

Common card fields:

- `id`: stable unique ID within the deck set.
- `name`: display text.
- `description`: optional explanation.
- `metadata`: optional structured data, such as word pairs.
- `imageUrl`: optional real reachable image URL for visual cards.
- `shareable`: optional boolean for cards intentionally shared across players.

Reusable roles:

```json
{
  "id": "roles",
  "name": "Roles",
  "lifecycle": "reusable",
  "cards": [
    { "id": "villager", "name": "Villager", "description": "Find the hidden threat." },
    { "id": "seer", "name": "Seer", "description": "Receives special information by table rules." },
    { "id": "werewolf", "name": "Werewolf", "description": "Stay hidden from the village." }
  ]
}
```

Consumable word pairs:

```json
{
  "id": "word-pairs",
  "name": "Word Pairs",
  "lifecycle": "consumable",
  "cards": [
    {
      "id": "apple-pear",
      "name": "Apple / Pear",
      "metadata": { "commonWord": "Apple", "spyWord": "Pear" }
    }
  ]
}
```

Do not put private/public visibility on cards. Visibility belongs to the dealt assignment through `defaultFace`.

## Runtime State Schema

Purpose: declare cross-round fields the engine may persist.

Use the smallest set possible. Common fields:

- `roundNumber`: current round index.
- `usedCardIds`: consumed cards already used.
- `lastSpecialPlayerId`: previous special-role target, such as previous king.

Minimal reusable game:

```json
{
  "fields": {
    "roundNumber": { "type": "number", "default": 0 }
  }
}
```

Consumable content:

```json
{
  "fields": {
    "roundNumber": { "type": "number", "default": 0 },
    "usedCardIds": { "type": "array", "items": "string", "default": [] }
  }
}
```

## Deal Rules

Purpose: define how cards are assigned to players.

Every deal rule must answer:

1. How many cards does each player receive?
2. Does distribution change by player count?
3. Are cards reused or consumed across rounds?
4. Does each dealt card start face up or face down?

Required visibility rule:

- Every dealt card assignment must include `defaultFace: "up"` or `defaultFace: "down"`.
- Use `"down"` for secrets: roles, hidden words, numbers, identities, or private prompts.
- Use `"up"` only when the table should immediately see the card.

Supported patterns:

- `one_card_each`
- `multi_card_each`
- `role_distribution_by_player_count`
- `spy_word_pair`
- `asymmetric_assignments`

Supported lifecycle values:

- `reusable`
- `consume_one_per_round`

### `one_card_each`

Use when every player gets one hidden card from the same deck.

```json
{
  "pattern": "one_card_each",
  "deckSetId": "prompts",
  "selection": { "mode": "shuffle", "count": "player_count" },
  "defaultFace": "down",
  "roundLifecycle": "reusable"
}
```

### `multi_card_each`

Use when every player receives the same structure of multiple cards.

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

### `role_distribution_by_player_count`

Use when role/card mix changes by player count.

```json
{
  "pattern": "role_distribution_by_player_count",
  "deckSetId": "roles",
  "variants": {
    "5": [
      { "cardId": "werewolf", "count": 1, "defaultFace": "down" },
      { "cardId": "seer", "count": 1, "defaultFace": "down" },
      { "cardId": "villager", "count": 3, "defaultFace": "down" }
    ],
    "8": [
      { "cardId": "werewolf", "count": 2, "defaultFace": "down" },
      { "cardId": "seer", "count": 1, "defaultFace": "down" },
      { "cardId": "villager", "count": 5, "defaultFace": "down" }
    ]
  },
  "roundLifecycle": "reusable"
}
```

For each variant, the sum of `count` values must equal the player count.

### `spy_word_pair`

Use when most players receive one word and spies receive a related different word.

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

The referenced deck cards must contain the metadata fields named in `assignments`.

### `asymmetric_assignments`

Use when one player or group receives a different number/type of cards.

```json
{
  "pattern": "asymmetric_assignments",
  "selector": {
    "target": "leader_player",
    "mode": "random_except_previous",
    "runtimeStateField": "lastSpecialPlayerId"
  },
  "assignments": [
    {
      "target": "leader_player",
      "cards": [
        { "deckSetId": "leader-cards", "count": 1, "defaultFace": "up" },
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
  "runtimeState": ["lastSpecialPlayerId", "roundNumber"]
}
```

Supported selector modes:

- `random`: choose any eligible target.
- `random_except_previous`: avoid the previous target when possible; requires `runtimeStateField`.

## Fixtures

Purpose: prove the package deals correctly.

Each fixture should include:

- `name`
- `playerCount`
- `expected`

Recommended expected fields:

- `cardsPerPlayer`: array of expected card counts.
- `hasMixedVisibility`: true when both up and down cards are expected.
- `faces`: optional list of expected `defaultFace` values.
- `usesDeckSetIds`: optional deck set IDs expected in the deal.
- `consumesCard`: true when a round should mark content as used.

Minimum fixture:

```json
{
  "name": "4 players",
  "playerCount": 4,
  "expected": {
    "cardsPerPlayer": [1, 1, 1, 1]
  }
}
```

Asymmetric fixture:

```json
{
  "name": "leader gets an extra visible card",
  "playerCount": 4,
  "expected": {
    "cardsPerPlayer": [1, 1, 1, 2],
    "hasMixedVisibility": true
  }
}
```

## Manual Validation Checklist

Use this checklist even when no validator CLI exists:

- `game.json.id` is stable and URL-safe.
- `game.json.deckSetIds` exactly match `decksets/*.json` IDs.
- Every deck set referenced by `deal-rule.json` exists.
- Every referenced `cardId` exists in the referenced deck set.
- Every dealt card assignment has `defaultFace`.
- Player counts stay within `minPlayers` and `maxPlayers`.
- For role variants, each variant total equals its player count.
- Equal-card patterns are used only when every player truly receives the same structure.
- Asymmetric patterns are used when players receive different counts/types.
- Consumable content declares state such as `usedCardIds`.
- Runtime state schema declares every field named by `deal-rule.json.runtimeState`.
- Fixtures cover minimum, maximum, and each distinct variant/risk.

## Adapting To A Target App

If the user has a specific app or engine:

1. Ask for its public schema, validator, or import format if it differs from this contract.
2. Map this package contract to the target format.
3. Keep the source package self-contained so it can be reviewed without private code.
4. Do not include secrets, service-role keys, internal database names, private paths, or unpublished repository details.

If a target app provides a validator, run it and report exact pass/fail output. If no validator exists, use the manual checklist and clearly say validation was structural/manual.
