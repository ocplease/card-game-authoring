---
name: card-game-authoring
description: Create, review, validate, submit, approve, or publish portable card and party game packages using declarative JSON. Use when converting tabletop, social deduction, role, prompt, number, or custom card games into a self-contained game package with metadata, deck sets, runtime state, deal rules, and fixture tests, especially approval-based submission without service-role credentials. Avoids repository-specific assumptions and does not require reading source code from any private app.
---

# Card Game Authoring

Create self-contained card game packages that an app or game engine can validate and deal without custom game-specific code. Treat each game as data: metadata, decks, runtime state, deal rules, and fixtures.

Do not assume access to a private repository. Do not tell the user to copy internal source files. If a target app has its own CLI or schema, adapt this portable contract to that app only after the user provides those public details.

Hard rule: do not write generated games into an app's built-in/source game directory, such as `games/`, `packages/games/`, `src/games/`, or similar source-controlled fallback directories, unless the user explicitly asks for source-code fallback files. Prefer approval-based submission to the target app or return a self-contained JSON payload.

## Delivery Modes

Choose the safest mode for the user's request:

- **Approval-based submission:** Build and validate the package, then submit it to the target app's submission API with a low-privilege submit token. Use this when agents should not receive a Supabase service-role key. The server should validate again before inserting and store valid submissions as `pending_approval`.
- **JSON payload:** Return the complete package JSON objects in the response or a neutral export file outside app source directories. Use this when the user wants reviewable artifacts but has not provided database credentials.
- **Temporary validation workspace:** If a validator requires files, write only to a temporary/staging directory outside built-in game source directories, then delete or clearly identify the temporary files.
- **Source fallback files:** Only write into a game's source repository when the user explicitly requests source-controlled fallback packages.

Never place generated games under built-in game directories by default.

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
7. Produce the package as an in-memory payload, response payload, approval submission, database row, or neutral export directory according to the delivery mode.
8. Add fixtures for minimum, maximum, and high-risk player counts.
9. Validate by checking references, counts, visibility, and fixture expectations.
10. If submitting or publishing, ensure validation runs before any database insert. If validation fails, return the exact errors so the package can be improved and resubmitted.

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
- `language`: primary language of the package content, usually one of the target app's supported language codes such as `"en"`, `"ja"`, or `"zh"`.
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
  "language": "en",
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
- `game.json.language` is present when the target app ranks or filters games by language, and uses a supported language code.
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

## Supabase-Only Submission

When the user says "Supabase only", "do not add to the repo", or similar, treat this as a hard boundary:

1. Create the game package in a temporary/staging directory outside source-controlled built-in game directories.
2. Validate that temporary package before any network request.
3. Submit through the target app's approval submission endpoint with a low-privilege token.
4. Confirm the server stored the package as `pending_approval`.
5. Check `git status` or otherwise confirm no repo files were changed.
6. Report the pending row id, status, checksum, validation result, and any issues encountered. Do not report secrets.

## Approval-Based Submission

Use this mode when agents should be able to create game packages without receiving the Supabase service-role key.

Expected flow:

1. Generate the package payload.
2. Validate locally with the available validator or the manual checklist.
3. Submit to the target app's submission endpoint with a low-privilege submit token, not a service-role key.
4. The server validates the same package again before any insert.
5. If validation fails, the server returns a structured error response and creates no database row.
6. If validation passes, the server inserts the row as `pending_approval`.
7. A maintainer later approves the pending row with service-role credentials; approval should revalidate the stored row, archive any currently published row for the same slug, then mark the selected row as `published`.

CardPlay-compatible submission command:

```bash
CARDPLAY_GAME_SUBMIT_TOKEN=... npx tsx packages/game-authoring/src/submit-cli.ts path/to/my-game --url https://your-cardplay-app.example
```

CardPlay-compatible validation failure response:

```json
{
  "error": "Game package validation failed",
  "validation": {
    "valid": false,
    "checkedGames": ["my-game"],
    "errors": ["deal rule references missing deck set prompts"]
  }
}
```

CardPlay-compatible success response:

```json
{
  "id": "my-game@2",
  "slug": "my-game",
  "version": 2,
  "status": "pending_approval",
  "checksum": "<sha256>"
}
```

Maintainer approval commands, when the target app exposes them:

```bash
npx tsx packages/game-authoring/src/publish-cli.ts --list-pending
npx tsx packages/game-authoring/src/publish-cli.ts --approve my-game@2
```

Pending approval packages should not appear in public game lists or become playable until approval changes their status to `published`.

## Maintainer Approval Details

Direct agent insert/upsert into `game_packages` should be disabled for CardPlay-style workflows. Agents submit packages as `pending_approval`; a maintainer-only approval command or service later revalidates the stored row, archives the currently published row for the same slug, and marks the selected pending version as `published`.

Maintainer approval may require server credentials such as:

```bash
NEXT_PUBLIC_SUPABASE_URL=...
SUPABASE_SERVICE_ROLE_KEY=...
```

Never print or commit credentials.

Recommended table contract:

```sql
create table game_packages (
  id text primary key,
  slug text not null,
  version integer not null,
  status text not null check (status in ('draft', 'pending_approval', 'published', 'archived')),
  image_url text,
  game jsonb not null,
  deck_sets jsonb not null,
  runtime_state_schema jsonb not null,
  deal_rule jsonb not null,
  fixtures jsonb not null default '[]',
  validation_report jsonb,
  checksum text not null,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  published_at timestamptz
);
```

Recommended pending submission row payload:

```json
{
  "id": "truth-bomb@1",
  "slug": "truth-bomb",
  "version": 1,
  "status": "pending_approval",
  "image_url": null,
  "game": {},
  "deck_sets": [],
  "runtime_state_schema": {},
  "deal_rule": {},
  "fixtures": [],
  "validation_report": {},
  "checksum": "<sha256>"
}
```

Create `checksum` from a stable JSON serialization of `game`, `deck_sets`, `runtime_state_schema`, `deal_rule`, and `fixtures`. If stable serialization tooling is unavailable, still submit or publish only after manual validation, and include a validation report that says the checksum method used.

Minimal JavaScript row shape for server-side approval implementations:

```js
import { createHash } from "node:crypto";
import { createClient } from "@supabase/supabase-js";

function stable(value) {
  if (Array.isArray(value)) return value.map(stable);
  if (!value || typeof value !== "object") return value;
  return Object.fromEntries(Object.entries(value).sort(([a], [b]) => a.localeCompare(b)).map(([key, item]) => [key, stable(item)]));
}

const pkg = { game, deck_sets, runtime_state_schema, deal_rule, fixtures };
const checksum = createHash("sha256").update(JSON.stringify(stable(pkg))).digest("hex");
const row = {
  id: `${game.id}@1`,
  slug: game.id,
  version: 1,
  status: "pending_approval",
  image_url: game.coverImageUrl ?? null,
  game,
  deck_sets,
  runtime_state_schema,
  deal_rule,
  fixtures,
  validation_report,
  checksum,
  updated_at: new Date().toISOString(),
  published_at: null
};

// The approval service should write pending rows or approve existing pending rows.
// Do not expose this credential to game-authoring agents.
```

Before inserting a pending row or approving one, run the available validator. If it fails, stop and report the validation errors; do not create or publish a row.

### Submission Troubleshooting

Record and report the problems encountered during submission, even after the final upload succeeds. This makes the run reproducible for maintainers and future agents.

Common issues and fixes:

- **Accidentally started writing to a built-in/source game directory:** stop, remove only the files/directories created for this attempt, confirm the worktree is clean, then recreate the package in a temporary directory.
- **Submit token not visible in the shell:** look for project-local env files only when appropriate, load `CARDPLAY_GAME_SUBMIT_TOKEN` into the process without printing it, and never commit env files.
- **Submission endpoint returns validation errors:** print the validation errors and do not retry unchanged payloads.
- **HTML instead of JSON:** verify the app URL is the application host, not the Supabase project URL, then retry the submission endpoint.

After publishing or submitting:

- Read the row back by `id`.
- Confirm `status`, `checksum`, and JSON columns match the intended payload.
- If the app exposes a game-list API, verify the game appears only when `status` is `published`; `pending_approval`, `draft`, and `archived` rows must stay hidden.
- Report the row ID, status, checksum, and validation result. Do not report secrets.

## Adapting To A Target App

If the user has a specific app or engine:

1. Ask for its public schema, validator, or import format if it differs from this contract.
2. Map this package contract to the target format.
3. Prefer approval-based submission or a self-contained JSON payload over source fallback files.
4. Do not include secrets, service-role keys, internal database names, private paths, or unpublished repository details.

If a target app provides a validator, run it and report exact pass/fail output. If no validator exists, use the manual checklist and clearly say validation was structural/manual.
