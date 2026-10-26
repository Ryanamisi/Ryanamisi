# Sprint 2 - Server Concurrency Architecture Design

## 1. Concurrency Model

The server will use a multi-threaded concurrency model. Each client that connects to the server will be handled by its own worker thread. This allows multiple clients to communicate with the server at the same time without one client blocking another.

Python's `threading` module will be used to manage the client connections. The server will support at least two simultaneous client connections.

Each worker thread will receive and process messages from its assigned client while shared game information remains stored on the server.

## 2. Thread Safety and Locking

Since multiple client threads can access the game state at the same time, the server will use a `threading.Lock` to protect shared data.

The lock will protect:

- The game board
- The current player's turn
- Player scores
- The connected client/socket list

The `threading.Lock` will be acquired before any worker thread reads or modifies shared game state that must remain consistent. The critical section includes validating the current turn, validating move legality, applying a valid move to the game board, changing the active player, and updating player scores.

After the shared state update is complete, the worker thread will release the lock. Client-specific socket communication that does not modify shared state does not require the lock.

This prevents two client threads from validating or modifying the same game state at the same time and helps prevent race conditions. For example, two clients cannot both make a valid move during the same turn because the first thread that acquires the lock will complete its validation and state update before the other thread can modify the shared state.

## 3. Client Connection Lifecycle

When a client connects to the server, the server accepts the connection and creates a new worker thread for that client. The thread is responsible for receiving and processing messages from that client.

The connection follows these steps:

1. The client connects to the server.
2. The server accepts the connection and creates a worker thread.
3. The client is added to the connected client list.
4. The client waits until another player joins if necessary.
5. Once both players are connected, the server starts the game.
6. The server receives moves from each client.
7. The server acquires the lock and verifies that it is that player's turn and that the requested move is legal.
8. If the move is valid, the server updates the game board and active turn while the lock is held.
9. The server releases the lock after the shared game state has been updated.
10. The server broadcasts the new `STATE_UPDATE` to both clients so they remain synchronized.
11. If the move is invalid or submitted out of turn, the server sends an `ERROR` response without changing the game state.
12. When a client disconnects, the server removes the client from the connected client list and closes the socket connection.

## 4. Client State and Turn Synchronization

The following sequence diagram shows how client moves are validated and how the server keeps both clients synchronized.

```mermaid
sequenceDiagram
    participant Client1
    participant Server
    participant Client2

    Client1->>Server: JOIN
    Server-->>Client1: WAITING

    Client2->>Server: JOIN
    Server-->>Client1: GAME_START
    Server-->>Client2: GAME_START

    Client1->>Server: MOVE

    alt Valid Move and Correct Turn
        Server-->>Client1: STATE_UPDATE
        Server-->>Client2: STATE_UPDATE
    else Invalid or Out-of-Turn Move
        Server-->>Client1: ERROR (Invalid Move or Not Your Turn)
    end

    Server-->>Client1: SCORE_UPDATE
    Server-->>Client2: SCORE_UPDATE

    Server-->>Client1: GAME_OVER
    Server-->>Client2: GAME_OVER
