# Frontend: Game Board (Main Game Screen)

## Overview

The Game Board is the primary screen during an active game. Its centerpiece is a scrollable grid showing the relationship between every player and every card, with icons reflecting the current inference state. The grid updates automatically after each logged guess.

---

## Screen Layout

**Overall structure:**
```
TopAppBar:  [← Back]   My Clue Game   [⋮ Menu]
────────────────────────────────────────────────
  Grid (horizontally + vertically scrollable)
────────────────────────────────────────────────
                            [+ Log Guess]  ← FAB
```

**Grid example (4 players):**

| Card | You | Alice | Bob | Carol |
|---|:---:|:---:|:---:|:---:|
| **— SUSPECTS —** | | | | |
| ✓ Scarlett | ✓ | ✗ | ✗ | ✗ |
| ✓ Mustard | ✗ | ✓ | ✗ | ✗ |
| ★ Plum | ✗ | ✗ | ✗ | ✗ |
| **— WEAPONS —** | | | | |
| ? Knife | ✗ | | % | ✗ |
| **— ROOMS —** | | | | |
| ✓ Kitchen | ✓ | ✗ | ✗ | ✗ |

- **Left column** (card names) — pinned, does not scroll horizontally.
- **Top row** (player names) — pinned, does not scroll vertically.
- **Grid body** — scrolls both horizontally (more players) and vertically (more cards).
- **FAB** — opens the Log Guess sheet.

---

## Card Name Solution Indicators

Each card name in the pinned left column carries an icon showing whether the card is still a candidate for the case file (the winning cards). Icons are subject to change.

| Indicator | Meaning | Derivation |
|---|---|---|
| *(none / neutral)* | Unknown — could be held by a player or be in the case file | No player is `KNOWN` for this card |
| ✓ (accounted for) | Confirmed held by a player — not in the case file | At least one player has `CellDisplay.HAS_CARD` |
| ★ (solution) | Confirmed in the case file — this IS one of the winning cards | All players have `CellDisplay.NOT_CARD` |

These indicators let the player scan down the card list at a glance to see which cards are still unaccounted for across all three types (suspect, weapon, room) and therefore still candidates for the solution.

## Cell Status Icons

Icons displayed inside each grid cell (per player per card). Icons are subject to change.
|---|---|---|
| No information | *(blank)* | No evidence for or against this player holding this card |
| Has the card | ✓ | Confirmed in this player's hand |
| Does not have the card | ✗ | Confirmed not in this player's hand |
| Possible | % | Active constraint exists linking this player to this card — they may hold it |

### Mapping from Inference State

| `CandidateStatus` | Evidence condition | Display icon |
|---|---|---|
| `KNOWN` | any | ✓ |
| `EXCLUDED` | any | ✗ |
| `POSSIBLE` | no constraint references this (player, card) | *(blank)* |
| `POSSIBLE` | at least one active `Constraint` includes this card for this player | % |

The % icon is driven by the `Constraint` list in the ViewModel — if any constraint for a player still has this card in its `cardIds` set, show %. This makes the distinction meaningful: blank means "we have no reason to suspect them" while % means "they showed something from a guess that included this card."

---

## State Model

```kotlin
data class CellState(
    val playerId: String,
    val cardId: String,
    val inferredDisplay: CellDisplay,  // always computed by the inference engine
    val manualDisplay: CellDisplay?,   // null = not overridden by user
) {
    val effectiveDisplay: CellDisplay get() = manualDisplay ?: inferredDisplay
    val isManualOverride: Boolean get() = manualDisplay != null
}

enum class CellDisplay { BLANK, HAS_CARD, NOT_CARD, POSSIBLE }

// Reflects whether this card is still a candidate for the case file
enum class CardSolutionStatus { UNKNOWN, ACCOUNTED_FOR, CONFIRMED_SOLUTION }

data class EvidenceDisplayItem(
    val text: String,        // human-readable sentence (see Evidence Formatting below)
    val turnNumber: Int?,    // used for chronological ordering
    val askId: String?       // non-null = tapping navigates to that turn in the history tab
)

data class CellDetailUiState(
    val playerName: String,
    val cardDisplayName: String,
    val inferredDisplay: CellDisplay,  // the auto icon shown in the dialog corner
    val currentDisplay: CellDisplay,   // effective display (may be manually overridden)
    val isManualOverride: Boolean,
    val probability: Float?,           // null = probabilistic scoring not enabled
    val evidenceItems: List<EvidenceDisplayItem>
)

data class GameBoardUiState(
    val gameId: String,
    val gameName: String,
    val players: List<PlayerSummary>,      // ordered by seat_order
    val cardRows: List<CardRowGroup>,
    val hasManualOverrides: Boolean,       // true = show Revert All action
    val cellDetail: CellDetailUiState?,    // non-null = detail dialog is open
    val isPrivacyScreenEnabled: Boolean = false,
    val isLoading: Boolean = false
)

data class PlayerSummary(
    val id: String,
    val name: String,
    val isLocalPlayer: Boolean
)

data class CardRowGroup(
    val type: CardType,                  // SUSPECT | WEAPON | ROOM
    val label: String,                   // "SUSPECTS", "WEAPONS", "ROOMS"
    val rows: List<CardRow>
)

data class CardRow(
    val cardId: String,
    val displayName: String,                // custom name if a game_card_override exists
    val solutionStatus: CardSolutionStatus, // drives the indicator icon on the card name cell
    val cells: List<CellState>              // one per player, in seat_order
)
```

---

## ViewModel

```kotlin
class GameBoardViewModel(
    private val gameId: String,
    private val gameRepository: GameRepository,
    private val inferenceEngine: InferenceEngine
) : ViewModel() {

    val uiState: StateFlow<GameBoardUiState>

    // Called after every new guess is logged; reruns the deduction fixpoint loop
    fun refreshInference() { ... }

    // Opens the cell detail dialog; loads evidence from Room and formats display strings
    fun onCellTap(playerId: String, cardId: String) { ... }

    fun onDismissCellDetail() { ... }

    // Persists override to cell_icon_overrides table; null = clear override (revert this cell)
    fun onOverrideCellIcon(playerId: String, cardId: String, display: CellDisplay?) { ... }

    // Deletes all cell_icon_overrides rows for this game
    fun onRevertAllOverrides() { ... }

    // Toggles the privacy screen overlay on/off
    fun onTogglePrivacyScreen() { ... }
}
```

`uiState` is derived by:
1. Loading players, cards (with overrides), and all asks from Room.
2. Running `InferenceEngine.buildMatrix()` to produce the candidate matrix and constraint list.
3. Mapping each `(player, card)` pair to a `CellDisplay` using the icon-mapping table above.
4. Merging any `cell_icon_overrides` rows from Room into `CellState.manualDisplay`.
5. Grouping cards into `CardRowGroup` by type.

Use `Flow` from the Room DAOs so the board reacts automatically when new asks are inserted.

---

## Evidence Formatting

When a cell is tapped, `onCellTap` loads the relevant `Evidence` entries and associated `asks`/`guesses` rows from Room, then formats each into a human-readable `EvidenceDisplayItem.text`:

| Evidence type | Formatted sentence |
|---|---|
| `INITIAL` | "You have this card in your starting hand." |
| `SHOWN_TO_YOU` | "[Player] showed you this card on turn [N]." |
| `SHOWN_TO_OTHER` | "[Player] showed [Other] one of: [Card1], [Card2] or [Card3] (turn [N])." |
| `CANNOT_SHOW` | "[Player] could not show [Card1], [Card2] or [Card3] (turn [N])." |
| `DEDUCTION` | "Deduced: only remaining candidate after all others were excluded." |

If `probability` is non-null: append `"Estimated likelihood: [N]%."` as a final item with no `askId`.

Each item with a non-null `askId` shows a **"View turn →"** link that will navigate to the history tab (not yet implemented — link is visible but disabled until that feature ships).

---

## Compose Components

### `GameBoardScreen`

Root composable. Hosts a `Scaffold` with a `TopAppBar`, a `FloatingActionButton` ("+ Log Turn"), and (when `uiState.cellDetail != null`) a `CellDetailDialog`.

The `TopAppBar` contains:
- **Privacy toggle button** — an eye / eye-off icon (`Icons.Default.Visibility` / `Icons.Default.VisibilityOff`) that calls `onTogglePrivacyScreen()`. Always visible so the user can reach it quickly without navigating.
- **Overflow menu (`⋮`)** — includes "Revert all to auto" when `uiState.hasManualOverrides == true`.

When `isPrivacyScreenEnabled = true`, `PrivacyScreenOverlay` is rendered on top of the entire `Scaffold` content, hiding the grid.

### `GameGrid`

The synchronized two-axis scrollable grid. Implementation approach:

```kotlin
@Composable
fun GameGrid(
    players: List<PlayerSummary>,
    cardRows: List<CardRowGroup>
) {
    val horizontalScroll = rememberScrollState()

    Column {
        // Pinned player header — scrolls horizontally with the body
        PlayerHeaderRow(players, horizontalScroll)

        // Scrollable card rows with pinned left label column
        LazyColumn {
            cardRows.forEach { group ->
                stickyHeader { CardGroupHeader(group.label) }
                items(group.rows) { row ->
                    CardGridRow(row, horizontalScroll)
                }
            }
        }
    }
}
```

Both `PlayerHeaderRow` and each `CardGridRow` share the same `horizontalScroll` `ScrollState` so horizontal scrolling stays synchronized.

### `PlayerHeaderRow`

A `Row` (inside a `horizontalScroll` modifier) with one `PlayerHeaderCell` per player. The local player's column is visually highlighted (e.g., bold name or tinted background).

### `CardNameCell`

Displays the card's display name with a leading solution indicator icon:

```kotlin
@Composable
fun CardNameCell(displayName: String, solutionStatus: CardSolutionStatus) {
    Row(verticalAlignment = Alignment.CenterVertically) {
        when (solutionStatus) {
            CardSolutionStatus.ACCOUNTED_FOR      -> Icon(Icons.Default.Check, tint = Color.Gray, contentDescription = "Accounted for")
            CardSolutionStatus.CONFIRMED_SOLUTION -> Icon(Icons.Default.Star,  tint = Color(0xFFFFD700), contentDescription = "In case file")
            CardSolutionStatus.UNKNOWN            -> Spacer(Modifier.size(24.dp)) // preserve alignment
        }
        Spacer(Modifier.width(4.dp))
        Text(displayName)
    }
}
```

The `ACCOUNTED_FOR` icon is intentionally muted — it means the card is ruled out as a solution. The `CONFIRMED_SOLUTION` star is prominent because it is a key deduction.

### `CardGroupHeader`

A full-width sticky header row showing the card type label ("SUSPECTS", "WEAPONS", "ROOMS") with a distinct background so it floats above the grid while scrolling vertically.

### `CardGridRow`

```kotlin
Row {
    CardNameCell(row.displayName, row.solutionStatus) // pinned, not in horizontal scroll
    Row(Modifier.horizontalScroll(scroll)) {
        row.cells.forEach { cell ->
            StatusIconCell(cell.display)
        }
    }
}
```

### `StatusIconCell`

Tappable cell; calls `onCellTap` when pressed. Shows the `effectiveDisplay` icon. When `isManualOverride` is true, renders a small **pencil badge** in the bottom-right corner to signal the icon was manually set.

```kotlin
@Composable
fun StatusIconCell(
    cell: CellState,
    onTap: () -> Unit
) {
    Box(
        Modifier.size(48.dp).clickable(onClick = onTap),
        contentAlignment = Alignment.Center
    ) {
        when (cell.effectiveDisplay) {
            CellDisplay.HAS_CARD  -> Icon(Icons.Default.Check, tint = Color.Green, ...)
            CellDisplay.NOT_CARD  -> Icon(Icons.Default.Close, tint = Color.Red, ...)
            CellDisplay.POSSIBLE  -> Text("%", style = MaterialTheme.typography.bodyMedium)
            CellDisplay.BLANK     -> {}
        }
        if (cell.isManualOverride) {
            Icon(
                Icons.Default.Edit,
                modifier = Modifier.size(12.dp).align(Alignment.BottomEnd),
                contentDescription = "Manually set"
            )
        }
    }
}
```

---

### `CellDetailDialog`

An `AlertDialog` (or `Dialog` with custom layout) shown when a cell is tapped.

**Layout:**
```
┌──────────────────────────────────────────────┐
│  Alice — Mustard                   [ Auto ✓ ]│  ← title + inferred icon badge
├──────────────────────────────────────────────┤
│  Estimated likelihood: 72%                   │
│                                              │
│  Evidence:                                   │
│  • Alice showed you this card on turn 4.     │
│    [View turn →]                             │
│  • Alice could not show Knife, Scarlett or   │
│    Study (turn 2).  [View turn →]            │
├──────────────────────────────────────────────┤
│  Set icon:  [ ]  [✓]  [✗]  [%]              │
│             ↑ blank is the clear/auto option  │
├──────────────────────────────────────────────┤
│                             [Done]           │
└──────────────────────────────────────────────┘
```

- **Title row** — player name + card name. Small badge in the top-right corner shows the inferred (auto) icon labeled "Auto"; this is always the inference engine's value regardless of any manual override.
- **Probability** — shown only when `probability != null`; format as `"Estimated likelihood: [N]%."`
- **Evidence list** — each `EvidenceDisplayItem` is a bullet. Items with a non-null `askId` have a **"View turn →"** tappable link; the link is disabled (grayed) until the history tab is implemented.
- **Set icon row** — four toggle buttons (blank / ✓ / ✗ / %). The currently active choice is highlighted. Selecting the blank button clears the manual override and reverts this cell to auto. Selecting any other button persists it via `onOverrideCellIcon`.
- **Done** — dismisses the dialog.

```kotlin
@Composable
fun CellDetailDialog(
    detail: CellDetailUiState,
    onOverride: (CellDisplay?) -> Unit,  // null = clear override
    onViewTurn: (askId: String) -> Unit, // navigate to history tab
    onDismiss: () -> Unit
) { ... }
```

---

## Privacy Screen

When enabled, the entire game board content is replaced by a blank overlay so other players cannot see the grid, cell details, or any inferred information. Tapping anywhere on the overlay (or pressing the toggle button again) restores the previous screen.

### `PrivacyScreenOverlay`

```kotlin
@Composable
fun PrivacyScreenOverlay(onReveal: () -> Unit) {
    Box(
        Modifier
            .fillMaxSize()
            .background(MaterialTheme.colorScheme.background)
            .clickable(onClick = onReveal),
        contentAlignment = Alignment.Center
    ) {
        Column(horizontalAlignment = Alignment.CenterHorizontally) {
            Icon(Icons.Default.VisibilityOff, contentDescription = null, modifier = Modifier.size(48.dp))
            Spacer(Modifier.height(12.dp))
            Text("Tap anywhere to reveal", style = MaterialTheme.typography.bodyLarge)
        }
    }
}
```

- The overlay covers the full screen including the `TopAppBar` so the game name and player count are also hidden.
- The `TopAppBar` privacy toggle button remains visible above the overlay so the user does not have to tap a specific spot.
- The eye-off icon in the `TopAppBar` is the only element rendered outside the overlay.
- `isPrivacyScreenEnabled` lives in `GameBoardViewModel` only — it is intentionally **not** persisted to Room or `SharedPreferences`. Closing and reopening the app always starts with the screen visible.

### Screenshot Prevention (Optional)

Set `WindowManager.LayoutParams.FLAG_SECURE` on the Activity window while the privacy screen is active to prevent screenshots and screen recordings from capturing the overlay or any content beneath it:

```kotlin
// In the Activity or via a side-effect in the composable
LaunchedEffect(isPrivacyScreenEnabled) {
    if (isPrivacyScreenEnabled) {
        window.addFlags(WindowManager.LayoutParams.FLAG_SECURE)
    } else {
        window.clearFlags(WindowManager.LayoutParams.FLAG_SECURE)
    }
}
```

This is opt-in — enable it if screenshot prevention is a priority.

- Each `StatusIconCell` must have a `contentDescription` (e.g., "Alice — Kitchen: has card", "Bob — Knife: no information") so screen readers can navigate the grid.
- Player column header cells should be marked with `Role.ColumnHeader`.
- Card name cells should be marked with `Role.RowHeader`.

---

## Navigation

```
GameBoardScreen --[+ Log Guess FAB]---------> LogGuessSheet (bottom sheet, same screen)
GameBoardScreen --[cell tap]----------------> CellDetailDialog (dialog, same screen)
GameBoardScreen --[View turn → link]--------> HistoryTab (not yet implemented)
GameBoardScreen --[back / menu]-------------> GameListScreen
```

The Log Guess sheet is a `ModalBottomSheet` overlaid on `GameBoardScreen`, not a separate destination, so the grid remains visible beneath it. The cell detail dialog is similarly in-screen.

---

## Edit History

- 2026-09-29 8:40 PM — Initial draft created.
- 2026-09-29 8:41 PM — Added `CardSolutionStatus` indicator on card name cells (accounted-for vs unknown vs confirmed solution).
- 2026-09-30 11:30 AM — Added cell tap → `CellDetailDialog` with evidence list, probability, history links, and manual icon override; added `cell_icon_overrides` persistence and Revert All action.
- 2026-09-30 1:35 PM — Added privacy screen overlay with eye-toggle button, `PrivacyScreenOverlay` composable, and optional `FLAG_SECURE` screenshot prevention.
