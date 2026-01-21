# Chess Platform - Initial PostgreSQL Schema & Seed

This database container uses a simple `startup.sh` that starts Postgres and writes a `db_connection.txt` containing the `psql postgresql://...` connection string.

Per container rules, schema and seed are applied by executing SQL statements **one at a time** via `psql -c`.

## Connection

Use the connection string from:

- `database_postgresql/db_connection.txt`

Example:

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp
```

Or for one-statement execution:

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "SELECT 1;"
```

## What was created

### Types
- `game_status` enum: `waiting | active | finished`

### Tables
- `users`
  - `id UUID PK`
  - `username TEXT UNIQUE NOT NULL`
  - `email TEXT UNIQUE NOT NULL`
  - `password_hash TEXT NOT NULL`
  - `created_at TIMESTAMPTZ NOT NULL DEFAULT now()`

- `sessions` (tokens)
  - `id UUID PK`
  - `user_id UUID FK -> users(id) ON DELETE CASCADE`
  - `token TEXT UNIQUE NOT NULL`
  - `expires_at TIMESTAMPTZ NOT NULL`
  - `created_at TIMESTAMPTZ NOT NULL DEFAULT now()`

- `games`
  - `id UUID PK`
  - `white_user_id UUID FK -> users(id) ON DELETE RESTRICT`
  - `black_user_id UUID FK -> users(id) ON DELETE RESTRICT`
  - `status game_status NOT NULL DEFAULT 'waiting'`
  - `moves JSONB NOT NULL DEFAULT []` (array of SAN strings)
  - `current_fen TEXT NOT NULL`
  - `winner_user_id UUID NULL FK -> users(id) ON DELETE SET NULL`
  - `created_at`, `updated_at` (TIMESTAMPTZ)
  - constraint: `white_user_id <> black_user_id`

- `game_moves`
  - `id UUID PK`
  - `game_id UUID FK -> games(id) ON DELETE CASCADE`
  - `move_number INT NOT NULL ( > 0 )`
  - `san TEXT NOT NULL`
  - `from_sq TEXT NULL`, `to_sq TEXT NULL`
  - `fen_after TEXT NOT NULL`
  - `created_at TIMESTAMPTZ NOT NULL DEFAULT now()`
  - unique: `(game_id, move_number)`

- `matchmaking_queue`
  - `id UUID PK`
  - `user_id UUID FK -> users(id) ON DELETE CASCADE`
  - `rating_snapshot INT NOT NULL DEFAULT 1200`
  - `queued_at TIMESTAMPTZ NOT NULL DEFAULT now()`
  - unique: `(user_id)` (one queue entry per user)

- `chat_messages`
  - `id UUID PK`
  - `game_id UUID NULL FK -> games(id) ON DELETE CASCADE`
  - `sender_user_id UUID FK -> users(id) ON DELETE CASCADE`
  - `message_text TEXT NOT NULL (<= 2000 chars)`
  - `created_at TIMESTAMPTZ NOT NULL DEFAULT now()`

- `ratings` (simple “current rating” per user; optional glicko fields available)
  - `id UUID PK`
  - `user_id UUID FK -> users(id) ON DELETE CASCADE`
  - `rating INT NOT NULL DEFAULT 1200 (>= 0)`
  - `deviation DOUBLE PRECISION NULL`
  - `volatility DOUBLE PRECISION NULL`
  - `updated_at TIMESTAMPTZ NOT NULL DEFAULT now()`
  - unique: `(user_id)`

### Leaderboard
- `leaderboard` VIEW (computed):
  - `user_id, username, rating, updated_at`
  - ordered by `rating DESC, updated_at DESC`

### Indexes
- `sessions(user_id)`, `sessions(expires_at)`
- `games(status, created_at DESC)`, `games(white_user_id)`, `games(black_user_id)`
- `game_moves(game_id)`
- `chat_messages(game_id, created_at)`, `chat_messages(sender_user_id, created_at)`
- `matchmaking_queue(queued_at)`
- `ratings(rating DESC)`

## Re-applying schema (idempotent)

Run each statement **one at a time**:

> Note: `pgcrypto` is used to provide `gen_random_uuid()`.

1) Extension:

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "CREATE EXTENSION IF NOT EXISTS pgcrypto;"
```

2) Enum:

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "DO \$\$ BEGIN IF NOT EXISTS (SELECT 1 FROM pg_type WHERE typname = 'game_status') THEN CREATE TYPE game_status AS ENUM ('waiting','active','finished'); END IF; END \$\$;"
```

3) Tables (run individually):

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "CREATE TABLE IF NOT EXISTS users (id UUID PRIMARY KEY DEFAULT gen_random_uuid(), username TEXT NOT NULL UNIQUE, email TEXT NOT NULL UNIQUE, password_hash TEXT NOT NULL, created_at TIMESTAMPTZ NOT NULL DEFAULT now());"
```

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "CREATE TABLE IF NOT EXISTS sessions (id UUID PRIMARY KEY DEFAULT gen_random_uuid(), user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE, token TEXT NOT NULL UNIQUE, expires_at TIMESTAMPTZ NOT NULL, created_at TIMESTAMPTZ NOT NULL DEFAULT now());"
```

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "CREATE TABLE IF NOT EXISTS games (id UUID PRIMARY KEY DEFAULT gen_random_uuid(), white_user_id UUID NOT NULL REFERENCES users(id) ON DELETE RESTRICT, black_user_id UUID NOT NULL REFERENCES users(id) ON DELETE RESTRICT, status game_status NOT NULL DEFAULT 'waiting', moves JSONB NOT NULL DEFAULT '[]'::jsonb, current_fen TEXT NOT NULL, winner_user_id UUID NULL REFERENCES users(id) ON DELETE SET NULL, created_at TIMESTAMPTZ NOT NULL DEFAULT now(), updated_at TIMESTAMPTZ NOT NULL DEFAULT now(), CONSTRAINT games_players_distinct CHECK (white_user_id <> black_user_id));"
```

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "CREATE TABLE IF NOT EXISTS game_moves (id UUID PRIMARY KEY DEFAULT gen_random_uuid(), game_id UUID NOT NULL REFERENCES games(id) ON DELETE CASCADE, move_number INTEGER NOT NULL CHECK (move_number > 0), san TEXT NOT NULL, from_sq TEXT NULL, to_sq TEXT NULL, fen_after TEXT NOT NULL, created_at TIMESTAMPTZ NOT NULL DEFAULT now(), CONSTRAINT game_moves_unique_move UNIQUE (game_id, move_number));"
```

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "CREATE TABLE IF NOT EXISTS matchmaking_queue (id UUID PRIMARY KEY DEFAULT gen_random_uuid(), user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE, rating_snapshot INTEGER NOT NULL DEFAULT 1200, queued_at TIMESTAMPTZ NOT NULL DEFAULT now(), CONSTRAINT matchmaking_queue_user_unique UNIQUE (user_id));"
```

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "CREATE TABLE IF NOT EXISTS chat_messages (id UUID PRIMARY KEY DEFAULT gen_random_uuid(), game_id UUID NULL REFERENCES games(id) ON DELETE CASCADE, sender_user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE, message_text TEXT NOT NULL CHECK (char_length(message_text) <= 2000), created_at TIMESTAMPTZ NOT NULL DEFAULT now());"
```

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "CREATE TABLE IF NOT EXISTS ratings (id UUID PRIMARY KEY DEFAULT gen_random_uuid(), user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE, rating INTEGER NOT NULL DEFAULT 1200 CHECK (rating >= 0), deviation DOUBLE PRECISION NULL, volatility DOUBLE PRECISION NULL, updated_at TIMESTAMPTZ NOT NULL DEFAULT now(), CONSTRAINT ratings_user_unique UNIQUE (user_id));"
```

4) Indexes (run individually):

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "CREATE INDEX IF NOT EXISTS idx_sessions_user_id ON sessions(user_id);"
```

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "CREATE INDEX IF NOT EXISTS idx_sessions_expires_at ON sessions(expires_at);"
```

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "CREATE INDEX IF NOT EXISTS idx_games_status_created_at ON games(status, created_at DESC);"
```

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "CREATE INDEX IF NOT EXISTS idx_games_white_user_id ON games(white_user_id);"
```

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "CREATE INDEX IF NOT EXISTS idx_games_black_user_id ON games(black_user_id);"
```

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "CREATE INDEX IF NOT EXISTS idx_game_moves_game_id ON game_moves(game_id);"
```

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "CREATE INDEX IF NOT EXISTS idx_chat_messages_game_id_created_at ON chat_messages(game_id, created_at);"
```

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "CREATE INDEX IF NOT EXISTS idx_chat_messages_sender_user_id_created_at ON chat_messages(sender_user_id, created_at);"
```

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "CREATE INDEX IF NOT EXISTS idx_matchmaking_queue_queued_at ON matchmaking_queue(queued_at);"
```

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "CREATE INDEX IF NOT EXISTS idx_ratings_rating_desc ON ratings(rating DESC);"
```

5) Leaderboard view:

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "CREATE OR REPLACE VIEW leaderboard AS SELECT u.id AS user_id, u.username, r.rating, r.updated_at FROM users u JOIN ratings r ON r.user_id = u.id ORDER BY r.rating DESC, r.updated_at DESC;"
```

## Minimal seed data (idempotent)

Two users + current ratings + one sample finished game + a couple moves + one chat message.

Run each statement one at a time:

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "INSERT INTO users (username,email,password_hash) VALUES ('alice','alice@example.com','seed_hash_alice') ON CONFLICT (username) DO NOTHING;"
```

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "INSERT INTO users (username,email,password_hash) VALUES ('bob','bob@example.com','seed_hash_bob') ON CONFLICT (username) DO NOTHING;"
```

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "INSERT INTO ratings (user_id,rating,updated_at) SELECT id, 1500, now() FROM users WHERE username='alice' ON CONFLICT (user_id) DO UPDATE SET rating=EXCLUDED.rating, updated_at=EXCLUDED.updated_at;"
```

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "INSERT INTO ratings (user_id,rating,updated_at) SELECT id, 1400, now() FROM users WHERE username='bob' ON CONFLICT (user_id) DO UPDATE SET rating=EXCLUDED.rating, updated_at=EXCLUDED.updated_at;"
```

Sample finished game (Scholar’s mate), seeded uniquely by the final `current_fen`:

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "INSERT INTO games (white_user_id, black_user_id, status, moves, current_fen, winner_user_id, created_at, updated_at) SELECT w.id, b.id, 'finished', '[\"e4\",\"e5\",\"Qh5\",\"Nc6\",\"Bc4\",\"Nf6\",\"Qxf7#\"]'::jsonb, 'r1bqkb1r/pppp1Qpp/2n2n2/4p3/2B1P3/8/PPPP1PPP/RNB1K1NR b KQkq - 0 4', w.id, now() - interval '1 day', now() - interval '1 day' FROM users w, users b WHERE w.username='alice' AND b.username='bob' AND NOT EXISTS (SELECT 1 FROM games g WHERE g.current_fen = 'r1bqkb1r/pppp1Qpp/2n2n2/4p3/2B1P3/8/PPPP1PPP/RNB1K1NR b KQkq - 0 4');"
```

A couple moves (seeded using the game’s `current_fen` and `ON CONFLICT DO NOTHING`):

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "INSERT INTO game_moves (game_id, move_number, san, from_sq, to_sq, fen_after, created_at) SELECT g.id, 1, 'e4', 'e2', 'e4', 'rnbqkbnr/pppppppp/8/8/4P3/8/PPPP1PPP/RNBQKBNR b KQkq - 0 1', g.created_at FROM games g WHERE g.current_fen = 'r1bqkb1r/pppp1Qpp/2n2n2/4p3/2B1P3/8/PPPP1PPP/RNB1K1NR b KQkq - 0 4' ON CONFLICT (game_id, move_number) DO NOTHING;"
```

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "INSERT INTO game_moves (game_id, move_number, san, from_sq, to_sq, fen_after, created_at) SELECT g.id, 2, 'e5', 'e7', 'e5', 'rnbqkbnr/pppp1ppp/8/4p3/4P3/8/PPPP1PPP/RNBQKBNR w KQkq - 0 2', g.created_at FROM games g WHERE g.current_fen = 'r1bqkb1r/pppp1Qpp/2n2n2/4p3/2B1P3/8/PPPP1PPP/RNB1K1NR b KQkq - 0 4' ON CONFLICT (game_id, move_number) DO NOTHING;"
```

One chat message:

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "INSERT INTO chat_messages (game_id, sender_user_id, message_text, created_at) SELECT g.id, u.id, 'gg! nice game', g.created_at + interval '10 minutes' FROM games g JOIN users u ON u.username='bob' WHERE g.current_fen = 'r1bqkb1r/pppp1Qpp/2n2n2/4p3/2B1P3/8/PPPP1PPP/RNB1K1NR b KQkq - 0 4' AND NOT EXISTS (SELECT 1 FROM chat_messages cm WHERE cm.game_id = g.id AND cm.message_text = 'gg! nice game');"
```

## Notes for the backend
- IDs are UUIDs; use `uuid` strings in API responses/requests.
- `games.moves` is a JSONB array of SAN strings; detailed move history is in `game_moves`.
- `leaderboard` can be queried directly as a view (e.g., `SELECT * FROM leaderboard LIMIT 100;`).

## Quick sanity query

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "SELECT 'users' AS table, count(*) FROM users UNION ALL SELECT 'games', count(*) FROM games UNION ALL SELECT 'game_moves', count(*) FROM game_moves UNION ALL SELECT 'chat_messages', count(*) FROM chat_messages UNION ALL SELECT 'ratings', count(*) FROM ratings;"
```
