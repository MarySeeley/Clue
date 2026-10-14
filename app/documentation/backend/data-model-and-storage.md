# Data Model & Storage Design

## Overview

This document captures the data structures, storage strategy, and inference model for the Clue companion app, based on the four distinct ways a player can gain information during a game.

---

## The Four Information Types

### 1. Your Initial Hand
Cards dealt to you at the start of the game. These are absolute certainties — you know exactly which cards you hold, and therefore that those cards are not in the case file or in any other player's hand.

### 2. Cards Shown Directly to You
When another player reveals a card to you in response to your suggestion. You know with certainty which card that player holds.

### 3. A Player Cannot Show Any Card
When a player is asked and responds that they have none of the three suggested cards (suspect, weapon, room). This eliminates all three cards from that player's hand in your model.

### 4. A Card Shown to Another Player (Unknown Card)
When a player shows a card to someone else and you do not see which card it was. You know only that the showing player holds at least one of the three suggested cards. Later, if you eliminate two of the three cards from that player through other information, the remaining card is confirmed — this is the key deductive inference this app enables.

---

## Recommended Data Structure

Use an **event-sourced log** (immutable suggestion/ask events) combined with a **derived in-memory candidate matrix** and a **constraint list**.

- **Event log** — full auditable history; supports corrections and replay.
- **Candidate matrix** — compact per-(player, card) inference state rebuilt from the log.
- **Constraint list** — tracks "player has at least one of these cards" for type-4 reveals; drives deductions when cards are later eliminated.

---

## Persistent Tables (Room / SQLite)

All tables are stored locally on-device using Android Room (SQLite). No server or internet connection is required.

### `games`
| Column | Type | Notes |
|---|---|---|
| `id` | TEXT PK | UUID |
| `name` | TEXT | Display name |
| `edition_name` | TEXT NULLABLE | Display name of the themed edition used (e.g., "Harry Potter"); null = standard |
| `saved_edition_id` | TEXT NULLABLE FK | → saved_editions.id; tracks which saved edition was used, if any |
| `variant` | TEXT | Rule variant (e.g., "standard") |
| `started_at` | INTEGER | Epoch ms |

### `players`
| Column | Type | Notes |
|---|---|---|
| `id` | TEXT PK | UUID |
| `game_id` | TEXT FK | → games.id |
| `name` | TEXT | Player name or label |
| `seat_order` | INTEGER | Turn order position; set by drag-to-reorder on setup screen |
| `character_name` | TEXT NULLABLE | The suspect token the player is using (e.g., "Scarlett"); optional |
| `is_local` | INTEGER | 1 if this is the device user |

### `cards`
| Column | Type | Notes |
|---|---|---|
| `id` | TEXT PK | e.g., "scarlet", "candlestick" |
| `type` | TEXT | SUSPECT \| WEAPON \| ROOM |
| `name` | TEXT | Display name |
| `card_index` | INTEGER | Small int for bitset operations |

### `player_hands`
| Column | Type | Notes |
|---|---|---|
| `game_id` | TEXT FK | → games.id |
| `player_id` | TEXT FK | → players.id |
| `card_id` | TEXT FK | → cards.id |

Rows here represent information type 1 (initial hand). Composite PK on `(game_id, player_id, card_id)`.

### `game_card_overrides`
| Column | Type | Notes |
|---|---|---|
| `game_id` | TEXT FK | → games.id |
| `card_id` | TEXT FK | → cards.id |
| `custom_name` | TEXT | Replaces the card's display name for this game (e.g., "Potions Classroom" instead of "Kitchen") |

Composite PK on `(game_id, card_id)`. Only rows that differ from the default card name need to be stored. At display time, join this table and prefer `custom_name` when present.

### `cell_icon_overrides`
| Column | Type | Notes |
|---|---|---|
| `game_id` | TEXT FK | → games.id |
| `player_id` | TEXT FK | → players.id |
| `card_id` | TEXT FK | → cards.id |
| `override_display` | TEXT | One of: `blank` \| `has_card` \| `not_card` \| `possible` |

Composite PK on `(game_id, player_id, card_id)`. Stores manual icon overrides set by the user from the cell detail dialog. The inferred icon is always recomputed from the event log; this table only stores the user’s visual preference. Delete all rows for a game to revert to auto icons.
| Column | Type | Notes |
|---|---|---|
| `id` | TEXT PK | UUID |
| `name` | TEXT | Display name (e.g., "Harry Potter") |
| `created_at` | INTEGER | Epoch ms |

Stores reusable themed editions a user has created. Independent of any specific game.

### `edition_card_overrides`
| Column | Type | Notes |
|---|---|---|
| `edition_id` | TEXT FK | → saved_editions.id |
| `card_id` | TEXT FK | → cards.id |
| `custom_name` | TEXT | Custom name for this card in this edition |

Composite PK on `(edition_id, card_id)`. When the user starts a game using a saved edition, these rows are **copied** into `game_card_overrides` so the game remains correct even if the edition is later edited or deleted.

### `guesses`
| Column | Type | Notes |
|---|---|---|
| `id` | TEXT PK | UUID |
| `game_id` | TEXT FK | → games.id |
| `turn_number` | INTEGER | |
| `guesser_id` | TEXT FK | → players.id |
| `suspect_id` | TEXT FK | → cards.id |
| `weapon_id` | TEXT FK | → cards.id |
| `room_id` | TEXT FK | → cards.id |
| `created_at` | INTEGER | Epoch ms |
| `corrected_at` | INTEGER NULLABLE | Epoch ms of the most recent edit; null = never corrected |
| `correction_note` | TEXT NULLABLE | Optional user note explaining what was fixed |

### `asks`
| Column | Type | Notes |
|---|---|---|
| `id` | TEXT PK | UUID |
| `guess_id` | TEXT FK | → guesses.id |
| `asked_player_id` | TEXT FK | → players.id |
| `asked_index` | INTEGER | Order asked after suggester |
| `response` | TEXT | `none` \| `shown` \| `shown_unknown` |
| `shown_card_id` | TEXT NULLABLE FK | → cards.id; set only when `response = 'shown'` |
| `created_at` | INTEGER | Epoch ms |

`response` values map directly to the four information types:
- `shown` + `shown_card_id` set → type 2 (card shown to you)
- `none` → type 3 (player cannot show)
- `shown_unknown` + `shown_card_id` null → type 4 (card shown to another player)

---

## In-Memory Inference Structures

These are **derived** — rebuilt by replaying the `asks` and `player_hands` tables on app start. Do not persist them separately.

### Candidate Matrix
```
Map<PlayerId, Map<CardId, Candidate>>
```

```kotlin
enum class CandidateStatus { POSSIBLE, KNOWN, EXCLUDED }

data class Candidate(
    val status: CandidateStatus,
    val probability: Float? = null, // null until probabilistic scoring is run
    val evidence: List<Evidence> = emptyList()
)
```

### Evidence (audit trail per candidate entry)
```kotlin
data class Evidence(
    val type: EvidenceType,  // INITIAL | SHOWN_TO_YOU | SHOWN_TO_OTHER | CANNOT_SHOW | DEDUCTION
    val guessId: String? = null,
    val askId: String? = null,
    val shownCardId: String? = null,
    val timestamp: Long
)
```

### Constraint List
```
List<Constraint>
```
```kotlin
data class Constraint(
    val playerId: String,
    val cardIds: MutableSet<String>, // shrinks as cards are excluded for this player
    val guessId: String
)
```
A constraint means "this player holds at least one card from this set." When `cardIds` is trimmed to size 1 by exclusions, that card is confirmed `KNOWN`.

---

## Deduction Algorithm (Fixpoint Loop)

Run after every new event until no state changes occur:

```
1. Apply the new event:
   - player_hands row         → set KNOWN for (player, card); attach INITIAL evidence
   - asks.response=shown      → set KNOWN for (asked_player, shown_card); attach SHOWN_TO_YOU evidence
   - asks.response=none       → set EXCLUDED for (asked_player, each of suspect/weapon/room); attach CANNOT_SHOW evidence
   - asks.response=shown_unknown → add Constraint(asked_player, {suspect, weapon, room})

2. Uniqueness propagation (run after every step that produces a KNOWN):
   For every (player, card) where status = KNOWN:
     set EXCLUDED for all (otherPlayer, card) where otherPlayer ≠ player; attach DEDUCTION evidence.
   Reason: each card exists exactly once — if one player has it, nobody else can.

3. For each Constraint:
   a. Remove cards that are now EXCLUDED for that player from constraint.cardIds
   b. If constraint.cardIds.size == 1 → set KNOWN for (player, that card); attach DEDUCTION evidence
      → immediately re-run step 2 for the newly KNOWN cell

4. For each Card:
   a. Collect all players whose status for this card is POSSIBLE
   b. If only one player remains → set KNOWN; attach DEDUCTION evidence
      → immediately re-run step 2 for the newly KNOWN cell

5. Repeat from step 3 until no state changes occur.
```

---

## Android Storage Details

| Layer | Technology | Purpose |
|---|---|---|
| Persistent store | Room (SQLite `.db` file) | All tables above |
| Active game pointer | `SharedPreferences` | Stores current `game_id` only |
| In-memory model | Kotlin data classes in ViewModel | Candidate matrix + constraints |

The database file lives at:
```
/data/data/com.example.clue/databases/clue.db
```

It is private to the app, requires no permissions, and works fully offline. The app architecture is:

```
Jetpack Compose UI
       ↓
   ViewModel  (holds candidate matrix + constraint list in memory)
       ↓
   Room DAO   (Kotlin @Query interfaces)
       ↓
   SQLite .db (on-device, private storage)
```

---

## Edit History

- 2026-09-29 8:08 PM — Initial draft created from design discussion.
- 2026-09-29 8:08 PM — Added `edition_name` + `saved_edition_id` to `games`; added `character_name` to `players`; added `game_card_overrides` table for themed edition card renaming.
- 2026-09-29 8:25 PM — Added `saved_editions` and `edition_card_overrides` tables for reusable custom editions.
- 2026-09-30 11:30 AM — Added `cell_icon_overrides` table for user manual icon overrides.
- 2026-09-30 11:45 AM — Added uniqueness propagation step to deduction algorithm: KNOWN on any cell immediately excludes all other players for that card.
- 2026-09-30 1:26 PM — Added `corrected_at` and `correction_note` to `guesses` table to support turn editing from the History screen.
