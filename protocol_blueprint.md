# Tic-Tac-Toe Protocol Blueprint
Student: Ryan Amisi
Course: CS 457
Sprint: 1

## 1. Game and Transport

The game supports two console clients. The server assigns Player_1
the X symbol and Player_2 the O symbol. X moves first.

The server owns the board, validates moves, controls turns, and
determines wins, draws, and forfeits. Clients submit requests and
display server responses.

Communication uses TCP. Messages are UTF-8 JSON objects terminated
by one newline byte (0x0A).

## 2. Message Framing

Each message is serialized as compact JSON followed by an actual
newline. Pretty-printed JSON with embedded line breaks is not used.

TCP is a byte stream. One recv() call may contain part of a message,
one complete message, or several messages.

The receiver must:
1. Append received bytes to a connection-specific buffer.
2. Extract every complete frame ending with 0x0A.
3. Keep incomplete bytes for the next recv() call.
4. Decode each complete frame as UTF-8.
5. Parse and validate the JSON object.

Decoding happens after a complete frame is extracted, because a
UTF-8 character may be split across recv() calls.

Senders use sendall() to transmit the complete encoded frame.

A frame may contain at most 4096 bytes of JSON, excluding its newline.
An oversized frame causes the connection to close. Invalid UTF-8,
invalid JSON, or an invalid schema produces ERROR when possible.

### Wire Example

In the notation below, <LF> represents one actual newline byte.
It is not literal text sent over the network.

{"msg_type":"MOVE","player_id":"Player_1","payload":{"row":0,"col":2}}<LF>

Two messages can arrive together:

{"msg_type":"CONNECT","player_id":null,"payload":{"alias":"Ryan"}}<LF>{"msg_type":"DISCONNECT","player_id":"Player_1","payload":{"reason":"quit"}}<LF>

If a message arrives in several chunks, the receiver waits for its
newline before decoding and processing it.

## 3. Common Message Schema

Every message contains exactly these three fields:

| Field | Type | Meaning |
|---|---|---|
| msg_type | string | One of the eight message types below |
| player_id | string or null | Player_1, Player_2, or null |
| payload | object | Fields required by the message type |

CONNECT uses a null player_id because the server has not assigned
an identity yet. Other client messages use the identity assigned
to that connection. Server messages directed to one player use that
player's identity; messages sent to both players use null.

The server checks the connection's assigned identity rather than
trusting a client-provided player_id.

Unknown message types, extra fields, missing fields, and wrong field
types are rejected. JSON booleans are not accepted as integers.
This version does not include a timestamp field.

## 4. Message Types and Payloads

| Message | Direction | Required payload fields | Purpose |
|---|---|---|---|
| CONNECT | Client → Server | alias: string, 1–32 characters | Request a player slot |
| LOBBY_WAIT | Server → Client | message: string | Tell the first player to wait |
| GAME_START | Server → Both clients | players: object; board: array; next_player: string | Assign symbols and start the game |
| MOVE | Client → Server | row: integer 0–2; col: integer 0–2 | Request a move |
| STATE_UPDATE | Server → Both clients | board: array; next_player: string or null | Publish the authoritative board |
| ERROR | Server → Client | code: string; message: string | Explain a rejected request |
| DISCONNECT | Client → Server | reason: string, 1–128 characters | Request an orderly departure |
| GAME_OVER | Server → Both connected clients | result: string; winner: string or null; reason: string; board: array; scores: object | Report the final outcome |

The board is a 3×3 array. Each cell is "", "X", or "O".
Coordinates are zero-based: row 0 is the top and column 0 is the left.

The players object contains Player_1 and Player_2. Each entry has
alias (string) and symbol ("X" or "O").

GAME_OVER result is "win", "draw", or "forfeit".
Its winner is Player_1, Player_2, or null for a draw.
Its reason is "three_in_a_row", "board_full", "player_quit",
or "connection_lost".

Scores describe this match only: a winner receives 1 and the other
player receives 0; a draw gives both players 0.

## 5. Concrete Message Examples

### CONNECT
{"msg_type":"CONNECT","player_id":null,"payload":{"alias":"Ryan"}}

### LOBBY_WAIT
{"msg_type":"LOBBY_WAIT","player_id":"Player_1","payload":{"message":"Waiting for Player_2"}}

### GAME_START
{"msg_type":"GAME_START","player_id":null,"payload":{"players":{"Player_1":{"alias":"Ryan","symbol":"X"},"Player_2":{"alias":"Alex","symbol":"O"}},"board":[["","",""],["","",""],["","",""]],"next_player":"Player_1"}}

### MOVE
{"msg_type":"MOVE","player_id":"Player_1","payload":{"row":0,"col":2}}

### STATE_UPDATE
{"msg_type":"STATE_UPDATE","player_id":null,"payload":{"board":[["","","X"],["","",""],["","",""]],"next_player":"Player_2"}}

### ERROR
{"msg_type":"ERROR","player_id":"Player_2","payload":{"code":"OUT_OF_TURN","message":"It is Player_1's turn"}}

### DISCONNECT
{"msg_type":"DISCONNECT","player_id":"Player_1","payload":{"reason":"quit"}}

### GAME_OVER
{"msg_type":"GAME_OVER","player_id":null,"payload":{"result":"win","winner":"Player_1","reason":"three_in_a_row","board":[["X","X","X"],["O","O",""],["","",""]],"scores":{"Player_1":1,"Player_2":0}}}

Each example above must be followed by an actual newline when sent.

## 6. Move Validation and Message Order

The server accepts CONNECT once per connection. The first accepted
player receives LOBBY_WAIT. When the second player joins, both
receive GAME_START. Additional players receive ERROR with code
GAME_FULL and their connections close.

A MOVE is accepted only during an active game, from the current
player, with valid coordinates pointing to an empty cell.

Rejected moves do not change the board or advance the turn.

ERROR codes include:
- INVALID_MESSAGE: invalid JSON, schema, or message type.
- INVALID_STATE: request is not allowed in the current state.
- IDENTITY_MISMATCH: player_id does not match the connection.
- OUT_OF_TURN: another player owns the current turn.
- INVALID_COORDINATES: row or col is not an integer from 0 to 2.
- CELL_OCCUPIED: the selected cell already contains a symbol.
- GAME_FULL: two players already occupy the game.

After an accepted move, the server checks for a win or draw.
For a continuing game, it sends STATE_UPDATE with the next player.
For a terminal move, it sends STATE_UPDATE with next_player null,
then GAME_OVER with the final board and scores.

## 7. Disconnects and Cleanup

DISCONNECT is an application message. TCP EOF and socket exceptions
are transport events, so they must also be handled independently.

During an active game, a departing player forfeits. The remaining
player receives GAME_OVER with result "forfeit". Its reason is
"player_quit" for DISCONNECT or "connection_lost" for a transport
failure.

While waiting for players, a departure frees the player slot
without awarding a win.

If recv() returns b"", the server stops reading that connection.
If EOF leaves an incomplete frame, that frame is discarded.

ConnectionResetError, BrokenPipeError, and other socket errors
trigger connection cleanup rather than crashing the server.
A failed send must not prevent cleanup of the other connection.

TCP alone does not guarantee prompt detection of an unreachable
peer. This protocol does not yet define a heartbeat or inactivity
timeout.

After GAME_OVER, the server closes both game connections, clears
the board and player records, and returns to waiting for players.
A new game requires new client connections.

## 8. Validation and Startup Clarifications

These rules clarify the earlier sections:

- Missing or extra fields produce INVALID_MESSAGE. For MOVE,
  when row and col are present, incorrect coordinate types or
  values produce INVALID_COORDINATES. Booleans are invalid
  coordinates. Rejected moves preserve the board and turn.

- Server ERROR messages use player_id null when the connection
  has no assigned player identity, including GAME_FULL.
  Otherwise, ERROR uses the connection's assigned identity.

- The match becomes active when the second valid CONNECT is
  accepted, including the GAME_START state. A departure after
  that causes a forfeit unless a terminal outcome is already
  recorded. Before that, a departure only frees a waiting slot.
