# Clue App Planning Skill

Scope: workspace

Version: 0.1

Summary
-------
This skill helps generate a complete planning and documentation bundle for a "Clue" (Cluedo) companion app. The app records per-turn suggestions (who guessed what), which player showed which card, and then uses that history to make informative inferences about which cards each player likely holds. The skill produces backend and frontend documentation, data models, API specs, QA/test plans, future roadmaps, risk/warning notes, and an edit log.

When to use
-----------
- Starting a new Clue companion app project.
- Converting ad-hoc design notes into structured specs and implementation tasks.
- Generating incremental documentation for backend, frontend, inference engine, and tests.

Primary outputs
---------------
- Architecture overview (backend, frontend, infra).
- Data model and SQL/NoSQL schema (tables, fields, indices).
- API spec (REST/OpenAPI or GraphQL endpoints with examples).
- Inference engine design and pseudocode (deterministic rules + probabilistic scoring).
- Frontend UX and component spec (screens, flows, mock data).
- Test plan (unit, integration, end-to-end) and sample test cases.
- Roadmap: future features, extensions, and project risks.

Documentation folder
--------------------
All generated documentation files (API specs, DB schemas, frontend docs, test plans, roadmaps, and exports) MUST be saved in a top-level `documentation/` folder in the project. Organize the folder with subsystem subfolders such as `backend/`, `frontend/`, `inference/`, `tests/`, and `roadmap/` to keep artifacts discoverable and version-controlled.

How to invoke (example prompts)
------------------------------
- "Plan the backend for the Clue app: OpenAPI spec, DB schema, sample queries, and migrations."
- "Draft the frontend: Android Kotlin Jetpack Compose screens and state model for adding players and logging suggestions."
- "Write the inference algorithm in pseudocode and provide a unit-testable Python implementation." 
- "Generate a QA checklist and sample E2E test that simulates a 4-player game with reveals." 

What this skill does (workflow)
------------------------------
1. Gather key requirements from the user: platform (mobile/web), multi-device sync, real-time vs manual input, privacy constraints, and Clue rule variant.
2. Produce an architecture sketch with recommended tech choices and trade-offs.
3. Generate a data model capturing Games, Players, Cards, Guesses, and Reveals.
4. Create API endpoints and example payloads for game lifecycle and logging events.
5. Design the inference engine (deterministic elimination + probabilistic scoring) and produce sample code.
6. Produce frontend screen-by-screen specs and component breakdowns.
7. Produce a QA/test plan and sample unit/integration tests.
8. Produce a short roadmap, feature ideas, and a risk/warning section.

Decision points and branching logic
----------------------------------
- Platform: single-device (manual entry) vs multi-device (sync + auth). Choose offline-first if players prefer pocketed use.
- Real-time vs turn-based logging: real-time requires websockets/Push while turn-based can be manual input.
- Data privacy: store identifying player names locally only vs persist to cloud.
- Inference strictness: deterministic-only (safe, transparent) vs probabilistic scoring (more useful but may mislead).
- Clue rule variants: In some variants, the first player to be able to show a card must do so; in others, rules vary. The skill asks this up-front and adapts the model.

Key data model (conceptual)
---------------------------
- Game: id, name, startedAt, variant, metadata
- Player: id, gameId, name (or anonymized label), seatOrder, isLocal
- Card: id, type (Suspect|Weapon|Room), name
- Guess: id, gameId, turnNumber, guessingPlayerId, suspectCardId, weaponCardId, roomCardId, timestamp
- AskSequence: guessId, askedPlayerId, askedIndex (order after guesser), responseType (None|Shown|Unable), shownCardId (nullable)
- Reveal: id, guessId, showingPlayerId, shownCardId, timestamp

Notes on modeling reveals
------------------------
- Only record the first positive reveal per guess (traditional rules). Optionally record if others also privately show cards (house rules).
- Consider an auditable 'audit trail' so users can correct mistakes (mis-clicks) with minimal data loss.

Inference engine (overview)
---------------------------
Goal: From all suggestions and reveals, produce for every (player, card) a state among: KnownInHand, KnownNotInHand, PossibilityScore (0..1), or Unknown.

Steps (deterministic elimination)
1. Initialize: for every player P and card C, set candidate(P,C)=true unless C is the case file (solution deck) known to be unavailable.
2. When a card becomes KNOWN for a player (from any source — initial hand, shown to you, or deduction), immediately set candidate(otherPlayer, card)=false for every other player. Each card exists exactly once.
3. If a player was asked and was unable to show any card in a guess, then for every card in that guess mark candidate(thatPlayer,card)=false.
4. If only one player remains as candidate for a card, mark KnownInHand — then apply step 2.

Probabilistic scoring (optional)
- Maintain prior that each non-solution card is uniformly distributed among players whose hands can contain it.
- Use recorded 'cannot show' responses as hard evidence (zero probability) and 'showed card X' as one-hot.
- Compute normalized probabilities per (player,card). Use Bayesian updates when a player shows a card from multiple suggestions.

Example pseudocode (deterministic core)
```python
def record_guess(game, guesser, suspect, weapon, room, asked_sequence):
    # asked_sequence: ordered list of (player, shown_card_or_none)
    for player, shown_card in asked_sequence:
        if shown_card:
            set_known(player, shown_card)          # triggers uniqueness propagation
            break
        else:
            # player failed to show any card from the guess
            for c in (suspect, weapon, room):
                mark_candidate(player, c, False)

def set_known(player, card):
    mark_candidate(player, card, True)
    # uniqueness: only one player can hold each card
    for other in game.players:
        if other != player:
            mark_candidate(other, card, False)

def infer_knowns(game):
    # scan cards and players to find unique candidates
    for card in game.cards:
        candidates = [p for p in game.players if candidate(p, card)]
        if len(candidates) == 1:
            set_known(candidates[0], card)
```

API design (minimal)
--------------------
- POST /games -> create game
- POST /games/{id}/players -> add player
- POST /games/{id}/start -> start game
- POST /games/{id}/guesses -> record guess + asked sequence
- GET  /games/{id}/state -> current knowns + probabilities
- POST /games/{id}/corrections -> edit/delete an event (audit trail)

Payload example: POST /games/{id}/guesses
```json
{
  "turnNumber": 12,
  "guesserId": "player-3",
  "suspectId": "scarlet",
  "weaponId": "candlestick",
  "roomId": "library",
  "askedSequence": [
    {"playerId":"player-4","shownCard":null},
    {"playerId":"player-1","shownCard":"library"}
  ]
}
```

Backend architecture recommendations
----------------------------------
- Minimal: Node.js/Express + PostgreSQL (relational model fits well), with a simple REST/OpenAPI spec.
- Alternative: Python/FastAPI + PostgreSQL for fast prototyping and autogenerated OpenAPI docs.
- For multi-device sync/real-time: add WebSockets via Socket.IO or use Firebase/Realtime DB for quick MVP.
- Store sensitive data locally or encrypted when hosted remotely; provide opt-in cloud sync.

Frontend architecture
---------------------
- Mobile-first design: Android (Kotlin + Jetpack Compose) or cross-platform React Native/Flutter.
- Components: GameList, GameDetail, PlayerEditor, SuggestionEntry modal, RevealEntry, InferenceDashboard, Export/Share.
- UX flows: quick 'Log Suggestion' flow that guides the user through selecting suggester, cards, and recording who was asked and what (if anything) was shown.

Testing and QA
--------------
- Unit tests for inference engine (deterministic and probabilistic paths).
- Integration tests that simulate multiple turns and verify knowns/probabilities.
- E2E tests that exercise frontend flows: add players -> start -> log suggestion -> verify dashboard updates.

Acceptance criteria / quality checklist
-------------------------------------
- Backend: OpenAPI spec, DB migrations, unit tests >80% coverage for inference code.
- Frontend: accessible UI for logging suggestions and viewing inferences; input validation for turn order.
- Reliability: changes are auditable; corrections are supported.

Future roadmap & ideas
----------------------
- Real-time multiplayer sync with a host and guests.
- Machine-learning recommender that suggests the next most-informative suggestion to narrow possibilities.
- Export/import game state and timeline for replay or arbitration.
- Privacy: local-only mode and optional encrypted cloud backups.
- Integrate with physical game timers and turn reminders.

Warnings & risks
----------------
- Cheating: this app can be used to gain an advantage; include clear warnings and consider UX options that discourage misuse (e.g., private-only mode).
- Data correctness: manual inputs are error-prone — provide edits and audit trails.
- Rule variations: house rules vary; make rule variant configurable.

Ambiguities & questions to clarify
---------------------------------
1. Which platform(s) do you want to target first? (Android native, iOS, web)
2. Do you want multi-device real-time synchronization or single-device/manual logging?
3. Which Clue rule variant are we modeling (first-positive-show stops further reveals)?
4. Do you want probabilistic scoring, or only deterministic eliminations?

Iteration guidance (how to use this skill)
----------------------------------------
1. Run: "Generate architecture overview" → review and pick stack.
2. Run: "Generate backend spec" → accept DB schema; ask for migrations.
3. Run: "Generate inference engine" → review pseudocode, ask for implementation language (Python/Kotlin/TypeScript).
4. Run: "Generate frontend screens" → get component list and sample UI markup.
5. Iterate: ask targeted follow-ups using the example prompts above.

Prompts this skill will emit for subsystems
-----------------------------------------
- Backend spec: "Create an OpenAPI v3 spec for the Clue app with endpoints for games, players, guesses, corrections, and a /state endpoint returning inferred probabilities. Include example request/response."
- DB schema: "Produce SQL CREATE TABLE statements for games, players, cards, guesses, ask_sequences, and reveals with indices and example queries."
- Inference: "Write a deterministic inference engine and an optional Bayesian scoring extension in [language]. Include unit tests."
- Frontend: "Produce Kotlin Jetpack Compose screen components and sample state holder classes to implement the 'Log Suggestion' flow."

Editable checklist for each doc produced
-------------------------------------
- Title and summary present
- Stakeholders and scope documented
- Endpoints and payloads complete
- DB schema with indices and migrations provided
- Inference algorithm test cases included
- E2E test scenario described

Next steps for me (suggested prompts)
-----------------------------------
- "Generate the OpenAPI spec for the backend (FastAPI) with example responses."
- "Write the Kotlin Jetpack Compose screens for logging a suggestion and recording reveals."
- "Implement the deterministic inference engine in Python and include unit tests."

Edit history
------------
Every entry MUST include the date AND time in the format `YYYY-MM-DD H:MM AM/PM — description`.
Use the local time from the `current date` context provided at the time of the edit.

- 2026-09-29 7:17 PM — Initial draft created.
