# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

```bash
# Install dependencies (creates venv automatically)
uv sync

# Install pre-commit hooks
uv run pre-commit install

# Format code
uv run ruff format

# Lint (auto-fix)
uv run ruff check --fix

# Type check
uv run pyright

# Run all tests (parallel)
uv run pytest -n auto -v

# Run a single test file
uv run pytest tests/sync/chat_test.py -v

# Run a single test by name
uv run pytest tests/sync/chat_test.py::test_unary -v

# Run async tests only
uv run pytest tests/aio/ -v

# Check formatting without fixing
uv run ruff format --check

# Check linting without fixing
uv run ruff check
```

The CI pipeline (`.github/workflows/ci.yaml`) runs format check, lint, pyright, and pytest across Python 3.10–3.13.

## Architecture Overview

### Package Structure

The SDK is a gRPC-based Python library with a strict **sync/async mirror pattern**. Every API surface exists in both `src/xai_sdk/sync/` and `src/xai_sdk/aio/`, backed by shared base classes in `src/xai_sdk/`.

```
src/xai_sdk/
├── __init__.py          # Exports Client (sync) and AsyncClient (async)
├── client.py            # BaseClient: shared gRPC channel setup, retry policy, auth
├── chat.py              # Shared BaseClient + BaseChat + all helpers (user, system, tool, etc.)
├── interceptors.py      # Sync + async gRPC interceptors (auth, timeout)
├── meta.py              # ProtoDecorator generic base for all response wrappers
├── telemetry/           # OpenTelemetry integration (get_tracer, Telemetry class)
├── types/               # Typed literals/enums re-exported from proto (ChatModel, ToolMode, etc.)
├── proto/               # Generated protobuf files
│   ├── v5/              # protobuf v5 generated code
│   └── v6/              # protobuf v6 generated code
├── sync/                # Synchronous implementations
│   ├── client.py        # Concrete Client: wires channels to sub-clients
│   └── chat.py          # Chat.sample(), .stream(), .parse(), .defer()
└── aio/                 # Async mirror of sync/
    ├── client.py
    └── chat.py          # async def sample(), stream(), parse(), defer()
```

### Proto Compatibility Layer

`src/xai_sdk/proto/__init__.py` detects the installed `google.protobuf` major version (5 or 6) at import time and selects the matching generated code from `proto/v5/` or `proto/v6/`. Proto files are pre-generated and committed — do not edit them manually.

### Client Hierarchy

`BaseClient` (in `client.py`) configures gRPC channel options (20 MiB message limits, keepalives, retry policy), sets SDK version metadata on all requests, and delegates concrete channel creation to subclasses. `sync.Client` and `aio.Client` each implement `_init()` to create the gRPC channel and instantiate sub-clients (`chat`, `image`, `video`, `files`, `models`, `batch`, `tokenize`, `collections`, `auth`).

Two separate gRPC channels exist: one for `api.x.ai` (API key via `XAI_API_KEY`) and one for `management-api.x.ai` (management key via `XAI_MANAGEMENT_KEY`, required only for Collections operations).

### Chat API Design

`BaseChat` (in `chat.py`) owns the mutable `GetCompletionsRequest` proto and is the base for both `sync.Chat` and `aio.Chat`. The `create()` factory on `BaseClient` only initializes the request — no network call. `sample()`, `stream()`, `parse()`, and `defer()` perform the actual RPCs.

`Response` wraps `GetChatCompletionResponse` via `ProtoDecorator`. A single response proto can hold multiple outputs (for server-side agentic tool calls). The `_index` field on `Response` controls whether convenience properties (`content`, `tool_calls`, etc.) expose a single output (index=0) or all outputs (index=None). Index auto-detection (`_auto_detect_multi_output_mode`) switches to multi-output mode when the server implicitly adds tool outputs.

Streaming accumulates content in string buffers (`_content_buffers`) for efficiency; they are flushed to the proto on demand via `_sync_buffers_to_proto()`.

### Message Helpers

All message construction helpers are in `chat.py` and produce `chat_pb2.Message` or `chat_pb2.Content` objects:
- `user()`, `system()`, `assistant()`, `developer()`, `tool_result()` — create messages
- `text()`, `image()`, `file()` — create content objects
- `tool()`, `required_tool()` — create tool definitions / tool choices

### ProtoDecorator Pattern

All response objects inherit from `ProtoDecorator[T]` in `meta.py`. It holds `._proto` privately and exposes it via the `.proto` property. This allows accessing the raw proto when needed (`response.proto`), while Pythonic properties provide the common interface.

### Telemetry

`src/xai_sdk/telemetry/config.py` provides `get_tracer()` (respects `XAI_SDK_DISABLE_TRACING=1`) and `Telemetry` class for configuring OpenTelemetry span export. Every RPC method wraps its call in an OTel span with standardized GenAI semantic convention attributes. Sensitive attributes (prompt content, completions) are omitted when `XAI_SDK_DISABLE_SENSITIVE_TELEMETRY_ATTRIBUTES=1`.

### Test Infrastructure

Tests run against an in-process gRPC server defined in `tests/server.py`. The `TestServer` class implements all service stubs with dummy responses and runs in a background thread. Tests use `pytest` fixtures with `scope="session"` for server lifecycle, `scope="module"` or `scope="session"` for clients.

The test server uses `portpicker` to allocate ports dynamically. It validates the `authorization: Bearer 123` header on all requests (management endpoints use `Bearer 456`). The `asyncio_mode = "strict"` pytest-asyncio config requires explicit `@pytest.mark.asyncio` on async tests.

## Key Conventions

- **Sync/async symmetry**: every feature in `sync/` must have a matching implementation in `aio/`. The shared base classes in the root package contain all business logic; sync/async subclasses only differ in their RPC call style (`stub.Method(request)` vs `await stub.Method(request)`).
- **No direct proto edits**: proto files in `proto/v5/` and `proto/v6/` are generated artifacts.
- **Ruff excludes `src/xai_sdk/proto`**: linting and formatting do not apply to generated proto code.
- **Relative imports are banned** in `src/` (enforced by `TID252` ban-relative-imports = "all" in ruff config, though `TID252` is also in the ignore list — absolute imports are the norm).
- **Type literals over enums**: public-facing parameters use `Literal` types (e.g., `ReasoningEffort = Literal["low", "high"]`) that are converted to proto enums internally. Proto enum values are not exposed in the public API.
- **Version**: managed via `src/xai_sdk/__about__.py` using hatch. Bumping this file on `main` auto-tags via GitHub Actions.
