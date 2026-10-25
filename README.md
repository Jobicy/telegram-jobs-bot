# Jobicy Telegram Bot

Publish new matching remote jobs to a Telegram channel without reposting existing listings.

## Requirements

Node.js 20.12+, a Telegram account, a Telegram bot, and a channel or chat.

## Create and configure the bot

1. In Telegram, open [@BotFather](https://t.me/BotFather), send `/newbot`, and follow the prompts.
2. Copy the token BotFather returns.
3. Create or open your Telegram channel and add the bot as an administrator allowed to post messages.
4. Use a public channel username such as `@your_channel` as `TELEGRAM_CHAT_ID`. For a private channel, retrieve its numeric chat ID from a bot update after adding the bot and posting a message.

```bash
git clone https://github.com/Jobicy/telegram-jobs-bot.git
cd telegram-jobs-bot
cp .env.example .env
npm install
npm start
```

Set `TELEGRAM_BOT_TOKEN` and `TELEGRAM_CHAT_ID` in `.env`. Optional `JOBICY_GEO` and `JOBICY_INDUSTRY` use the official Jobicy slugs. `JOBICY_KEYWORDS` accepts comma-separated case-insensitive alternatives, for example `python,backend,platform`.

The default and minimum synchronization interval is one hour (`CHECK_INTERVAL_SECONDS=3600`). Each pass requests pages of 100 jobs using `nextCursor` until the seven-day feed is exhausted. Filters remain fixed within a pass. HTTP 429 responses extend the delay using `Retry-After` when available; failed traversals are retried from the first page on a later pass.

On the first successful full traversal, the bot records current matching job IDs without publishing them. Later polls publish only newly appearing listings. State is stored in `data/seen-jobs.json`, survives restarts, retains IDs belonging to the current matching seven-day feed, and is rebuilt safely if corrupt. Failed Telegram deliveries are not marked as sent and will be retried on a later check.

Messages use escaped Telegram HTML, preserve the original Jobicy listing URL, and include a discreet Jobicy attribution link.

IDs are pruned only after a successful full traversal, avoiding duplicates caused by a fixed ID-count cap. If you change filters and want a new silent baseline, stop the bot and remove its state file before restarting. Cursors are used only during a pass, expire after 24 hours, and are never stored as a long-term polling checkpoint.

[OpenAPI JSON](https://jobicy.com/api/openapi.json) · [OpenAPI YAML](https://jobicy.com/api/openapi.yaml)

## Stored listings

If you extend this integration to retain listings, use the [batch status endpoint](https://github.com/Jobicy/remote-jobs-api#batch-status-check) to check up to 100 IDs at a time. Older open jobs can remain `active` after leaving the seven-day feed. Do not mark jobs closed just because a fresh feed omits them, or because a check returns `unknown` or fails. The sample does not automatically reconcile stored listings.

## Developer resources

[All integrations](https://github.com/Jobicy/jobicy-api-examples) · [API contract](https://github.com/Jobicy/remote-jobs-api) · [MCP](https://github.com/Jobicy/remote-jobs-mcp-server)

Public interfaces must show **[Jobs powered by Jobicy](https://jobicy.com/)** and preserve original listing URLs. Start automated feed synchronization at most once per hour.

## License

[MIT](LICENSE) for code; see [source and data terms](PROVENANCE.md).

## Run with Docker Compose

```bash
cp .env.example .env
# Set your Telegram token, destination and optional filters in .env.
docker compose up -d --build
docker compose logs -f
```

Keep the `jobicy-data` volume: it stores delivered IDs across restarts. Run only one bot instance per state volume. `docker compose down` keeps the volume; deleting it creates a new silent baseline. A successful Telegram delivery followed by a process crash before the state write can still repeat a message. The bot does not guarantee exactly-once delivery.

Alternatively, run `npm ci && npm start` with a process supervisor on any Node.js 20.12+ host and preserve the `data/` directory. No GitHub Actions schedule is needed.

## Checks

```bash
npm ci
npm test
npm run check
```

Tests cover all-page traversal, local keyword filtering, cursor cycles and persisted deduplication. Telegram token permissions and delivery must be verified in your own channel. Tests never send Telegram messages.
