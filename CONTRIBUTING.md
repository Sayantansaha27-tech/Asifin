# Contributing to AsiFin

## Development Workflow

### Setup

Follow the [Local Development Setup](README.md#local-development-setup) section in the README.

Install pre-commit hooks after setting up your Python environment:

```bash
pre-commit install
```

The hooks run on `git commit`:
- `ruff` — lint and format
- `mypy` — type checking

And on `git push`:
- `pytest -x` — fast-fail test suite

### Branch Conventions

- `main` — stable, deployable
- `feat/<description>` — new features
- `fix/<description>` — bug fixes
- `chore/<description>` — tooling, deps, config

### Commit Style

Use imperative mood in commit messages:

```
add cross-encoder reranking to RAG pipeline
fix circuit breaker not resetting after recovery timeout
chore: bump qdrant-client to 1.8.0
```

---

## Adding a New LLM Provider

The model layer uses a strategy pattern. Adding a new provider requires three steps.

### 1. Create the backend class

Create `src/models/backends/my_provider_model.py`:

```python
from src.models.base import LLMBase
from typing import Any, Dict


class MyProviderModel(LLMBase):
    def __init__(self, api_key: str):
        self.client = MyProviderClient(api_key=api_key)

    async def generate_text(self, system_prompt: str, user_prompt: str) -> str:
        response = await self.client.chat(
            system=system_prompt,
            user=user_prompt,
        )
        return response.text

    async def analyze_image(self, image_path: str, prompt: str) -> Dict[str, Any]:
        # Implement if the provider supports vision
        # Return {"vision_analysis": "<analysis text>"}
        raise NotImplementedError("Vision not supported by this provider")

    async def health_check(self) -> bool:
        try:
            await self.client.ping()
            return True
        except Exception:
            return False
```

### 2. Register the provider in the factory

In `src/models/factory.py`, add your provider to the `reload_model` method:

```python
elif provider == "my_provider":
    from src.models.backends.my_provider_model import MyProviderModel
    if not settings.MY_PROVIDER_API_KEY:
        print("Warning: MY_PROVIDER_API_KEY missing.")
        return
    self._model = MyProviderModel(api_key=settings.MY_PROVIDER_API_KEY)
```

### 3. Add the config key

In `src/core/config.py`, add the API key setting:

```python
MY_PROVIDER_API_KEY: Optional[str] = None
```

And add it to `.env.example`:

```bash
MY_PROVIDER_API_KEY=
```

Set `ACTIVE_MODEL_PROVIDER=my_provider` to activate it.

---

## Adding a New Document Processor

Document processors live in `src/processors/`. Each processor handles a specific file type.

### 1. Create the processor

```python
# src/processors/my_format_loader.py
from src.processors.base import BaseProcessor
from typing import List, Dict, Any


class MyFormatLoader(BaseProcessor):
    def load(self, file_path: str) -> List[Dict[str, Any]]:
        # Return a list of page/chunk dicts:
        # [{"text": "...", "page_number": 1, "metadata": {...}}, ...]
        ...
```

### 2. Register in the processor manager

In `src/processors/manager.py`, add your loader to the format dispatch:

```python
from src.processors.my_format_loader import MyFormatLoader

PROCESSORS = {
    "pdf": PDFLoader,
    "table": ExcelLoader,
    "image": ImageLoader,
    "my_format": MyFormatLoader,
}
```

### 3. Update document type detection

In `src/core/rag_pipeline.py`, add your extension to `detect_document_type`:

```python
if extension in {".myext"}:
    return "my_format"
```

---

## Code Conventions

### Python

- Line length: 100 characters (ruff enforced)
- Target: Python 3.11+
- Type hints on all new functions
- Async functions for all I/O (database, LLM, vector store)
- No bare `except:` — always catch specific exceptions

### API Routes

- All routes return Pydantic response models
- Use `require_permission()` dependency for auth-gated routes
- Long-running operations use the async task pattern (return 202 + task_id)
- Health-affecting services register status via `service_registry`

### Frontend

- Components in `src/components/`
- API calls via `src/lib/api.js` (typed with generated OpenAPI types)
- No inline styles — use Tailwind classes
- Async analysis uses `AbortController` for cancellable requests

---

## Running Tests

```bash
# Full suite
./venv/bin/pytest

# Single file
./venv/bin/pytest tests/test_rag_pipeline.py -v

# Integration tests (needs Docker)
RUN_TESTCONTAINERS=1 ./venv/bin/pytest tests/integration/ -v

# Type check only
./venv/bin/mypy src/

# Lint only
./venv/bin/ruff check src/
```

---

## Pull Request Checklist

Before opening a PR:

- [ ] All existing tests pass (`pytest`)
- [ ] New functionality has tests
- [ ] Type hints added to new functions
- [ ] No secrets or API keys in code
- [ ] `.env.example` updated if new env vars added
- [ ] Pre-commit hooks pass locally
