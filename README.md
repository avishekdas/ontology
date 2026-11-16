# NVIDIA Auto Ontology — local setup

A working local build of [NVIDIA Auto Ontology](https://github.com/NVIDIA/auto-ontology) with the
Wide World Importers sample database from
[auto-ontology-eval](https://github.com/NVIDIA/auto-ontology-eval), following
[syedair/youtube-content › nvidia-auto-ontology](https://github.com/syedair/youtube-content/tree/main/nvidia-auto-ontology).

| Folder | Source | Upstream commit |
|---|---|---|
| `auto-ontology/` | NVIDIA/auto-ontology (Apache 2.0) | `62337897` + local fixes below |
| `auto-ontology-eval/` | NVIDIA/auto-ontology-eval (Apache 2.0) | `0b88bbd` (unchanged) |

No `.env` files are committed. Create them from each folder's `.env.example`.

## Local changes to `auto-ontology/`

1. **`LLM_CLIENT=openai`** (`auto_ontology/utils/llm_invoke.py`): the hosted build.nvidia.com API
   rejects `nvext.guided_json` (HTTP 400). This routes chat models through `ChatOpenAI`, caps
   Nemotron thinking tokens, and skips `temperature` for Claude / GPT-5. From the video's
   `fixes.patch`.
2. **All-NULL key columns** (`auto_ontology/semantic/visit_enter.py`): the key-value probe got a
   column-less DataFrame for an all-NULL column and raised `IndexError`, dropping the table
   (`sales.specialdeals`) from semantic compilation. Fixed, with a unit test.

The video's third fix (cross-schema foreign keys) is already merged upstream (PR #479).

## Setup

Needs Docker Desktop, git and [uv](https://docs.astral.sh/uv/) (`uv self update` first).

```bash
cd auto-ontology
cp .env.example .env
```

Edit `auto-ontology/.env`:

```dotenv
POSTGRES_PORT=5433                      # host port; use 5432 if nothing else is on it
POSTGRES_PASSWORD=<openssl rand -hex 16>
PGADMIN_PASSWORD=<random>
CONNECTION_STRINGS=                     # empty: add the DB in Settings → Connections
DEFAULT_MODELS_API_KEY=nvapi-...        # build.nvidia.com key
EMBED_MODEL=nvidia/nemotron-3-embed-1b  # 3.5-embed fails on batched requests (HTTP 500)
AUTO_ONTOLOGY_ADMIN_PASSWORD=<random>   # UI login, user admin@nvidia.com
AUTH_SECRET=<openssl rand -hex 32>
LLM_CLIENT=openai
```

Delete the `REASONING_*`, `NON_REASONING_*`, `EMBED_API_KEY` and `RERANK_API_KEY` lines so they
fall back to the `DEFAULT_MODELS_*` values (NVIDIA Nemotron).

Optional, Claude Sonnet for the chat roles (needs Anthropic API credits):

```dotenv
REASONING_ENDPOINT=https://api.anthropic.com/v1/
REASONING_MODEL=claude-sonnet-5-5
REASONING_API_KEY=sk-ant-...
NON_REASONING_ENDPOINT=https://api.anthropic.com/v1/
NON_REASONING_MODEL=claude-sonnet-5-5
NON_REASONING_API_KEY=sk-ant-...
```

Load the sample data and start:

```bash
docker compose up -d --wait postgres
cd ../auto-ontology-eval && cp .env.example .env
# set POSTGRES_PORT and POSTGRES_PASSWORD to the same values, then:
uv sync && uv run python scripts/seed/seed_wwi.py --database-name wideworldimporters --drop
cd ../auto-ontology && docker compose up -d --build --wait
```

Open http://localhost:3000 and sign in as `admin@nvidia.com`:

1. **Settings → Connections → PostgreSQL**: host `postgres`, port `5432`, user `postgres`, your
   password, database `wideworldimporters`, table denylist `_archive$`. Create.
2. **Settings → Semantic Compilation**: turn on *Enable Semantic Compilation*. Takes ~13 min for
   the 34 tables with Nemotron.

Expected: 98 foreign keys in the catalog, 59 of them cross-schema.

After switching chat models: `docker compose up -d`, then **Reset** in Settings → Semantic
Compilation (no re-ingest needed).

## Example queries (custom analyses)

Example queries raise accuracy a lot. [`examples/custom_analyses.json`](examples/custom_analyses.json)
holds the ones used here, in the same format as `auto-ontology-eval/datasets/*/custom_analyses.json`.
They live in the app's database, so add each one after setup: **Analysis → Create New SQL
Analysis**, paste name, description and SQL, **Validate SQL**, **Save**. Each is embedded right
away; no recompile needed.

Test question: *"Which 5 customers had the highest total invoiced amount in 2015, and which city
do they deliver to?"* Correct top 5: Laszlo Gardenier (Marion Hill), Knut Svensson (Madison Park),
Wingtip Toys (Montoya, NM), Svetlana Todorovic (Venersborg), Wingtip Toys (Bokeelia, FL).

| Nemotron (NVIDIA default) | Amount | City | Top 5 |
|---|---|---|---|
| Without example | summed item counts | `postalcityid` | wrong |
| With example | `SUM(extendedprice)` | `deliverycityid` | correct |

Both runs took about 5 minutes.
