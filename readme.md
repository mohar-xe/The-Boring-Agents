# The Boring Agents

AI-powered content generation API for [The Boring Education](https://www.theboringeducation.com) platform.

## Overview

The Boring Agents is a FastAPI service that generates educational content with LLMs. It exposes four
content agents behind a common REST API:

| Agent | Prefix | What it generates |
|---|---|---|
| **Quiz** | `/api/v1/quiz` | Multiple-choice quizzes with explanations, per topic |
| **Interview** | `/api/v1/interview` | Interview question sheets and per-question answer generation |
| **DSA** | `/api/v1/dsa` | DSA question sets plus a structured study guide, per topic |
| **Aptitude** | `/api/v1/aptitude` | Aptitude question batches with answers and markdown study guides |

Generation is **asynchronous**. A create endpoint returns a `session_id` immediately and the
workflow runs in a FastAPI background task; you poll the session endpoint for progress and then
fetch the output. Sessions are resumable and retryable, so a failed run can continue from where it
stopped rather than starting over.

Everything is driven from the Admin UI over HTTP. There is no CLI.

## Architecture

```
POST /api/v1/dsa/topics
  → routes/dsa.py            request/response validation
  → controllers/dsa_controller.py   creates session, schedules BackgroundTasks
  → agents/dsa/workflow.py   LangGraph: generate_questions → generate_study_guide → finalize
  → agents/dsa/generators.py BaseAgent: format prompt → call LLM → parse JSON → normalize
  → core/llm/                provider (OpenAI or Gemini) via LangChain
  → core/session/            session state persisted to temp/dsa_sessions/<id>.json
                             final artifact written to output/dsa/dsa_<slug>.json
```

Every agent follows the same four-layer shape, which is why adding an agent means copying one
package rather than wiring new plumbing:

- **`workflow.py`** — LangGraph `StateGraph` of nodes, plus an orchestrator subclass. Resume rules
  live in `state_from_session()` and `determine_resume_status()`.
- **`generators.py`** — a `BaseAgent` subclass holding prompt templates and response parsing.
- **`validators.py`** — normalizes raw LLM output into the stable API schema.
- **`session.py`** — the agent's session manager, extending `BaseSessionManager`.

Shared infrastructure sits in `src/core/`:

| Module | Responsibility |
|---|---|
| `core/llm/` | `LLMProvider` abstraction, `PROVIDER_REGISTRY`, OpenAI and Gemini implementations |
| `core/orchestrator.py` | `BaseWorkflowOrchestrator` — session → state → `graph.invoke` → sync back |
| `core/session/` | `BaseSessionManager`, file-backed session store, progress and chunk tracking |
| `core/pipeline/` | `Job`, `ContentPipeline`, `ChunkedPipeline` (resumable generation), publish/push |
| `core/env.py`, `core/config.py` | Environment loading, validation, defaults |

### Repository layout

```
The-Boring-Agents/
├── src/
│   ├── agents/                 # one package per content agent
│   │   ├── base.py             # BaseAgent: prompts, LLM handle, truncation guard
│   │   ├── aptitude/           # questions, constants, study guide, workflow
│   │   ├── dsa/                # questions, study guide, schema, validators, workflow
│   │   ├── interview/          # sheets, questions, answers, utils, workflow
│   │   └── quiz/               # quizzes, question metadata, workflow
│   ├── api/
│   │   ├── app.py              # FastAPI app factory, router registration, /health + /ping
│   │   ├── routes/             # quiz, interview_prep, dsa, aptitude, session
│   │   ├── controllers/        # business logic, background task scheduling
│   │   ├── models/             # Pydantic request/response models
│   │   ├── middleware.py       # request/response logging, X-Request-ID
│   │   └── logging_config.py   # JSON log formatter, rotating file handler
│   ├── core/                   # llm, orchestrator, session, pipeline, env, config
│   └── utils/                  # helpers, validation, session/request logging
├── tests/                      # pytest suite (api, unit, per-agent unit + integration)
├── docs/LANGGRAPH_QUIZ_LEARNING.md
├── output/                     # generated artifacts (gitignored)
├── temp/                       # session state (gitignored)
├── logs/                       # api.log, rotated (gitignored)
├── run.py                      # uvicorn entry point
├── pyproject.toml
├── Dockerfile
└── docker-compose.yml
```

## Quick Start

### Prerequisites

- Python 3.9 or newer
- At least one supported provider key: `OPENAI_API_KEY` or `GEMINI_API_KEY`

### Installation

```bash
git clone https://github.com/The-Boring-Education/The-Boring-Agents.git
cd The-Boring-Agents

python3 -m venv .venv
source .venv/bin/activate

pip install -e .            # add dev extras for the test suite
pip install -e ".[dev]"
```

### Configure environment

Create a `.env` file in the repo root. At least one provider key is required.

```bash
cat > .env <<'EOF'
# LLM providers (at least one)
OPENAI_API_KEY=your_openai_key
GEMINI_API_KEY=your_gemini_key

# Behavior
ENVIRONMENT=dev                # local | dev | prod
LOG_LEVEL=INFO
OUTPUT_DIR=./output
TEMP_DIR=./temp
LOG_DIR=./logs

# Generation
DEFAULT_MODEL=gpt-4o-mini
TEMPERATURE=0.8
MAX_TOKENS=4000
MAX_CONTEXT_LENGTH=16000

# Server
AGENTS_API_HOST=0.0.0.0
AGENTS_API_PORT=8000
RELOAD=1

# Upload targets (used by the /upload and /push endpoints)
LOCAL_API_BASE_URL=http://localhost:3000
DEV_API_BASE_URL=https://tbe-dev-git-development-tbe.vercel.app
PROD_API_BASE_URL=https://www.theboringeducation.com

# Shared secret sent as the x-admin-secret header on upload/push
ADMIN_SECRET=TBEAdmin
EOF
```

Files are loaded in this order, later wins: `.env.local` → `.env` → process environment.

> **Note** — `DEFAULT_MODEL` must name a model the configured provider actually serves. With
> `GEMINI_API_KEY`, set it to a Gemini model id (for example `gemini-2.0-flash`); the shipped
> default, `gpt-4o-mini`, is an OpenAI model.

### Run the server

```bash
python run.py                 # http://localhost:8000, auto-reload by default
```

- Swagger UI: `http://localhost:8000/docs`
- ReDoc: `http://localhost:8000/redoc`
- OpenAPI JSON: `http://localhost:8000/openapi.json`

### Run with Docker

```bash
docker-compose up -d
docker-compose logs -f
docker-compose down
```

The compose file mounts `./output`, `./temp` and `./logs`, so generated content survives container
restarts.

## Configuration

All environment variables, their defaults and precedence rules are documented in
[`src/core/README.md`](src/core/README.md). Summary:

| Variable | Default | Purpose |
|---|---|---|
| `OPENAI_API_KEY` | — | OpenAI provider key |
| `GEMINI_API_KEY` | — | Gemini provider key |
| `DEFAULT_MODEL` | `gpt-4o-mini` | Model id passed to the provider |
| `TEMPERATURE` | `0.8` | Sampling temperature |
| `MAX_TOKENS` | `4000` | Max output tokens per call |
| `MAX_CONTEXT_LENGTH` | `16000` | Prompt budget; longer prompts are truncated with a warning |
| `ENVIRONMENT` | `dev` | `local`, `dev` or `prod`; selects the upload base URL |
| `OUTPUT_DIR` | `./output` | Final artifact directory |
| `TEMP_DIR` | `./temp` | Session state directory |
| `LOG_DIR` | `./logs` | Log file directory |
| `LOG_LEVEL` | `INFO` | Log level |
| `AGENTS_API_HOST` | `0.0.0.0` | Bind address |
| `AGENTS_API_PORT` | `8000` | Bind port |
| `RELOAD` | `1` | uvicorn auto-reload |
| `ADMIN_SECRET` | `TBEAdmin` | Value of the `x-admin-secret` header on upload/push |
| `LOCAL_API_BASE_URL` | `http://localhost:3000` | Upload target for `ENVIRONMENT=local` |
| `DEV_API_BASE_URL` | TBE dev Vercel URL | Upload target for `ENVIRONMENT=dev` |
| `PROD_API_BASE_URL` | `https://www.theboringeducation.com` | Upload target for `ENVIRONMENT=prod` |

## API

Base URL: `http://localhost:8000` locally. Upload endpoints resolve their target from
`ENVIRONMENT`.

### Health

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1` | Service metadata and endpoint index |
| `GET` | `/api/v1/health` | Health check |
| `GET` | `/api/v1/ping` | Health check with version |

### Quiz — `/api/v1/quiz`

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/v1/quiz/quizzes` | Create a quiz session |
| `GET` | `/api/v1/quiz/topics` | List supported quiz topics |
| `POST` | `/api/v1/quiz/topics` | Generate a quiz for one topic |
| `POST` | `/api/v1/quiz/generate` | Generate a quiz |
| `GET` | `/api/v1/quiz/pending` | List artifacts awaiting upload |
| `GET` | `/api/v1/quiz/pending/{filename}/content` | Read a pending artifact |
| `DELETE` | `/api/v1/quiz/pending/{filename}` | Delete a pending artifact |
| `GET` | `/api/v1/quiz/sessions` | List quiz sessions |
| `GET` | `/api/v1/quiz/sessions/{session_id}` | Session status and progress |
| `GET` | `/api/v1/quiz/sessions/{session_id}/output` | Generated quiz |
| `GET` | `/api/v1/quiz/progress/{session_id}` | Progress only |
| `POST` | `/api/v1/quiz/sessions/{session_id}/cancel` | Cancel a session |
| `POST` | `/api/v1/quiz/sessions/{session_id}/resume` | Resume a paused session |
| `POST` | `/api/v1/quiz/sessions/{session_id}/retry` | Retry a failed session |
| `DELETE` | `/api/v1/quiz/sessions/{session_id}` | Delete a session and its artifact |
| `POST` | `/api/v1/quiz/validate` | Validate a quiz against the DB schema |
| `POST` | `/api/v1/quiz/upload` | Upload a generated quiz |

`/api/v1/quiz/session/{session_id}` (singular) exists as a legacy alias of the plural routes.

### Interview — `/api/v1/interview`

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/v1/interview/sheets` | Create an interview sheet |
| `POST` | `/api/v1/interview/topics` | Generate questions for one topic |
| `POST` | `/api/v1/interview/generate-topic` | Generate questions for one topic (alias) |
| `GET` | `/api/v1/interview/topic-templates` | List topic templates |
| `GET` | `/api/v1/interview/roadmap-suggestions` | List roadmap suggestions |
| `GET` | `/api/v1/interview/sessions` | List sessions |
| `GET` | `/api/v1/interview/sessions/{session_id}` | Session status |
| `GET` | `/api/v1/interview/sessions/{session_id}/output` | Generated sheet |
| `PUT` | `/api/v1/interview/sessions/{session_id}/sheet` | Update sheet metadata |
| `GET` | `/api/v1/interview/sessions/{session_id}/questions/{question_id}` | Read one question |
| `PUT` | `/api/v1/interview/sessions/{session_id}/questions/{question_id}` | Update one question |
| `DELETE` | `/api/v1/interview/sessions/{session_id}/questions/{question_id}` | Delete one question |
| `POST` | `/api/v1/interview/sessions/{session_id}/questions` | Add a question |
| `POST` | `/api/v1/interview/sessions/{session_id}/cancel` | Cancel a session |
| `POST` | `/api/v1/interview/sessions/{session_id}/resume` | Resume a session |
| `POST` | `/api/v1/interview/sessions/{session_id}/retry` | Retry a session |
| `DELETE` | `/api/v1/interview/sessions/{session_id}` | Delete a session |
| `POST` | `/api/v1/interview/validate` | Validate generated content |
| `POST` | `/api/v1/interview/upload` | Upload generated content |

The singular `/api/v1/interview/session/{session_id}` family is a legacy alias kept for
backwards compatibility.

### DSA — `/api/v1/dsa`

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/v1/dsa/topics` | Generate questions and study guide for a topic |
| `POST` | `/api/v1/dsa/generate-topic` | Same, under the legacy path |
| `GET` | `/api/v1/dsa/sessions` | List sessions |
| `GET` | `/api/v1/dsa/sessions/{session_id}` | Session status |
| `GET` | `/api/v1/dsa/sessions/{session_id}/output` | Questions plus study guide |
| `PUT` | `/api/v1/dsa/sessions/{session_id}/questions/{question_index}` | Update one question |
| `POST` | `/api/v1/dsa/sessions/{session_id}/push` | Publish generated content to TBE-Web |
| `POST` | `/api/v1/dsa/sessions/{session_id}/cancel` | Cancel a session |
| `POST` | `/api/v1/dsa/sessions/{session_id}/resume` | Resume a session |
| `POST` | `/api/v1/dsa/sessions/{session_id}/retry` | Retry a session |
| `DELETE` | `/api/v1/dsa/sessions/{session_id}` | Delete a session |

### Aptitude — `/api/v1/aptitude`

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/v1/aptitude/generate` | Generate questions and answers for a topic |
| `POST` | `/api/v1/aptitude/generate-batch` | Generate across several topics |
| `GET` | `/api/v1/aptitude/topics` | List supported topics |
| `POST` | `/api/v1/aptitude/generate-study-guide` | Generate a markdown study guide |
| `POST` | `/api/v1/aptitude/upload-study-guide` | Upload a study guide |
| `POST` | `/api/v1/aptitude/upload` | Upload generated questions |

### Sessions — `/api/v1/sessions`

Agent-agnostic session endpoints.

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/sessions/active` | List active sessions across all agents |
| `GET` | `/api/v1/sessions/logs/{session_id}` | Recent JSONL logs for a session |
| `GET` | `/api/v1/sessions/detail/{session_id}` | Full session detail |
| `POST` | `/api/v1/sessions/resume/{session_id}` | Resume any session |
| `DELETE` | `/api/v1/sessions/{session_id}` | Delete a session and its artifacts |

### Examples

Generate a DSA package:

```bash
curl -X POST http://localhost:8000/api/v1/dsa/topics \
  -H 'Content-Type: application/json' \
  -d '{"topic": "Sliding Window", "questionCount": 20, "difficulty": "MEDIUM"}'
```

```json
{"sessionId": "0f2c...","message": "DSA generation started"}
```

Poll it, then read the result:

```bash
curl http://localhost:8000/api/v1/dsa/sessions/$SESSION_ID
curl http://localhost:8000/api/v1/dsa/sessions/$SESSION_ID/output
```

Generate an interview sheet:

```bash
curl -X POST http://localhost:8000/api/v1/interview/sheets \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "React.js Interview",
    "description": "Comprehensive React.js preparation",
    "agentType": "tech",
    "technology": "React",
    "questionCount": 30,
    "roadmap": "Frontend"
  }'
```

### Session lifecycle

`pending` → `in_progress` → `completed`, with `failed` and `cancelled` as terminal alternatives.

- **Resume** picks up where the previous run stopped: nodes whose output is already populated are
  skipped.
- **Retry** restarts a failed run from its persisted session state.
- Because state is persisted to `TEMP_DIR`, re-running a session id will not regenerate content
  that already exists. Delete the session, or point `TEMP_DIR` at a fresh directory, to force
  regeneration.

## Logging

Structured JSON logging to console and to a rotating file.

- **Console**: JSON to stdout
- **File**: `logs/api.log`, rotated at 10 MB with 5 backups
- **Levels**: `DEBUG`, `INFO`, `WARNING`, `ERROR`

```json
{
  "timestamp": "2024-12-17T22:43:13Z",
  "level": "INFO",
  "request_id": "uuid-here",
  "action": "generate_quiz",
  "topic": "React",
  "status": "success",
  "session_id": "abc123",
  "environment": "local"
}
```

Every request gets a `request_id` that appears in its logs and is returned in the `X-Request-ID`
response header. API keys and admin secrets are masked before logging.

```bash
tail -f logs/api.log
jq 'select(.level == "ERROR")' logs/api.log
jq 'select(.request_id == "request-id-from-header")' logs/api.log
```

Session-scoped logs are also available over HTTP at
`GET /api/v1/sessions/logs/{session_id}?limit=100`.

## Development

```bash
export RELOAD=1     # auto-reload
python run.py

export RELOAD=0     # no reload
python run.py
```

### Tests

```bash
pip install -e ".[dev]"
pytest                              # whole suite
pytest tests/unit                   # core and utils
pytest tests/api                    # API layer
pytest tests/aptitude tests/quiz    # per-agent
```

See [`tests/README.md`](tests/README.md) for suite layout and conventions. `tests/conftest.py`
provides the shared fixtures: an in-process `TestClient`, temp directories that isolate
`TEMP_DIR`/`OUTPUT_DIR`, and mocked LLM providers so tests never make network calls.

### Adding an agent

1. Create `src/agents/<name>/` with `prompts.py`, `generators.py`, `validators.py`, `session.py`
   and `workflow.py`.
2. Subclass `BaseWorkflowOrchestrator`; implement `state_from_session()` and
   `determine_resume_status()` for resume support.
3. Register a router in `src/api/routes/` and include it in `src/api/app.py`.

## Docker

```bash
docker build -t the-boring-agents .

docker run -d \
  -p 8000:8000 \
  -v $(pwd)/output:/app/output \
  -v $(pwd)/temp:/app/temp \
  -v $(pwd)/logs:/app/logs \
  -e OPENAI_API_KEY=your_key \
  the-boring-agents
```

Or use compose, which reads `./.env`:

```yaml
environment:
  - OPENAI_API_KEY=${OPENAI_API_KEY}
  - GEMINI_API_KEY=${GEMINI_API_KEY}
  - ENVIRONMENT=prod
  - LOG_LEVEL=INFO
```

## Troubleshooting

**Server will not start**
1. Confirm at least one of `OPENAI_API_KEY` / `GEMINI_API_KEY` is set
2. Confirm `AGENTS_API_PORT` (8000) is free
3. Check `logs/api.log`

**No log file**
1. Check `LOG_DIR` and write permissions
2. Confirm `LOG_LEVEL` is set

**Provider errors**
1. Confirm the key is valid and not expired
2. Confirm `DEFAULT_MODEL` is a model your provider serves — an OpenAI model id with
   `GEMINI_API_KEY` set will fail
3. Check the request duration against `MAX_TOKENS` and `MAX_CONTEXT_LENGTH`

**Session stuck in `in_progress`**
Sessions run as FastAPI background tasks in-process. A worker restart leaves them incomplete —
resume or retry the session. Generation is slow for large `questionCount` values.

## Contributing

Issues and pull requests are welcome. Please keep changes scoped, and update `readme.md` when you
change the API surface or configuration.

## License

MIT, as declared in `pyproject.toml`. Note that no `LICENSE` file is present in the repository yet
— adding one would be a welcome contribution.