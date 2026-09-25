# Real-Time Tic Tac Toe

Assignment 5 — Ashutosh Pawar (150096725130)

A two-player Tic Tac Toe game where both browsers stay in sync over a WebSocket
connection instead of polling the server. Express serves the static client, Socket.IO
carries the moves, and the server keeps the board in memory so it — not the browser —
decides whose turn it is and who won. The first player to join is assigned `X`, the
second `O`, and a third connection is rejected. When a game ends, the result is written
to MongoDB Atlas and the client re-fetches the history table over a normal HTTP route.

## Live demo

https://assignment-5-real-time-tic-tac-toe-game-2hp5.onrender.com

Open it in two browser tabs (or on two devices) and join under different usernames.
The same Render service hosts both the game page and the Socket.IO server. It runs on
Render's free tier, so the first visit after a period of inactivity can take up to a
minute, and a game in progress resets if the server restarts. Finished games are kept
in MongoDB.

## Tech stack

- Express 5 — static file serving and the history route
- Socket.IO 4 — real-time events between the two players
- Mongoose 9 — MongoDB Atlas schema and queries
- cors — cross-origin access for the Socket.IO handshake
- dotenv — config
- Plain HTML, CSS and JavaScript on the client (no framework, no build step)

## Project structure

```
Ashutosh_Pawar_150096725130/
├── models/
│   └── Game.js                   # finished-game schema, collection pinned to 'tictactoe'
├── public/
│   ├── index.html                # login box, 3x3 board, history table
│   ├── style.css                 # plain black-and-white styling
│   └── script.js                 # socket client, board rendering, history fetch
├── requests.http                 # sample requests
├── .env.example                  # environment template
└── server.js                     # app entry, socket handlers, game logic, port 3000
```

Game state (`board`, `players`, `currentTurn`, `totalMoves`) lives in module-level
variables in `server.js`. That is deliberate for this assignment — one server process
hosts one match at a time.

## Setup

Requires Node.js and a MongoDB connection string (Atlas free tier or a local `mongod`).

```bash
npm install
cp .env.example .env      # then fill in MONGO_URI
npm start                 # or: npm run dev
```

Open `http://localhost:3000` in **two** browser tabs — the first tab plays `X`, the
second plays `O`. A third tab receives a `login-error` and cannot join.

On Atlas, whitelist your IP under Network Access first, and percent-encode any special
characters in the password before putting it in the URI.

## Deployment

Deployed as one Render web service: root directory `Ashutosh_Pawar_150096725130`,
build `npm install`, start `npm start`, with `MONGO_URI` set to a MongoDB Atlas
connection string. Render provides `PORT` and supports WebSockets, so no extra
configuration is needed.

## Environment variables

| Variable | Required | Description |
| --- | --- | --- |
| `MONGO_URI` | yes | MongoDB connection string, database `tictactoe` |
| `PORT` | no | HTTP port, defaults to `3000` |

## Data model

One document is written per finished game — nothing is stored while a game is in
progress.

### Game (collection `tictactoe`)

| Field | Type | Notes |
| --- | --- | --- |
| `playerX` | String | required, username of the `X` player |
| `playerO` | String | required, username of the `O` player |
| `winner` | String | winning username, `null` on a draw |
| `winnerSymbol` | String | `X`, `O`, or `null` on a draw |
| `result` | String | required, one of `X_WON`, `O_WON`, `DRAW` |
| `totalMoves` | Number | required, moves played across both players |
| `playedAt` | Date | defaults to the time the game finished |

`winnerSymbol` and `result` are `enum` fields, so an invalid value is rejected by
Mongoose before it reaches the database.

## Socket events

There is no login token — a player is identified by their `socket.id`, which the server
records when they join and uses to reject moves made out of turn.

### Client to server

| Event | Payload | Description |
| --- | --- | --- |
| `user-login` | `{ username }` | Join the match, get assigned `X` or `O` |
| `make-move` | `{ index, symbol }` | Claim cell `0`–`8` |
| `reset-game` | — | Clear the board and drop both players |

### Server to client

| Event | Payload | Description |
| --- | --- | --- |
| `login-success` | `{ username, symbol }` | Sent to the joining player only |
| `login-error` | `{ message }` | Sent when the match already has two players |
| `players-update` | `[{ id, username, symbol }]` | Broadcast on every join |
| `game-start` | `{ players }` | Broadcast once the second player joins |
| `move-made` | `{ index, symbol }` | Broadcast after a move is accepted |
| `game-over` | `{ winner, winnerSymbol, result, board }` | Broadcast when a line fills or the board is full |
| `game-reset` | — | Broadcast on reset or on a player disconnecting |

The server rejects a `make-move` when it comes from a socket that is not a player, when
it is not that symbol's turn, or when the cell is already taken. The client greys these
out too, but the server check is what actually enforces the rules.

## Endpoints

| Method | Route | Description |
| --- | --- | --- |
| GET | `/` | The game client, served from `public/` |
| GET | `/api/history` | Last 10 finished games, newest first |

## Status codes

| Code | When |
| --- | --- |
| 200 | History returned successfully |
| 404 | Unknown route |
| 500 | Database error while reading history |

## Testing

`requests.http` holds the two HTTP requests. The interesting part of this assignment is
the socket traffic, which is tested by opening two browser tabs on
`http://localhost:3000`, joining under different usernames and playing a full game —
each move should appear in both tabs at once. When the game ends, the history table
picks up the new row, and the document appears in the `tictactoe` collection on Atlas.

The server logs every connect and disconnect with the socket id, which makes it easy to
see which tab is which.
