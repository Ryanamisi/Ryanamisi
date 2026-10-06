# AI Assistance and Prompting Plan
Student: Ryan Amisi
Course: CS 457
Sprint: 1
Date: October 6, 2026

## 1. AI Use During Sprint 1

I used ChatGPT/Codex to help draft the Tic-Tac-Toe protocol blueprint
and server state machine from the assignment instructions and my
selected game.

The assistance included message schemas, TCP framing rules, move
validation, disconnect handling, and a Mermaid state diagram.

This sprint contains design documents. Application code has not
been generated or tested as part of this work.

## 2. Recorded Design Prompt

One prompt I provided in the design conversation was:

> I chose tic tac toe for sprint 0

I also supplied screenshots of the assignment requirements and my
existing Statement of Work. The assistant used that context to draft
protocol_blueprint.md and fsm_specification.md.

The following prompts are prepared for future implementation.
They have not yet been used to generate application code.

## 3. Implementation Constraint Prompt

Use this as the initial instruction for a future coding session.
Provide the complete contents of both design documents with it.

```text
You are assisting with a Python 3 console Tic-Tac-Toe application.

Treat the supplied protocol_blueprint.md and fsm_specification.md
as the implementation contract. Read both before generating code.

Follow their exact message names, field names, field types,
message directions, validation rules, and state transitions.

Do not add or rename protocol fields or message types.
Do not add timestamps, heartbeats, rematches, or other features.
If a requirement is missing or contradictory, identify it before
coding rather than silently choosing different behavior.

Use TCP with compact UTF-8 JSON objects followed by one actual
newline byte, 0x0A. Do not use a length prefix or assume that one
recv() call contains one message.

Maintain a separate byte buffer for each connection. Extract all
complete newline-terminated frames and preserve incomplete bytes.
Decode UTF-8 only after extracting a complete frame.

Limit JSON frame content to 4096 bytes, excluding the newline.
Close an oversized connection and apply the documented departure
rules. Use sendall() for complete outgoing frames.

Each message must have exactly:
msg_type: string
player_id: Player_1, Player_2, or null as specified
payload: object with the exact fields for that message type

Validate both top-level fields and payload fields. Reject unknown
types, unexpected fields, missing fields, and incorrect types.
Do not accept JSON booleans as coordinate integers.

The server owns the board, player identities, turn, and outcome.
Validate player_id against the sending connection.
Process game events sequentially so only one move changes the
board at a time.

Invalid moves must not change the board or advance the turn.
Check for a winning line before checking for a full-board draw.

Handle DISCONNECT separately from TCP EOF and socket exceptions.
Stop reading when recv() returns b"". Discard incomplete frames
on EOF. Catch socket failures and complete cleanup.

Record a terminal outcome once. A later disconnect must not
replace an already recorded win or draw.

After a match, close the client connections, reset match data,
and keep the listening socket available for new connections.

Use Python's standard library and a console interface.
Before generating code, summarize how the implementation will
satisfy these constraints.
```

## 4. Parser and Serializer Prompt

Planned prompt for implementing the messaging component:

```text
Using the supplied protocol blueprint and implementation constraints,
implement only message serialization, stream framing, and schema
validation. Do not implement the game server yet.

Support exactly CONNECT, LOBBY_WAIT, GAME_START, MOVE, STATE_UPDATE,
ERROR, DISCONNECT, and GAME_OVER.

Preserve the blueprint's payload schemas and allowed values.
Validate the 3-by-3 board and its "", "X", and "O" cell values.

Show how the parser handles:
1. One frame split across multiple receives.
2. Multiple frames received together.
3. A UTF-8 character split across receives.
4. Invalid JSON and invalid field types.
5. A frame exceeding 4096 bytes.
6. EOF with an incomplete frame.

Do not claim tests passed unless they were actually executed.
```

## 5. Server State Machine Prompt

Planned prompt for implementing server behavior:

```text
Implement the server behavior using the supplied FSM specification
and protocol blueprint without changing either contract.

Include INIT, WAITING_FOR_PLAYERS, GAME_START, PLAYER_TURN,
EVALUATE_MOVE, CHECK_WIN_DRAW, GAME_OVER, and CLEANUP.

Assign Player_1 to X and Player_2 to O. X moves first.
Reject additional players with GAME_FULL.

Reject out-of-turn moves, invalid coordinates, identity mismatches,
occupied cells, and messages invalid for the current state.
Rejected moves must preserve the board and current player.

For a continuing move, switch turns and send STATE_UPDATE.
For a terminal move, send STATE_UPDATE with next_player null,
then GAME_OVER.

A departure during an active game causes a forfeit unless an
outcome was already recorded. A departure in the waiting lobby
frees the slot without awarding a win.

Make cleanup complete even if a notification cannot be sent.
Explain how each branch corresponds to the documented FSM.
```

## 6. Review Prompt

Planned prompt for reviewing generated code:

```text
Compare the supplied implementation with protocol_blueprint.md
and fsm_specification.md.

Identify specific contract violations in framing, schema validation,
identity checks, move processing, message order, disconnect handling,
or cleanup.

Check fragmentation and coalescing explicitly.
Check that invalid requests preserve the board and turn.
Check that terminal outcomes cannot be recorded twice.
Check that socket failures cannot prevent cleanup.

Separate findings from suggested changes.
Do not invent test results or silently revise the design documents.
```

## 7. Verification Plan

For later implementation, I will compare generated code against
both design documents and run checks for:

- Fragmented and combined TCP frames.
- Invalid JSON, fields, types, coordinates, and identities.
- Out-of-turn and occupied-cell moves.
- Wins, draws, orderly departures, and unexpected socket failures.
- Cleanup followed by a new game.

These checks are planned. No application tests have been run yet.

I will record the actual prompts, revisions, and observed test
results when implementation begins.

## 8. Actual Constrained Review

Date: October 6, 2026
Tool: ChatGPT/Codex

### Prompt Used

> Review my Tic-Tac-Toe protocol and FSM drafts above. Keep the existing JSON fields, newline framing, message types, and state names unchanged. Check for inconsistencies in turn validation, disconnects, and cleanup. Report issues without generating application code.

### Review Findings

The review found three rules needing clarification:

1. Missing or extra fields produce INVALID_MESSAGE. Present
   coordinates with incorrect types or values produce
   INVALID_COORDINATES.
2. ERROR messages sent before player assignment use player_id null,
   including GAME_FULL.
3. The match becomes active when the second valid CONNECT is
   accepted. A departure after that causes a forfeit. A departure
   before that only frees a waiting slot.

The review also found agreement between the drafts on preserving
the turn after invalid moves, keeping recorded terminal outcomes,
and completing cleanup even when notifications fail.

These findings identify clarifications to apply to the design
documents. No application code was generated or tests executed
during this review.
