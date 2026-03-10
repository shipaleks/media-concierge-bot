# Media Concierge Bot

AI-powered Telegram bot that acts as a personal cinema concierge — finds movies and TV shows through natural conversation, downloads them to your seedbox, and learns your taste over time.

Built on Claude API with tool_use, it orchestrates searches across torrent trackers, fetches rich metadata from TMDB/Kinopoisk, and maintains a MemGPT-style persistent memory of each user's preferences and watch history.

## What It Does

Talk to the bot in natural language (Russian or English). It understands context, remembers your preferences, and handles everything from discovery to download:

```
You:  Хочу что-то похожее на Интерстеллар, но чтобы не больше двух часов

Bot:  Отличный запрос! Вот что приходит на ум из научной фантастики с похожей
      атмосферой, но компактнее:

      1. Arrival (Прибытие, 2016) — 116 мин. Вильнёв, как и Нолан, строит
         sci-fi вокруг большой идеи: здесь это лингвистика и восприятие времени.
         TMDB: 7.9 | KP: 7.6

      2. Moon (Луна 2112, 2009) — 97 мин. Камерная история Данкана Джонса
         с Сэмом Рокуэллом в одиночку на лунной базе. Минимализм, который
         работает мощнее спецэффектов.
         TMDB: 7.6 | KP: 7.4

      [🔍 Искать на Rutracker] [🔍 Искать на PirateBay]
```

```
You:  Скачай Arrival в 4K

Bot:  Нашёл на Rutracker:
      🎬 Прибытие / Arrival (2016) 2160p UHD BDRemux HDR — 55.3 GB
      Seeders: 42 | Качество: отличное

      [⬇️ Скачать на Seedbox] [🧲 Magnet-ссылка]
```

```
You:  Посмотрел, 9 из 10. Потрясающий финал

Bot:  Записал! Arrival — 9/10. Вильнёв у тебя стабильно высоко: Дюна 8,
      Бегущий по лезвию 2049 — 9. Похоже, тебе нравится его подход к sci-fi.
      Хочешь, поставлю на мониторинг его следующий проект?
```

## Features

### Core
- **Natural language search** — ask for movies by mood, similarity, director, genre, decade, or any combination
- **Multi-tracker search** — Rutracker, PirateBay, and TorAPI (Kinozal, RuTor, NoNameClub) searched in parallel
- **Rich metadata** — ratings from TMDB, Kinopoisk, IMDB, Rotten Tomatoes; cast, crew, trailers
- **Entity cards** — tap on any movie, show, or person mentioned to see a rich info card with poster
- **Streaming responses** — real-time progressive message updates, no waiting for full response

### Personalization (MemGPT-style Memory)
- **Core memory** — 8 persistent memory blocks (preferences, watch context, style, instructions, blocklist, learnings) always in context
- **Recall memory** — searchable session summaries and user notes
- **Archival memory** — long-term storage with automatic pruning
- **Learning detector** — automatically spots patterns in your viewing history ("you tend to rate Korean thrillers above 8")
- **Auto-compaction** — memory blocks compact at 70% capacity to stay within token limits

### Downloads & Monitoring
- **One-click seedbox download** — sends magnet to Transmission, qBittorrent, or Deluge
- **Per-user seedbox config** — each user connects their own seedbox, credentials encrypted with Fernet
- **NAS sync** — optional daemon syncs completed downloads from seedbox to local NAS, sorts into Movies/TV folders
- **Release monitoring** — track upcoming releases; bot checks trackers automatically and notifies when available
- **Download tracking** — get notified when torrent finishes downloading, and again when synced to NAS

### Tracking & Social
- **Watchlist** — add/remove titles, see what's on your list
- **Ratings** — rate watched content, build a viewing history
- **Letterboxd integration** — import watchlist and ratings from Letterboxd export
- **Personalized digests** — daily and weekly cinema news digests powered by Claude

### Bot Commands
| Command | Description |
|---------|-------------|
| `/start` | Welcome message and onboarding |
| `/help` | List available commands |
| `/profile` | View your preferences and stats |
| `/settings` | Adjust quality, language, notification preferences |
| `/model` | Choose AI model (Haiku/Sonnet/Opus) and thinking budget |
| `/seedbox` | Configure your seedbox connection |
| `/rutracker` | Set your Rutracker credentials |
| `/library` | Browse your NAS media library |
| `/digest` | Get a personalized cinema news digest |
| `/reset_profile` | Start fresh with preferences |

Or just talk naturally — the bot figures out what you need.

## Architecture

```
┌──────────────┐     ┌──────────────┐     ┌──────────────────────────────┐
│   Telegram    │────▶│   Bot Module  │────▶│     Claude API (tool_use)    │
│   User        │◀────│  streaming.py │◀────│     + System Prompt          │
└──────────────┘     └──────┬───────┘     │     + Memory Context          │
                            │              └──────────┬───────────────────┘
                            │                         │ tool calls
                            ▼                         ▼
                     ┌──────────────┐     ┌──────────────────────────────┐
                     │  User Store  │     │       Tool Executor           │
                     │  (Postgres/  │     │                              │
                     │   SQLite)    │     │  Search ──▶ Rutracker        │
                     │              │     │            PirateBay         │
                     │  - profiles  │     │            TorAPI            │
                     │  - memory    │     │                              │
                     │  - watchlist │     │  Media ───▶ TMDB             │
                     │  - ratings   │     │            Kinopoisk         │
                     │  - monitors  │     │            OMDB              │
                     │  - downloads │     │                              │
                     └──────────────┘     │  Seedbox ─▶ Deluge           │
                                          │            Transmission      │
                                          │            qBittorrent       │
                                          │                              │
                                          │  Memory ──▶ Core memory      │
                                          │            Notes             │
                                          │            Search            │
                                          └──────────────────────────────┘

Background:
┌──────────────────────────────────────────────────────┐
│  APScheduler                                         │
│  - Release checker (every 6h, smart intervals)       │
│  - Torrent monitor (every 60s, checks Deluge)        │
│  - Daily Deluge cleanup                              │
│  - News digest delivery (daily 19:00, weekly Tue/Fri)│
└──────────────────────────────────────────────────────┘
```

### How a Search Works

1. User sends a message in Telegram
2. `conversation.py` loads user's memory context and preferences
3. `claude_client.py` sends message + tools + system prompt to Claude API (streaming)
4. Claude returns `tool_use` blocks (e.g., `rutracker_search`, `tmdb_search`)
5. `tools.py` ToolExecutor routes each call to the right handler
6. Results go back to Claude for natural language formatting
7. `streaming.py` progressively updates the Telegram message as tokens arrive
8. User sees inline buttons for download, more info, etc.

### Module Map

```
src/
├── bot/                    # Telegram interface layer
│   ├── main.py             # Entry point: webhook (prod) or polling (dev)
│   ├── conversation.py     # Message handler, tool dispatch, download callbacks
│   ├── streaming.py        # Progressive message updates, markdown→HTML
│   ├── handlers.py         # /help, /profile, /settings, /digest commands
│   ├── onboarding.py       # First-run setup wizard
│   ├── entity_cards.py     # Movie/TV/person info cards with inline keyboards
│   ├── model_settings.py   # /model command: choose AI model + thinking budget
│   ├── library.py          # /library: browse NAS media index
│   ├── rutracker_auth.py   # Per-user Rutracker credentials flow
│   ├── seedbox_auth.py     # Per-user seedbox credentials flow
│   └── sync_api.py         # HTTP API for NAS sync daemon
│
├── ai/                     # Claude API integration
│   ├── claude_client.py    # Async streaming client, conversation history mgmt
│   ├── tools.py            # 20+ tool definitions (JSON schema) + ToolExecutor
│   └── prompts.py          # System prompt builder with memory injection
│
├── search/                 # Torrent tracker clients
│   ├── rutracker.py        # Rutracker: search, quality filter, magnet extraction
│   ├── piratebay.py        # PirateBay: international content fallback
│   └── torapi.py           # TorAPI: unified API for RuTracker/Kinozal/RuTor/NNC
│
├── media/                  # Metadata APIs (all responses cached)
│   ├── tmdb.py             # TMDB: movie/TV/person search, credits, images
│   ├── kinopoisk.py        # Kinopoisk: Russian ratings, box office
│   └── omdb.py             # OMDB: IMDB, Rotten Tomatoes, Metacritic
│
├── seedbox/                # Torrent client integration
│   ├── client.py           # DelugeClient, TransmissionClient, QBittorrentClient
│   └── __init__.py         # send_magnet_to_seedbox() convenience functions
│
├── user/                   # Data persistence and personalization
│   ├── storage.py          # Dual-backend: PostgresStorage + SQLiteStorage
│   ├── memory.py           # MemGPT memory: CoreMemoryManager, SessionManager
│   └── profile.py          # Renders user profile as markdown for system prompt
│
├── services/               # External service integrations
│   ├── letterboxd.py       # Letterboxd OAuth client
│   ├── letterboxd_export.py # Letterboxd CSV/ZIP import parser
│   ├── letterboxd_rss.py   # Letterboxd RSS activity monitor
│   └── news.py             # News sources for digests
│
├── monitoring/             # Background jobs (APScheduler)
│   ├── scheduler.py        # Job management and scheduling
│   ├── checker.py          # Release availability checker (smart intervals)
│   ├── torrent_monitor.py  # Deluge download completion detector
│   └── news_digest.py      # Claude-powered personalized news digests
│
├── config.py               # pydantic-settings: all env vars, SecretStr
└── logger.py               # structlog: JSON (prod) / colored console (dev)
```

## Setup

### 1. Get API Keys

| Service | Required | Where to get | What for |
|---------|----------|--------------|----------|
| Telegram Bot Token | Yes | [@BotFather](https://t.me/botfather) | Bot identity |
| Anthropic API Key | Yes | [console.anthropic.com](https://console.anthropic.com/settings/keys) | Claude AI |
| TMDB API Key | Yes | [themoviedb.org/settings/api](https://www.themoviedb.org/settings/api) | Movie/TV metadata |
| Kinopoisk API Token | Yes | [kinopoiskapiunofficial.tech](https://kinopoiskapiunofficial.tech/) | Russian ratings |
| Encryption Key | Yes | Generate (see below) | Encrypt user credentials |
| OMDB API Key | No | [omdbapi.com](https://www.omdbapi.com/apikey.aspx) | IMDB/RT ratings |

Generate encryption key:
```bash
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
```

### 2. Local Development

```bash
git clone https://github.com/shipaleks/media-concierge-bot.git
cd media-concierge-bot

python -m venv venv
source venv/bin/activate

pip install -e ".[dev]"

cp .env.example .env
# Edit .env with your API keys

python -m src.bot.main  # Starts in polling mode (no webhook needed)
```

The bot runs in **polling mode** locally (no public URL needed) and **webhook mode** in production.

### 3. Production Deployment (Koyeb)

The bot is designed for [Koyeb](https://www.koyeb.com/) (free tier works):

1. Push to GitHub
2. Create Koyeb secrets for all API keys
3. Deploy from GitHub repo using the included `Dockerfile` and `koyeb.yaml`
4. Set `WEBHOOK_URL` to your Koyeb app URL

Two ports: **8000** (Telegram webhook) and **8080** (health check + sync API).

See [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md) for step-by-step instructions.

### 4. Optional: Seedbox

Users configure their seedbox via the `/seedbox` command in Telegram. Supported clients:
- **Deluge** (JSON-RPC) — recommended, used by Ultra.cc and similar providers
- **Transmission** (RPC API)
- **qBittorrent** (Web API)

Without a seedbox, the bot shows magnet links directly.

See [docs/SEEDBOX_SETUP.md](docs/SEEDBOX_SETUP.md) for setup guide.

### 5. Optional: NAS Sync

If you want automatic seedbox → NAS file transfer:
1. Set up a Linux VM or server with NAS access
2. Configure the sync scripts from `scripts/`
3. The daemon polls the bot's API for completed downloads, rsyncs them, and sorts into Movies/TV folders

See [docs/NAS_VM_SETUP.md](docs/NAS_VM_SETUP.md) for setup guide.

## Configuration Reference

All configuration is via environment variables (see `.env.example` for the full template):

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `TELEGRAM_BOT_TOKEN` | Yes | — | Bot token from @BotFather |
| `ANTHROPIC_API_KEY` | Yes | — | Claude API key |
| `TMDB_API_KEY` | Yes | — | TMDB API key |
| `KINOPOISK_API_TOKEN` | Yes | — | Kinopoisk API token |
| `ENCRYPTION_KEY` | Yes | — | Fernet key for credential encryption |
| `DATABASE_URL` | No | SQLite | PostgreSQL URL (recommended for production) |
| `BOT_USERNAME` | No | `yourbotname` | Bot username for deep links (without @) |
| `WEBHOOK_URL` | No | — | Public URL for webhook mode |
| `ENVIRONMENT` | No | `production` | `development` or `production` |
| `LOG_LEVEL` | No | `INFO` | `DEBUG`, `INFO`, `WARNING`, `ERROR` |

Optional integrations: `OMDB_API_KEY`, `RUTRACKER_USERNAME`/`PASSWORD`, `SEEDBOX_HOST`/`USER`/`PASSWORD`, `SYNC_API_KEY`, `LETTERBOXD_CLIENT_ID`/`SECRET`, `YANDEX_SEARCH_API_KEY`/`FOLDER_ID`.

## Key Design Decisions

**Why Claude with tool_use?** Instead of hardcoding search/download flows, Claude decides which tools to call based on natural language. This means the bot handles unexpected queries gracefully — "find something like the movie we discussed last week" works because Claude has conversation history and memory context.

**Why MemGPT-style memory?** Traditional chatbots lose context between sessions. The memory system gives Claude persistent knowledge about each user: their taste, watch history, quality preferences, and even communication style. The bot gets better the more you use it.

**Why dual storage (Postgres/SQLite)?** SQLite for zero-config local development, Postgres for production persistence. Same interface, swap via one env var.

**Why HTML for Telegram?** Telegram's Markdown parser is unreliable with special characters in URLs, parentheses, and underscores. HTML mode is stable and predictable. The bot's `streaming.py` converts Claude's markdown to Telegram HTML on the fly.

**Why streaming?** Claude can take several seconds to respond. Streaming shows the response progressively (word by word) so the user sees immediate feedback instead of a loading spinner.

## Development

```bash
# Lint and format
ruff check . --fix && ruff format .

# Run all tests
pytest -v

# Run single test file
pytest tests/test_rutracker.py -v

# Tests with coverage
pytest --cov=src --cov-report=term
```

11 test files covering all major modules. Tests use `pytest-asyncio` (auto mode) and mock external APIs.

## Documentation

| Document | Description | Best for |
|----------|-------------|----------|
| [CLAUDE.md](CLAUDE.md) | Developer guide: conventions, patterns, pitfalls | Contributing to the codebase |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | System design diagrams and data flows | Understanding the big picture |
| [docs/FEATURES.md](docs/FEATURES.md) | Complete feature specification with scenarios | Product understanding |
| [docs/API.md](docs/API.md) | Tool definitions and parameters | Working with Claude tools |
| [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md) | Step-by-step Koyeb deployment | Going to production |
| [docs/SEEDBOX_SETUP.md](docs/SEEDBOX_SETUP.md) | Seedbox client configuration | Setting up downloads |
| [docs/NAS_VM_SETUP.md](docs/NAS_VM_SETUP.md) | NAS sync daemon setup | Automatic file transfer |

## Tech Stack

- **Python 3.11+** — async/await everywhere
- **python-telegram-bot** — Telegram API client with webhook support
- **anthropic** — Claude API with streaming and tool_use
- **httpx** — Async HTTP client
- **pydantic / pydantic-settings** — Config validation, data models
- **asyncpg / aiosqlite** — Async database drivers
- **APScheduler** — Background job scheduling
- **BeautifulSoup + lxml** — HTML parsing for tracker scraping
- **structlog** — Structured logging (JSON in prod)
- **cryptography (Fernet)** — Credential encryption
- **Docker** — Multi-stage build, Python 3.11-slim

## License

[MIT](LICENSE)
