# Legacy Wiki CLI — Technical Specification

> Command-line tool that bridges the study workflow and LM Studio,
> implementing the CODE operations defined in the Wiki Specification.
> Governed by HARNESS.md — Clean Architecture, DDD, SOLID, functional micro / OO macro.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Technical Stack](#2-technical-stack)
3. [Project Structure](#3-project-structure)
4. [Architecture](#4-architecture)
5. [Installation](#5-installation)
6. [Commands](#6-commands)
   - 6.1 [wiki organize](#61-wiki-organize)
   - 6.2 [wiki distill](#62-wiki-distill)
   - 6.3 [wiki lint](#63-wiki-lint)
   - 6.4 [wiki init](#64-wiki-init)
7. [Domain Layer](#7-domain-layer)
8. [Application Layer](#8-application-layer)
9. [Adapters Layer](#9-adapters-layer)
10. [Shared Behaviors](#10-shared-behaviors)
11. [Configuration](#11-configuration)
12. [Error Handling](#12-error-handling)
13. [Sensors and Quality Gates](#13-sensors-and-quality-gates)
14. [Decisions Log](#14-decisions-log)

---

## 1. Overview

The Wiki CLI is a Python 3.13 command-line tool that implements the CODE workflow
operations defined in `SPECIFICATION.md`. It reads files from the wiki repository,
sends them to a local LM Studio instance, shows a diff of proposed changes, and
applies them only after explicit human confirmation.

**Design principles (from HARNESS.md):**

- Human always in control — no file is modified without `[y/n]` confirmation
- Stateless — no local database; truth lives in the wiki files
- Fail loudly — typed domain errors, no silent failures
- Functional micro, OO macro — pure functions inside, classes at the boundaries
- Clean Architecture — domain has zero I/O imports; adapters implement ports
- Wiki-first — always reads `WIKI.md` and `index.md` before any LLM operation

**Non-goals:**

- Automated scheduling or watchers
- GUI or web interface
- Cloud LLM support
- Parsing or compiling Java source code

---

## 2. Technical Stack

| Concern | Choice | Rationale |
|---|---|---|
| Language | Python 3.13 | Stable, `tomllib` stdlib, `match` statements |
| Package manager | uv | Fast installs, lockfile support |
| CLI framework | Typer | Type-safe, auto-generates help |
| YAML parsing | PyYAML | Standard frontmatter parsing |
| HTTP client | httpx | Sync client, explicit timeouts |
| Terminal output | Rich | Colors, tables, spinners |
| Linter + formatter | ruff | HARNESS §5.2 Python sensor |
| Type checker | mypy --strict | HARNESS §5.2 Python sensor |
| Test runner | pytest + pytest-cov | HARNESS §4.5 — 80% minimum coverage |
| Architecture guard | lint-imports | HARNESS §5.2 — import boundary enforcement |

### `pyproject.toml`

```toml
[build-system]
requires = ["setuptools>=69", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "cli"
version = "0.1.0"
description = "Legacy Wiki CLI"
readme = "README.md"
requires-python = ">=3.13"
dependencies = [
    "typer>=0.12",
    "pyyaml>=6.0",
    "httpx>=0.27",
    "rich>=13.0",
]

[project.scripts]
wiki = "main:main"

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-cov>=5.0",
    "mypy>=1.10",
    "ruff>=0.4",
    "lint-imports>=1.12",
]

[tool.ruff]
line-length = 88
select = ["E", "F", "I", "N", "UP", "ANN"]

[tool.mypy]
strict = true
python_version = "3.13"

[tool.setuptools]
package-dir = {"" = "src"}
py-modules = ["main"]

[tool.setuptools.packages.find]
where = ["src"]
include = ["adapters*", "application*", "domain*"]
```

### `.importlinter` (architecture boundary enforcement)

```ini
[importlinter]
root_package = domain

[importlinter:contract:domain-is-pure]
name = Domain layer must not import adapters or I/O libraries
type = forbidden
source_modules = domain
forbidden_modules =
    adapters
    httpx
    yaml

[importlinter:contract:application-no-adapters]
name = Application layer must not import adapters
type = forbidden
source_modules = application
forbidden_modules =
    adapters
```

---

## 3. Project Structure

```
wiki-cli/
├── pyproject.toml
├── .python-version            # 3.13
├── .importlinter              # architecture boundary rules
├── main.py                    # bootstrap script at project root
├── wiki.toml                  # CLI configuration (user-provided)
├── README.md
└── src/
    ├── __init__.py
    ├── main.py                # Typer app module used by entry point
    ├── domain/
    │   ├── __init__.py
    │   ├── model.py           # Value Objects: WikiArticle, LintFinding, etc.
    │   ├── errors.py          # typed domain errors
    │   ├── ports.py           # Protocol interfaces (ports) for all I/O
    │   └── services.py        # pure domain functions
    ├── application/
    │   ├── __init__.py
    │   ├── organize.py        # OrganizeUseCase
    │   ├── distill.py         # DistillUseCase
    │   ├── lint.py            # LintUseCase
    │   └── init_wiki.py       # InitWikiUseCase
    └── adapters/
        ├── __init__.py
        ├── inbound/
        │   └── cli/
        │       ├── __init__.py
        │       ├── context.py     # AppContext dataclass
        │       └── commands/
        │           ├── organize.py
        │           ├── distill.py
        │           ├── lint.py
        │           └── init_wiki.py
        └── outbound/
            ├── filesystem/
            │   ├── __init__.py
            │   ├── frontmatter_repo.py
            │   ├── markdown_repo.py
            │   ├── index_repo.py
            │   └── log_repo.py
            └── lmstudio/
                ├── __init__.py
                ├── client.py
                └── prompts.py
```

---

## 4. Architecture

Following HARNESS.md §3 — Clean Architecture with DDD building blocks.

```
┌──────────────────────────────────────────────────────────┐
│  Frameworks & Drivers (Typer, httpx, PyYAML)             │
│  ┌────────────────────────────────────────────────────┐  │
│  │  Interface Adapters                                │  │
│  │  inbound/cli/   outbound/filesystem/ outbound/llm  │  │
│  │  ┌──────────────────────────────────────────────┐  │  │
│  │  │  Application Layer                           │  │  │
│  │  │  OrganizeUseCase  DistillUseCase  LintUseCase│  │  │
│  │  │  ┌────────────────────────────────────────┐  │  │  │
│  │  │  │  Domain Layer (zero I/O imports)       │  │  │  │
│  │  │  │  model.py  ports.py  services.py       │  │  │  │
│  │  │  └────────────────────────────────────────┘  │  │  │
│  │  └──────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

**Dependency rules (HARNESS §3.1):**

- Domain layer: zero imports from adapters, httpx, yaml, or any I/O library
- Application layer: depends only on domain ports and models
- Adapters: implement domain ports — never the reverse
- Use Cases: one public method, orchestration only, no business rules

---

## 5. Installation

```bash
git clone <repo> wiki-cli && cd wiki-cli
uv pip install -e ".[dev]"
wiki --help

# Run all sensors
ruff check . && ruff format --check .
mypy src/ --strict
pytest --cov=src --cov-fail-under=80
lint-imports
```

---

## 6. Commands

All commands share this execution flow:

```
1. Load wiki.toml configuration
2. Read WIKI.md (schema) from wiki root
3. Read index.md (current state) from wiki root
4. Instantiate Use Case with injected adapter implementations
5. Execute Use Case
6. Display diff / report via Rich
7. Prompt [y/n] for confirmation
8. Apply changes if confirmed
9. Append entry to log.md
10. Update index.md if needed
```

---

### 6.1 wiki organize

**Purpose:** Generate article skeletons for a feature from a raw source file.

```bash
wiki organize <source> [--feature <slug>] [--dry-run]
```

| Argument / Option | Type | Description |
|---|---|---|
| `source` | path (required) | Source file to extract knowledge from |
| `--feature` | str (optional) | Feature slug — inferred from filename if omitted |
| `--dry-run` | flag | Show proposed files without writing |

**Execution flow:**

1. Read source file as plain text
2. Infer feature slug via `domain.services.infer_feature_slug()` — confirm with user
3. Check if feature already exists in `index.md` — warn and confirm if so
4. Call `OrganizeUseCase.execute(source_content, feature_slug)`
5. Display proposed articles with diff
6. Prompt `[y/n]` — skip if `--dry-run`
7. On confirmation: write files, create inbox item, update index, append log

**LLM output contract:**

```json
{
  "feature": "processing-order",
  "articles": {
    "business-rule": "<full markdown content>",
    "data-flow": "<full markdown content>",
    "implementation": "<full markdown content>"
  },
  "open_questions": ["question 1", "question 2"],
  "suggested_domains": ["order", "payment"]
}
```

---

### 6.2 wiki distill

**Purpose:** Refine writing of an article or all articles in a feature.

```bash
wiki distill <path> [--dry-run]
```

| Argument / Option | Type | Description |
|---|---|---|
| `path` | path (required) | Article file or feature folder |
| `--dry-run` | flag | Show diffs without writing |

**Execution flow:**

1. Resolve path — single file or all articles in folder
2. For each article: call `DistillUseCase.execute(article_path)`
3. Display unified diff per article
4. Prompt `[y/n]` (single) or `[y/n/a]` (folder)
5. On confirmation: write refined content, append revision, advance status, log

**LLM output contract:**

```json
{
  "content": "<full refined markdown content>",
  "suggested_related": [
    {"label": "Data Flow", "path": "../data-flow.md"},
    {"label": "Order", "path": "../../domains/order.md"}
  ],
  "revision_description": "refined writing, added link to domain/order"
}
```

---

### 6.3 wiki lint

**Purpose:** Scan for ambiguities, gaps, orphans, stale inbox items, and criteria conflicts.

```bash
wiki lint [path] [--dry-run]
```

| Argument / Option | Type | Description |
|---|---|---|
| `path` | path (optional) | Scope to a feature. Defaults to entire wiki. |
| `--dry-run` | flag | Show findings without creating inbox items |

**Two-stage execution:**

Stage 1 — Static checks (pure functions, no LLM):

- Orphan articles: no inbound relative links
- Missing articles: features with `status: pending`
- Stale inbox items: `status: pending` older than 1 day
- Frontmatter date mismatch: `updated` differs from latest revision entry
- Long-stale revised: `status: revised`, not modified in 7+ days

Stage 2 — LLM semantic analysis:

- Ambiguities between features
- Conflicts between `business-rule` and `implementation` acceptance criteria
- Missing domain pages for referenced concepts

**Lint finding severity:**

| Severity | Symbol | Meaning |
|---|---|---|
| `error` | 🔴 | Conflict between business-rule and implementation criteria |
| `warning` | 🟡 | Ambiguity, missing domain, stale inbox item |
| `info` | 🔵 | Orphan article, coverage gap, open question |

**LLM output contract:**

```json
{
  "findings": [
    {
      "severity": "error",
      "type": "criteria-conflict",
      "description": "conflicting acceptance criteria for reserved order",
      "affected": ["features/processing-order/business-rule.md"],
      "suggested_inbox_title": "Investigate reserved order criteria conflict"
    }
  ]
}
```

Exit code 0 if no errors, 1 if any errors found.

---

### 6.4 wiki init

**Purpose:** Bootstrap a new wiki directory with full structure and templates.

```bash
wiki init <wiki-root> [--force]
```

**Execution flow:**

1. Validate `wiki-root` does not exist (or `--force` provided)
2. `InitWikiUseCase.execute(wiki_root)` creates full directory structure
3. Generates `WIKI.md`, `index.md`, `log.md`, all four article templates
4. Creates `wiki.toml` in current working directory
5. Displays Rich summary of created files

---

## 7. Domain Layer

> Zero I/O imports. Pure functions and typed models only.
> HARNESS §3.1: domain must have zero deps on frameworks, DB, or HTTP.

### `domain/model.py`

```python
from dataclasses import dataclass
from pathlib import Path
from typing import Literal

ArticleStatus = Literal["pending", "draft", "revised", "validated"]
ArticleType = Literal["business-rule", "data-flow", "implementation"]
ArticleTag = Literal["inbox", "index", "feature", "domain", "log", "schema"]
LintSeverity = Literal["error", "warning", "info"]
Operation = Literal["capture", "organize", "distill", "express", "lint"]

@dataclass(frozen=True)
class RevisionEntry:
    version: str
    date: str
    description: str

@dataclass(frozen=True)
class RelatedLink:
    label: str
    path: str  # relative markdown path

@dataclass(frozen=True)
class WikiArticle:
    path: Path
    title: str
    tag: ArticleTag
    status: ArticleStatus
    updated: str
    related: tuple[RelatedLink, ...]
    revision: tuple[RevisionEntry, ...]
    body: str

@dataclass(frozen=True)
class LintFinding:
    severity: LintSeverity
    type: str
    description: str
    affected: tuple[Path, ...]
    suggested_inbox_title: str

@dataclass(frozen=True)
class OrganizeResult:
    feature: str
    articles: dict[ArticleType, str]
    open_questions: tuple[str, ...]
    suggested_domains: tuple[str, ...]

@dataclass(frozen=True)
class DistillResult:
    content: str
    suggested_related: tuple[RelatedLink, ...]
    revision_description: str
```

### `domain/errors.py`

```python
class WikiError(Exception):
    """Base domain error."""

class ConfigNotFoundError(WikiError): ...
class WikiSchemaNotFoundError(WikiError): ...
class ArticleNotFoundError(WikiError): ...
class FeatureAlreadyExistsError(WikiError): ...
class InvalidOperationError(WikiError): ...

class LmStudioError(WikiError):
    def __init__(self, message: str, raw_response: str | None = None) -> None: ...
```

### `domain/ports.py`

Protocol interfaces for all I/O (HARNESS §3.2 — ports in domain layer):

```python
from typing import Protocol
from pathlib import Path
from .model import WikiArticle, LintFinding, OrganizeResult, DistillResult

class FrontmatterRepository(Protocol):
    def read(self, path: Path) -> WikiArticle: ...
    def write(self, article: WikiArticle) -> None: ...
    def append_revision(self, path: Path, description: str) -> None: ...

class MarkdownRepository(Protocol):
    def read_file(self, path: Path) -> str: ...
    def write_file(self, path: Path, content: str) -> None: ...
    def find_articles(self, root: Path, tag: str | None = None) -> list[Path]: ...
    def find_inbound_links(self, wiki_root: Path, target: Path) -> list[Path]: ...

class IndexRepository(Protocol):
    def add_feature(self, feature: str) -> None: ...
    def update_status(self, article_path: Path, status: str) -> None: ...
    def rebuild(self) -> None: ...

class LogRepository(Protocol):
    def append(self, operation: str, target: str, details: list[str] | None = None) -> None: ...

class LmStudioPort(Protocol):
    def organize(self, wiki_schema: str, source: str, feature: str) -> OrganizeResult: ...
    def distill(self, wiki_schema: str, index: str, article: str, related: list[str]) -> DistillResult: ...
    def lint(self, wiki_schema: str, index: str, articles: list[str]) -> list[LintFinding]: ...
```

### `domain/services.py`

Pure functions — no side effects (HARNESS §3.4 functional micro):

```python
def infer_feature_slug(filename: str) -> str:
    """Infer kebab-case feature slug from a Java filename. Pure function."""

def diff_content(original: str, proposed: str) -> list[str]:
    """Return unified diff lines. Pure function — no file I/O."""

def next_version(current_versions: list[str]) -> str:
    """Increment latest version string. Pure function."""

def is_stale_inbox_item(file_mtime: float, threshold_days: int = 1) -> bool:
    """Return True if file mtime exceeds threshold. Pure function."""

def validate_operation(operation: str) -> bool:
    """Return True if operation is a valid CODE operation name."""
```

---

## 8. Application Layer

> One Use Case per operation. Orchestration only — no business rules.
> HARNESS §3.2: Use Cases have one public method, depend only on domain ports.

### `application/organize.py`

```python
@dataclass
class OrganizeUseCase:
    """Orchestrates the Organize CODE step. All deps injected via constructor."""
    lm_studio: LmStudioPort
    markdown_repo: MarkdownRepository
    index_repo: IndexRepository
    log_repo: LogRepository
    wiki_schema: str

    def execute(self, source_content: str, feature_slug: str) -> OrganizeResult:
        """Single public method. Calls LM Studio, returns result. Does not write files."""
```

### `application/distill.py`

```python
@dataclass
class DistillUseCase:
    lm_studio: LmStudioPort
    frontmatter_repo: FrontmatterRepository
    markdown_repo: MarkdownRepository
    index_repo: IndexRepository
    log_repo: LogRepository
    wiki_schema: str
    index_content: str

    def execute(self, article_path: Path) -> DistillResult:
        """Reads article + related, calls LM Studio, returns result. Does not write files."""
```

### `application/lint.py`

```python
@dataclass
class LintUseCase:
    lm_studio: LmStudioPort
    markdown_repo: MarkdownRepository
    frontmatter_repo: FrontmatterRepository
    log_repo: LogRepository
    wiki_schema: str
    index_content: str

    def run_static_checks(self, scope: Path) -> list[LintFinding]:
        """Pure static analysis — no LLM call. Uses domain.services functions."""

    def run_semantic_checks(self, scope: Path) -> list[LintFinding]:
        """LLM semantic analysis."""

    def execute(self, scope: Path) -> list[LintFinding]:
        """Combines static + semantic findings."""
```

---

## 9. Adapters Layer

### 9.1 Inbound — CLI

`adapters/inbound/cli/context.py`:

```python
@dataclass
class AppContext:
    """Shared context injected into all CLI commands via Typer."""
    config: Config
    console: Console
    wiki_schema: str
    index_content: str
    frontmatter_repo: FrontmatterRepository
    markdown_repo: MarkdownRepository
    index_repo: IndexRepository
    log_repo: LogRepository
    lm_studio: LmStudioPort
```

CLI commands are thin adapters: translate CLI args to Use Case calls,
display results via Rich. No business logic (HARNESS §7.2).

### 9.2 Outbound — Filesystem

Implements all repository ports from `domain/ports.py`.

Rules:

- All file I/O lives here — never in domain or application layers
- `pathlib` errors caught here and translated to domain errors (HARNESS §4.3)
- Side-effecting methods named with verbs implying mutation (`write`, `append`)

### 9.3 Outbound — LM Studio

Implements `LmStudioPort` from `domain/ports.py`.

`client.py` — HTTP adapter:

```
Base URL:  http://localhost:1234/v1
Endpoint:  POST /chat/completions
Timeout:   configurable (default 30s — HARNESS §4.3 explicit timeouts)
```

`prompts.py` — pure functions building `(system_prompt, user_prompt)` tuples.
No I/O — string construction only (HARNESS §3.4 functional micro).

Shared system prompt base:

```
You are a wiki maintainer for a legacy Java system documentation project.
You follow the rules defined in WIKI.md strictly.
You never invent facts — mark unknowns as open questions.
You always respond with valid JSON matching the requested output contract.
Links are always relative Markdown — never [[wikilinks]].
Respond ONLY with JSON — no preamble, no markdown fences.
```

---

## 10. Shared Behaviors

### Diff display

```
─────────────────────────────────────────
  features/processing-order/business-rule.md
─────────────────────────────────────────
- status: draft
+ status: revised
+ ## Acceptance Criteria
+ - Given a confirmed order...
─────────────────────────────────────────
Apply changes? [y/n]:
```

### Confirmation prompts

- Single file: `Apply changes? [y/n]:`
- Multiple files: `Apply to all? [y/n/a]:` (yes / no / ask per file)
- `--dry-run`: skips all confirmation, never writes files

### Progress indicators

```
⠸ Calling LM Studio... (organize | OrderService.java)
```

### Wiki root resolution order

1. `--wiki-root` CLI option
2. `root` in `wiki.toml` in current directory
3. `root` in `~/.wiki.toml`
4. `ConfigNotFoundError` → `"wiki.toml not found. Run: wiki init <path>"`

---

## 11. Configuration

### `wiki.toml`

```toml
[wiki]
root = "C:/Users/junior/projects/my-system/wiki"

[lmstudio]
base_url = "http://localhost:1234/v1"
model = "local-model"
temperature = 0.2
max_tokens = 4096
timeout = 30
```

### `Config` dataclass (`domain/model.py`)

```python
@dataclass(frozen=True)
class Config:
    wiki_root: Path
    lmstudio_base_url: str
    lmstudio_model: str
    lmstudio_temperature: float
    lmstudio_max_tokens: int
    lmstudio_timeout: int
```

---

## 12. Error Handling

Following HARNESS §4.3 — typed errors, no silent failures, no generic exceptions
in domain logic, infrastructure errors translated at adapter boundary.

| Error Type | Trigger | Exit Code |
|---|---|---|
| `ConfigNotFoundError` | `wiki.toml` not found | 2 |
| `WikiSchemaNotFoundError` | `WIKI.md` missing | 2 |
| `ArticleNotFoundError` | Path does not exist | 4 |
| `FeatureAlreadyExistsError` | Feature already in index | warn + confirm |
| `LmStudioError` | Connection refused / timeout / bad JSON | 3 |
| `InvalidOperationError` | Bad operation name in log | 2 |
| User abort `[n]` | — | 1 |

---

## 13. Sensors and Quality Gates

Following HARNESS.md §5.2 — all must pass before any task is complete.

```bash
# Lint + format
ruff check . && ruff format --check .

# Type checking (strict)
mypy src/ --strict

# Tests + coverage (minimum 80%)
pytest --cov=src --cov-fail-under=80 --cov-report=term-missing

# Architecture boundaries
lint-imports
```

### Self-correction checklist (HARNESS §5.1)

```
ARCHITECTURE
[ ] Domain layer has zero imports from adapters, httpx, yaml, or any I/O lib
[ ] Application layer imports only domain ports and models
[ ] Adapters implement domain ports — not the reverse
[ ] Use Cases have exactly one public method
[ ] No business rules in CLI commands or adapters

SOLID / DDD
[ ] Each class has exactly one reason to change (SRP)
[ ] Ports are narrow Protocol classes (ISP)
[ ] Use Cases depend on abstractions, not concrete adapters (DIP)
[ ] Domain errors are typed — not generic exceptions

PARADIGM
[ ] Domain/services functions are pure (no side effects, no mutation)
[ ] Side effects isolated in adapter layer only

GENERAL
[ ] ruff passes with zero errors
[ ] mypy --strict passes with zero errors
[ ] pytest --cov-fail-under=80 passes
[ ] lint-imports passes
[ ] All public functions have type annotations and docstrings
[ ] No function exceeds 30 lines (HARNESS §4.1)
[ ] No file exceeds 300 lines (HARNESS §4.1)
[ ] No hardcoded secrets or credentials
[ ] No debug print statements
```

### Human escalation triggers (HARNESS §10)

Escalate to human before proceeding if:

- Task requires modifying more than 3 existing files not created in this session
- A domain model (`WikiArticle`, `LintFinding`, ports) needs to change
- A sensor fails and root cause is unclear after 2 attempts
- A new external dependency needs to be introduced
- The requirement contradicts HARNESS.md or SPECIFICATION.md

Escalation format: what you tried → what blocked you → options with
trade-offs → your recommended option.

---

## 14. Decisions Log

| # | Decision | Rationale |
|---|---|---|
| 1 | Clean Architecture package structure | HARNESS §3.1 — enforced by lint-imports |
| 2 | Domain ports as Protocol classes | HARNESS §6.2 — structural typing, no ABC overhead |
| 3 | Frozen dataclasses for Value Objects | HARNESS §6.2 — immutable by default |
| 4 | Typed domain errors | HARNESS §4.3 — no generic exceptions in domain |
| 5 | Use Cases with single public method | HARNESS §3.2 |
| 6 | Pure functions in `domain/services.py` | HARNESS §3.4 — testable without mocking |
| 7 | Constructor DI in Use Cases | HARNESS §3.4 — explicit, no service locators |
| 8 | ruff + mypy --strict + lint-imports | HARNESS §5.2 — all Python sensors mandatory |
| 9 | 80% coverage minimum | HARNESS §4.5 |
| 10 | JSON output contract with LLM | Structured, parseable |
| 11 | `response_format: json_object` | Forces valid JSON from LM Studio |
| 12 | `--dry-run` on all commands | Safe exploration without unintended writes |
| 13 | Stateless design | Wiki files are single source of truth |
| 14 | Typer for CLI | Type-safe, auto-generates help |

---

*Specification version: 2.0 — 2026-04-11 (updated with HARNESS.md)*
