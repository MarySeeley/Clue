# Frontend: Start a Game

## Overview

This document covers the UI, state model, and Compose component design for the "Start a Game" setup flow. This is the first screen a user interacts with before any guesses are logged. It collects player names, optional character assignments, seating order (via drag-to-reorder), and optional card name customizations for themed editions.

---

## Screen Flow

```
App Launch
    └── Game List Screen
            └── [+ New Game] button
                    └── New Game Setup Screen  ← this doc
                            └── [Start Game] button
                                    └── Game Board / Guess Log Screen
```

---

## New Game Setup Screen

### Sections (top to bottom)

1. **Game name** — text field, defaults to "My Clue Game".
2. **Edition picker** — optional. A dropdown lets the user pick a previously saved edition or choose "New custom edition". Selecting a saved edition pre-fills all card custom names. Choosing "New custom edition" shows a name field and the full card customization section with an option to save it for future games.
3. **Player list** — each row has a name field, optional character field, and a drag handle. "You" (local player) is always the first row and cannot be removed.
4. **Add Player button** — appends a new blank player row (max 6 players total).
5. **Card customization section** — shown only when edition toggle is on. Lists all 21 cards grouped by type (Suspect, Weapon, Room) with an optional custom name field next to each default name.
6. **Start Game button** — validates and saves the game, then navigates to the Guess Log screen.

---

## State Model

```kotlin
data class PlayerDraft(
    val id: String = UUID.randomUUID().toString(),
    val name: String = "",
    val characterName: String = "",   // empty string = not set
    val isLocalPlayer: Boolean = false
)

data class CardOverrideDraft(
    val cardId: String,
    val type: CardType,               // SUSPECT | WEAPON | ROOM
    val defaultName: String,
    val customName: String = ""       // empty string = use default
)

data class SavedEditionSummary(
    val id: String,
    val name: String
)

data class NewGameUiState(
    val gameName: String = "My Clue Game",
    val savedEditions: List<SavedEditionSummary> = emptyList(), // loaded from DB on screen open
    val selectedEditionId: String? = null,    // null = standard, "new" = creating new
    val newEditionName: String = "",
    val saveNewEdition: Boolean = false,       // whether to persist the new edition for reuse
    val players: List<PlayerDraft> = listOf(PlayerDraft(isLocalPlayer = true)),
    val cardOverrides: List<CardOverrideDraft> = emptyList(),
    val isCustomEdition: Boolean = false,
    val validationError: String? = null
)
```

---

## ViewModel

```kotlin
class NewGameViewModel(
    private val gameRepository: GameRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(NewGameUiState())
    val uiState: StateFlow<NewGameUiState> = _uiState.asStateFlow()

    fun onGameNameChange(name: String) { ... }
    fun onEditionSelected(editionId: String?) { ... }   // null = standard, "new" = new custom
    fun onNewEditionNameChange(name: String) { ... }
    fun onSaveNewEditionToggle(save: Boolean) { ... }
    fun onPlayerNameChange(index: Int, name: String) { ... }
    fun onPlayerCharacterChange(index: Int, character: String) { ... }
    fun onPlayerReorder(from: Int, to: Int) { ... }   // called by drag-to-reorder
    fun onAddPlayer() { ... }
    fun onRemovePlayer(index: Int) { ... }            // not allowed for index 0 (local player)
    fun onCardCustomNameChange(cardId: String, customName: String) { ... }
    fun onStartGame(onSuccess: (gameId: String) -> Unit) { ... }
}
```

`onStartGame` validates inputs, writes to Room via `GameRepository`, then calls `onSuccess` with the new game ID to trigger navigation.

---

## Compose Components

### `NewGameScreen`

Root screen composable. Hosts a `Scaffold` with a top app bar ("New Game") and a scrollable `Column` containing all sections. Collects `uiState` from the ViewModel via `collectAsStateWithLifecycle()`.

### `GameNameField`

A single `OutlinedTextField` for the game name. Shows an error label if blank on submit.

### `EditionSection`

```
Edition:  [ Standard (none) ▾ ]        ← DropdownMenu
```

Dropdown options:
- **Standard (none)** — no customization, card overrides section hidden.
- **[Saved edition name]** — one entry per saved edition; selecting one pre-fills `cardOverrides` from that edition's stored overrides.
- **New custom edition…** — reveals `NewEditionForm`.

### `NewEditionForm`

Shown below the dropdown when "New custom edition…" is selected:

```
Edition name:  [________________________]
[ ] Save this edition for future games
```

- `OutlinedTextField` for the edition name (required if saving).
- `Checkbox` + label for save-for-later. When unchecked the overrides apply to this game only and are not persisted to `saved_editions`.
- `CardCustomizationSection` is always shown beneath this form when a custom edition is active.

### `PlayerListSection`

A `Column` containing a `ReorderableLazyColumn` (see Drag-to-Reorder below) of `PlayerRow` items plus an "Add Player" `TextButton` at the bottom.

### `PlayerRow`

```
[≡]  Name: [___________]   Character: [___________]  [✕]
```

- Drag handle icon on the left (`≡`) — triggers drag reorder.
- `OutlinedTextField` for name (required).
- `OutlinedTextField` for character (optional, placeholder "Optional").
- Remove button (`✕`) hidden for the local player row (index 0).

### `CardCustomizationSection`

Shown only when `isCustomEdition = true`. Displays three sub-sections (Suspects, Weapons, Rooms), each with a header and a list of `CardOverrideRow` items.

### `CardOverrideRow`

```
Knife  →  [________________]  (placeholder: "Custom name")
```

- Default card name on the left (read-only).
- `OutlinedTextField` on the right for the custom name. Empty = keep default.

### `StartGameButton`

A full-width `Button` at the bottom. Disabled if fewer than 2 players or if any player name is blank.

---

## Drag-to-Reorder

Use the [`sh.calvin.reorderable`](https://github.com/Calvin-LL/Reorderable) library for Jetpack Compose `LazyColumn` drag-to-reorder support.

```kotlin
// build.gradle.kts
implementation("sh.calvin.reorderable:reorderable:<latest>")
```

```kotlin
@Composable
fun PlayerListSection(
    players: List<PlayerDraft>,
    onReorder: (from: Int, to: Int) -> Unit,
    ...
) {
    val lazyListState = rememberLazyListState()
    val reorderState = rememberReorderableLazyListState(lazyListState) { from, to ->
        // index 0 (local player) must stay fixed — guard in ViewModel
        onReorder(from.index, to.index)
    }

    LazyColumn(state = lazyListState) {
        itemsIndexed(players, key = { _, p -> p.id }) { index, player ->
            ReorderableItem(reorderState, key = player.id) { isDragging ->
                PlayerRow(
                    player = player,
                    dragHandle = { ReorderableItemScope.DragHandle() },
                    isDragging = isDragging,
                    ...
                )
            }
        }
    }
}
```

The local player row (index 0) should visually suppress the drag handle and the ViewModel should reject any reorder that moves index 0.

---

## Validation Rules

| Rule | Error message |
|---|---|
| Game name is blank | "Please enter a game name." |
| Fewer than 2 players | "Add at least one other player." |
| Any player name is blank | "All players must have a name." |
| Duplicate player names | "Player names must be unique." |
| "New custom edition" selected but name blank and save checked | "Enter a name to save this edition." |

Validation runs on "Start Game" tap. Errors appear inline below the relevant field or as a `Snackbar` for game-level errors.

---

## Persistence on Start

When the user taps "Start Game", `NewGameViewModel.onStartGame()` performs these writes inside a single Room transaction:

1. If `saveNewEdition = true` and `selectedEditionId == "new"`: insert a `saved_editions` row and its `edition_card_overrides` rows first; capture the new edition ID.
2. Insert `games` row — set `edition_name` to the edition name and `saved_edition_id` to the saved edition’s ID (or null for standard).
3. Insert one `players` row per player, with `seat_order` = list index and `character_name` if provided.
4. Insert `game_card_overrides` rows for any card where `customName` is non-empty, copying from the saved edition or the new-edition form. This snapshot ensures the game is unaffected by future edits to the saved edition.
5. Store the new `game_id` in `SharedPreferences` as the active game.

---

## Navigation

```
NewGameScreen --[onStartGame success]--> GuessLogScreen(gameId)
NewGameScreen --[back]--> GameListScreen
```

Use the Navigation Compose `NavController`:
```kotlin
navController.navigate("guess_log/$gameId") {
    popUpTo("new_game") { inclusive = true }
}
```

---

## Edit History

- 2026-09-29 8:25 PM — Initial draft created.
- 2026-09-29 8:35 PM — Added saved edition picker, `NewEditionForm`, `SavedEditionSummary` state, and save-for-later persistence flow.
