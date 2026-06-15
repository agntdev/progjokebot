## Summary
A minimal Telegram bot that replies to /joke with a random programming joke and tracks how many jokes each user has seen. Tiny, single-file-friendly implementation using SQLite and a small external Joke API with a local fallback.

## Audience
Telegram users who want a quick programming joke and want to see how many jokes they've received from the bot.

## Core entities
- User
  - user_id (Telegram numeric id, primary key)
  - username (nullable)
  - first_name (nullable)
  - joke_count (integer)
  - last_seen (timestamp)
- Joke source (external API or local fallback list)

## Integrations & notification targets
- Telegram Bot API (required) — bot receives commands and sends messages.
- External Joke API (default: JokeAPI at https://v2.jokeapi.dev/joke/Programming) — fetch random programming joke.
- No external notification targets (no email, Slack, etc.).

## Interaction flows
- /start (optional): Bot responds with a short greeting and brief instructions: "Use /joke for a programming joke, /stats to see your count." No DB write required beyond creating row on first /joke or /stats as below.

- /joke:
  1. Bot receives /joke (message contains user.id and optional username/first_name).
  2. Ensure the user exists in the users table; if not, insert a new row with joke_count=0 and captured metadata.
  3. Fetch a random programming joke from JokeAPI (request type=single,two-part). If the external API fails or returns an error, pick a joke from a built-in small list.
  4. Send the joke to the user: if single-part, send the text; if two-part, send setup then punchline as a single message separated by a blank line (or two messages if you prefer).
  5. Increment joke_count for that user atomically and update last_seen.
  6. (Optional) append a small footer like "You've seen X jokes." (default: include count on same reply).

- /stats:
  1. Bot receives /stats.
  2. Ensure the user exists in DB; if not, create with joke_count=0.
  3. Read joke_count and last_seen and reply with "You've seen N jokes. Last seen: <timestamp>" (timestamp is optional human-friendly).

- Error handling:
  - If joke fetch fails, fallback to built-in jokes with a brief note "(local joke fallback)".
  - Handle and log Telegram API errors. Respond with a polite error message to the user on unexpected failures.

## Persistence
- SQLite database file: ./bot_data.db (relative to working directory).
- Schema (single table):
  CREATE TABLE IF NOT EXISTS users (
    user_id INTEGER PRIMARY KEY,
    username TEXT,
    first_name TEXT,
    joke_count INTEGER NOT NULL DEFAULT 0,
    last_seen TEXT
  );
- All updates run within short transactions. Single-process bot expected; SQLite is sufficient for this scale.

## Payments
- None.

## Non-goals
- No admin dashboard or web UI.
- No multi-instance shared locking or advanced scaling. Designed as a single-process bot.
- No analytics beyond per-user joke_count.
- No persistent storage of jokes fetched; only counts and basic user metadata stored.

## Implementation notes (concise)
- Language: Python 3.9+.
- Telegram library: python-telegram-bot (v13+ or v20 async) — pick v20 if using async; keep code minimal.
- HTTP: use requests (sync) or aiohttp (async) depending on chosen PTB mode. Default recommendation: python-telegram-bot v13 with requests for simplest synchronous single-file implementation.
- Logging to stdout.
- Provide a small built-in array of ~10 programming jokes as fallback.

## Assumptions & defaults
- Bot framework: python-telegram-bot (sync v13 recommended) — simplest, well-known library for a tiny bot.
- Joke source: default to JokeAPI (https://v2.jokeapi.dev/joke/Programming) with a built-in local fallback list — ensures variety and resilience.
- Database file: ./bot_data.db (SQLite) — simple persistent storage in working directory.
- Commands: expose /start, /joke, /stats, and /help — minimal UX and discoverability.
- Message format: send single message per joke (two-part jokes concatenated with a blank line) and append "\n\nYou've seen X jokes." — provides immediate feedback to user.
- Concurrency: single-process deployment; SQLite row locking is acceptable for expected low volume — scale later if needed.

