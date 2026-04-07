# AGENTS.md

Guidance for agentic coding agents working in this repository.

## 对话语言规则

与用户交流时，除代码、专业技术术语（如 FastAPI、Pydantic、Redis、WebSocket、ReAct 等）外，**所有回复内容尽可能使用中文**。错误分析、方案说明、步骤描述、注释解释均须用中文表达。

## Project Overview

**Multi-Agent System** — FastAPI + async/await, Manager-Worker pattern, pluggable LLM providers.

- **Language**: Python 3.10+
- **Framework**: FastAPI, Pydantic v2, Loguru, redis.asyncio
- **AI Providers**: Qwen, GLM, Kimi, MiniMax, OpenAI, Anthropic
- **Architecture**: `OrchestratorAgent` singleton (ReAct loop) → `ToolManager` / `SkillManager` → `HistoryManager` (Redis) → `EventDispatcher` (WebSocket)

## Repository Layout

```
code/                        # All runnable code lives here
├── src/
│   ├── main.py              # FastAPI app + lifespan init
│   ├── core/                # config.py, logger.py, exceptions.py
│   ├── models/              # Pydantic models: messages, events, tools, requests
│   ├── agents/              # base.py, orchestrator.py, manager.py, config.py
│   ├── processors/          # base.py, factory.py, qwen.py, glm.py, kimi.py, minimax.py
│   ├── tools/               # manager.py, core/, domain/
│   ├── skills/              # manager.py, core/, domain/
│   ├── history/             # manager.py  — Redis-backed session history singleton
│   ├── websocket/           # handlers.py, dispatcher.py, session.py, auth.py
│   ├── api/                 # routes/agents.py, routes/websocket.py
│   ├── elastic/             # client.py, service.py
│   └── hooks/               # hookable.py, registry.py, executor.py
├── tests/
│   ├── conftest.py          # session-scoped event_loop, logger setup
│   ├── unit/                # mirrors src/ structure
│   └── e2e/                 # test_websocket.py + websocket_client.py
├── .env                     # real secrets (never commit)
├── .env.example             # template — copy to .env
├── pyproject.toml           # black / isort / mypy / pytest / coverage config
└── pytest.ini               # active pytest config (overrides pyproject.toml)
docs/                        # Chinese documentation (design, requirements, testing)
```

## Environment Setup

```bash
cd code
pip install -r requirements.txt
cp .env.example .env          # then fill in API keys and Redis credentials
```

Required `.env` fields: `SECRET_KEY`, at least one `*_API_KEY`, `REDIS_HOST`, `REDIS_PORT`, `REDIS_PASSWORD`, `REDIS_DB`.

## Running the Server

```bash
# from code/
uvicorn src.main:app --reload --host 0.0.0.0 --port 8000
```

E2E tests require the server to be running. They skip automatically when it is not.

## Build / Lint / Test Commands

All commands run from the `code/` directory.

### Format & Lint

```bash
black src/ tests/             # line length = 100
isort src/ tests/             # profile = black
flake8 src/ tests/            # follows black config
mypy src/                     # see [tool.mypy] in pyproject.toml
```

### Testing

```bash
# Run all tests (unit + e2e)
pytest

# Run a single test file
pytest tests/unit/processors/test_qwen_processor.py -v -s

# Run a single test class
pytest tests/unit/processors/test_qwen_processor.py::TestQwenProcessor -v -s

# Run a single test function
pytest tests/unit/processors/test_qwen_processor.py::TestQwenProcessor::test_chat_basic -v -s

# Run only unit tests
pytest tests/unit/ -v -s

# Run only e2e tests (server must be running)
pytest tests/e2e/ -v -s

# Run by marker
pytest -m unit
pytest -m "not slow"

# Skip coverage for a quick run
pytest tests/unit/ --no-cov -v -s
```

`pytest.ini` enables `asyncio_mode = auto`, `--strict-markers`, and coverage by default.

## Code Style

### Formatting
- **Line length**: 100 characters (black + isort configured).
- **Quotes**: double quotes (black default).
- **Trailing comma**: always in multi-line collections (isort `include_trailing_comma = true`).

### Imports
Order: standard library → third-party → local (`src.*`). `isort` enforces this.

```python
# standard library
import json
from typing import Any, Dict, List, Optional

# third-party
from fastapi import WebSocket
from pydantic import BaseModel

# local
from src.core.logger import logger
from src.core.exceptions import BusinessError, ErrorCode
```

Never use relative imports (`from .foo import bar`). Always use absolute `src.*` paths.

### Type Hints
- All function signatures must have parameter and return type annotations.
- Use `Optional[X]` (not `X | None`) for Python 3.10 compatibility.
- Use `List`, `Dict`, `Tuple` from `typing` (not built-in generics) for 3.10 compat.
- `async def` for every function that performs I/O.

### Naming
| Kind | Convention | Example |
|---|---|---|
| Modules | `snake_case` | `history_manager.py` |
| Classes | `PascalCase` | `HistoryManager` |
| Functions / methods | `snake_case` | `get_history()` |
| Constants | `UPPER_SNAKE` | `_MAX_STORED = 30` |
| Private module globals | leading `_` | `_orchestrator_singleton` |
| ErrorCode values | `MODULE_ERROR_NAME` | `WS_AUTH_FAILED` |

### Docstrings
Every module, public class, and public method needs a docstring. Module docstrings go at the top and describe purpose and design principles. Use Chinese for domain/business-level comments; English is acceptable for technical annotations.

### Error Handling
- Raise `BusinessError(ErrorCode.X, "human message")` for all expected business failures.
- Use `logger.exception(...)` (not `logger.error`) inside `except` blocks that swallow exceptions, so the full traceback is captured.
- Never use bare `except:`; catch `Exception` at most.
- `_process()` in `OrchestratorAgent` **must always return `None`** — all output goes via `dispatcher`.

## Architecture Conventions

### Singleton Pattern
- `get_settings()` — configuration singleton, initialised once by `init_settings()`.
- `get_history_manager()` — Redis history singleton, initialised by `HistoryManager.init(settings)`.
- `get_orchestrator()` — agent singleton, initialised by `await init_orchestrator(settings)`.
- `ProcessorFactory.get_instance()` — processor factory singleton.
- All singletons are initialised in `lifespan()` inside `main.py`. Never initialise them at module import time.

### Logging
- Import: `from src.core.logger import logger`
- **Never use `print()`** anywhere in source or tests. Use `logger.info/debug/warning/error/exception`.
- Tests must log request parameters and responses at `INFO` level.
- WebSocket client logs every send with `[WS] >>>` and every receive with `[WS] <<<`.

### WebSocket Event Flow
Dispatcher methods (all in `EventDispatcher`): `send_tool_call` → `send_tool_result` → `send_message(is_final=True)` → `send_session_end(success=True)`.
On error: `send_error` → `send_session_end(success=False)`.
The `thinking` event **is not pushed** by `OrchestratorAgent` (removed from design).

### `FunctionCall.arguments`
Must be a **JSON string**, not a dict. Always `json.dumps(action_input)` before constructing `FunctionCall`.

### `OrchestratorAgent` singleton
Handlers must call `get_orchestrator()` — never `AgentManager.create_agent()` for the orchestrator. Context dict passed to `execute()`:
```python
{"dispatcher": EventDispatcher(websocket, session_id), "session_id": session_id}
```

## Testing Rules (from `docs/测试规范.md`)

1. **No mocks** — no `Mock`, `AsyncMock`, or `patch`. All tests use real API calls and real services.
2. **No `print`** — use `logger` exclusively.
3. **Real API keys required** — if a key is missing or is a placeholder, call `pytest.skip(...)`.
4. **Server availability** — e2e tests must catch `ServerNotAvailableError` from `WebSocketClient` and call `pytest.skip(...)`.
5. **Markers**: annotate with `@pytest.mark.unit`, `@pytest.mark.integration`, or `@pytest.mark.slow`.
6. **Async**: use `@pytest.mark.asyncio` (or rely on `asyncio_mode = auto`).

### Test File Header (required)

```python
"""
<Module> 测试

⚠️ 重要测试规则：
1. 禁止使用 Mock/AsyncMock/patch
2. 所有测试必须发起真实 API 请求
3. 使用 logger，不使用 print
"""
# ============================================================================
# 运行方式:  pytest tests/unit/test_xxx.py -v -s
# ============================================================================
# 测试方法:
#   1. test_method_name - 说明
# ============================================================================
```

## Documentation Standards

- **架构设计文档禁止出现代码**：`docs/` 下的设计文档（如 `设计文档-系统架构.md`）中**严禁包含任何代码片段**，包括但不限于 Python 代码、配置文件片段、JSON/YAML 示例、命令行示例等。只使用文字描述、Mermaid 图、流程图、表格和结构化文本来表达设计方案。
- Use Mermaid diagrams, sequence diagrams, and structured text instead of code.
- Documentation language is Chinese.

### 文档体系结构

`docs/` 目录采用**主文档 + 模块子文档**的组织方式：

| 文档 | 角色 | 说明 |
|------|------|------|
| `设计文档-系统架构.md` | **主文档（总纲）** | 系统整体架构，包含所有模块的概述和章节目录；各模块章节末尾通过链接指向对应子文档 |
| `设计文档-Skill系统架构.md` | 模块子文档 | Skill 系统详细设计（目录规范、工具链、SkillManager、迁移计划等） |
| `设计文档-RAG系统架构.md` | 模块子文档 | RAG 系统详细设计（RAGAgent、rag_query、检索流程、配置等） |
| `设计文档-配置管理与日志管理.md` | 模块子文档 | 配置管理和日志管理的实现级别规范（含代码示例，属于实现规范而非架构设计） |

**规则**：
- 新增模块设计时，创建独立子文档 `设计文档-XXX系统架构.md`，并在主文档中添加对应章节（摘要 + 链接）
- 主文档每个模块章节只放核心摘要（组件表格 + 设计要点），详细内容通过链接引向子文档
- 修改模块设计时，同步更新主文档摘要和子文档详细内容
