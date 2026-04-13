# Legacy Wiki CLI — Implementation Plan

> Sequenced task list for LLM-assisted implementation.
> Each task is self-contained with clear input, output, and validation.
> Governed by HARNESS.md — every task must pass all sensors before completion.

---

## How to Use This Plan

Each task follows this structure:
- **Context:** files to provide as context to the LLM
- **Prompt:** what to ask the LLM
- **Output:** files to be produced
- **Sensors:** commands to run after implementation
- **Validation:** behavioural checks beyond sensors

**Always provide as context:**
- `SPECIFICATION.md`
- `SPECIFICATION-CLI.md`
- `HARNESS.md`

Start a new LLM session per task. Do not carry unrelated context.

**Before marking any task complete, run:**
```bash
ruff check . && ruff format --check .
mypy src/ --strict
pytest --cov=src --cov-fail-under=80
lint-imports
```

**Human escalation triggers (HARNESS §10):**
Stop and ask the human if:
- Task requires modifying more than 3 existing files not created in this session
- A domain model or port interface needs to change
- A sensor fails after 2 fix attempts
- A new external dependency must be introduced

---

## Phase 0 — Project Bootstrap

---

### Task 0.1 — Initialize uv project and package skeleton

**Context:** `SPECIFICATION-CLI.md` sections 2 and 3

**Prompt:**
```
Using SPECIFICATION-CLI.md sections 2 and 3 as reference, create the
complete project skeleton for wiki-cli.

Produce:
1. pyproject.toml — with all dependencies, dev dependencies, entry point,
   ruff config, and mypy config exactly as specified in section 2
2. .python-version — content: 3.13
3. .importlinter — with the two contracts from section 2
4. src/ package with all __init__.py files matching the directory
   structure in section 3
5. src/main.py — empty Typer app stub with `app = typer.Typer()`
   and a placeholder callback

All __init__.py files must be empty but present.
Do not implement any logic yet — only structure.
```

**Output:**
```
wiki-cli/
├── pyproject.toml
├── .python-version
├── .importlinter
└── src/
    ├── __init__.py
    ├── main.py
    ├── domain/__init__.py
    ├── application/__init__.py
    └── adapters/__init__.py
        adapters/inbound/__init__.py
        adapters/inbound/cli/__init__.py
        adapters/inbound/cli/commands/__init__.py
        adapters/outbound/__init__.py
        adapters/outbound/filesystem/__init__.py
        adapters/outbound/lmstudio/__init__.py
```

**Sensors:**
```bash
uv pip install -e ".[dev]"
ruff check .
mypy src/ --strict
lint-imports
wiki --help
```

**Validation:**
- `wiki --help` shows the Typer app with no commands yet
- No mypy errors on the empty skeleton

---

### Task 0.2 — Implement Config and load_config

**Context:** `SPECIFICATION-CLI.md` sections 7 (domain/model.py), 11

**Prompt:**
```
Using SPECIFICATION-CLI.md sections 7 and 11 as reference, implement:

1. src/domain/model.py — add ONLY the Config frozen dataclass as defined
   in section 11. No other models yet.
2. src/domain/errors.py — add ONLY ConfigNotFoundError and
   WikiSchemaNotFoundError as defined in section 7.
3. src/adapters/inbound/cli/config_loader.py — implement load_config():
   - Search for wiki.toml in current directory, then ~/.wiki.toml
  - Parse using tomllib (Python 3.13 stdlib)
   - Return a Config instance
   - Raise ConfigNotFoundError with a clear message if not found
   - Raise ConfigNotFoundError if TOML is malformed
4. A sample wiki.toml at the project root
5. tests/adapters/inbound/test_config_loader.py:
   - test_loads_from_current_directory
   - test_loads_from_home_directory
   - test_raises_when_not_found
   - test_raises_on_malformed_toml

HARNESS rules to apply:
- Config must be @dataclass(frozen=True)
- ConfigNotFoundError must extend WikiError (also define WikiError in errors.py)
- load_config is an adapter — it may use tomllib and pathlib
- All functions must have type annotations
- mypy --strict must pass
```

**Output:**
```
src/
├── domain/
│   ├── model.py        # Config dataclass
│   └── errors.py       # WikiError, ConfigNotFoundError, WikiSchemaNotFoundError
└── adapters/inbound/cli/
    └── config_loader.py
tests/
└── adapters/inbound/
    └── test_config_loader.py
wiki.toml               # sample config at project root
```

**Sensors:**
```bash
ruff check . && ruff format --check .
mypy src/ --strict
pytest tests/adapters/inbound/test_config_loader.py -v
pytest --cov=src --cov-fail-under=80
lint-imports
```

---

## Phase 1 — Domain Layer

> Implement domain models, ports, and pure service functions.
> No I/O imports allowed. mypy --strict must pass on every file.

---

### Task 1.1 — Implement domain/model.py (all Value Objects)

**Context:** `SPECIFICATION-CLI.md` section 7, `HARNESS.md` §3.2 and §6.2

**Prompt:**
```
Using SPECIFICATION-CLI.md section 7 (domain/model.py) as reference,
implement all Value Objects and result types.

Add to src/domain/model.py:
- ArticleStatus, ArticleType, ArticleTag, LintSeverity, Operation (Literal types)
- RevisionEntry — frozen dataclass
- RelatedLink — frozen dataclass
- WikiArticle — frozen dataclass (use tuple, not list, for immutable collections)
- LintFinding — frozen dataclass
- OrganizeResult — frozen dataclass
- DistillResult — frozen dataclass

HARNESS rules:
- All dataclasses must be @dataclass(frozen=True)
- Use tuple[T, ...] instead of list[T] for collection fields on frozen dataclasses
- All fields must have type annotations
- mypy --strict must pass

Add tests in tests/domain/test_model.py:
- test_wiki_article_is_immutable (verify frozen=True raises on assignment)
- test_revision_entry_equality_by_value
- test_related_link_equality_by_value
```

**Output:**
```
src/domain/model.py     # all Value Objects
tests/domain/
└── test_model.py
```

**Sensors:**
```bash
ruff check . && ruff format --check .
mypy src/ --strict
pytest tests/domain/test_model.py -v
lint-imports
```

---

### Task 1.2 — Implement domain/errors.py (all typed errors)

**Context:** `SPECIFICATION-CLI.md` section 7 (domain/errors.py), `HARNESS.md` §4.3

**Prompt:**
```
Using SPECIFICATION-CLI.md section 7 (domain/errors.py) as reference,
implement all typed domain errors.

Complete src/domain/errors.py with:
- WikiError (base)
- ConfigNotFoundError
- WikiSchemaNotFoundError
- ArticleNotFoundError (include path: Path field)
- FeatureAlreadyExistsError (include feature: str field)
- InvalidOperationError (include operation: str field)
- LmStudioError (include raw_response: str | None field)

HARNESS rules:
- All errors extend WikiError
- Each error must have a meaningful __str__ that includes context fields
- No generic Exception usage in domain layer
- mypy --strict must pass

Tests in tests/domain/test_errors.py:
- test_each_error_has_meaningful_str_representation
- test_lm_studio_error_carries_raw_response
- test_all_errors_are_wiki_error_subtypes
```

**Output:**
```
src/domain/errors.py
tests/domain/
└── test_errors.py
```

**Sensors:**
```bash
ruff check . && ruff format --check .
mypy src/ --strict
pytest tests/domain/test_errors.py -v
lint-imports
```

---

### Task 1.3 — Implement domain/ports.py (Protocol interfaces)

**Context:** `SPECIFICATION-CLI.md` section 7 (domain/ports.py), `HARNESS.md` §3.2 and §6.2

**Prompt:**
```
Using SPECIFICATION-CLI.md section 7 (domain/ports.py) as reference,
implement all port Protocol interfaces.

Implement src/domain/ports.py with:
- FrontmatterRepository (Protocol)
- MarkdownRepository (Protocol)
- IndexRepository (Protocol)
- LogRepository (Protocol)
- LmStudioPort (Protocol)

HARNESS rules:
- Use typing.Protocol (not ABC) — HARNESS §6.2
- Ports must be narrow and role-specific (ISP — HARNESS §3.3)
- Zero imports from adapters, httpx, yaml, or any I/O library
- All method signatures must have full type annotations
- mypy --strict must pass
- lint-imports must pass (domain imports nothing from adapters)

Tests in tests/domain/test_ports.py:
- test_filesystem_stub_satisfies_frontmatter_repository_protocol
- test_filesystem_stub_satisfies_markdown_repository_protocol
- test_lm_studio_stub_satisfies_lm_studio_port_protocol
(Use simple in-memory stubs — no real I/O)
```

**Output:**
```
src/domain/ports.py
tests/domain/
└── test_ports.py
```

**Sensors:**
```bash
ruff check . && ruff format --check .
mypy src/ --strict
pytest tests/domain/test_ports.py -v
lint-imports
```

---

### Task 1.4 — Implement domain/services.py (pure functions)

**Context:** `SPECIFICATION-CLI.md` section 7 (domain/services.py), `HARNESS.md` §3.4

**Prompt:**
```
Using SPECIFICATION-CLI.md section 7 (domain/services.py) as reference,
implement all pure domain service functions.

Implement src/domain/services.py with:
- infer_feature_slug(filename: str) -> str
- diff_content(original: str, proposed: str) -> list[str]
- next_version(current_versions: list[str]) -> str
- is_stale_inbox_item(file_mtime: float, threshold_days: int = 1) -> bool
- validate_operation(operation: str) -> bool

HARNESS rules:
- All functions must be pure: same input → same output, no side effects
- No I/O imports (no pathlib, no os, no httpx)
- No mutable default arguments
- All functions must have type annotations and docstrings
- mypy --strict must pass

Tests in tests/domain/test_services.py covering:
- infer_feature_slug: OrderService.java → order-service
- infer_feature_slug: PaymentGatewayAdapter.java → payment-gateway-adapter
- diff_content: returns correct additions and removals
- diff_content: returns empty list for identical strings
- next_version: 1.0 → 1.1, 1.9 → 1.10
- is_stale_inbox_item: file older than threshold → True
- is_stale_inbox_item: file within threshold → False
- validate_operation: all valid operations return True
- validate_operation: invalid operation returns False
```

**Output:**
```
src/domain/services.py
tests/domain/
└── test_services.py
```

**Sensors:**
```bash
ruff check . && ruff format --check .
mypy src/ --strict
pytest tests/domain/test_services.py -v --cov=src/domain/services.py
lint-imports
```

---

## Phase 2 — Adapter: Filesystem

> Implement all repository ports against the real filesystem.
> I/O is allowed here. Translate pathlib errors to domain errors.

---

### Task 2.1 — Implement FrontmatterRepository

**Context:** `SPECIFICATION-CLI.md` sections 9.2, `HARNESS.md` §4.3

**Prompt:**
```
Implement src/adapters/outbound/filesystem/frontmatter_repo.py.

This class implements domain.ports.FrontmatterRepository using PyYAML
and pathlib for real file I/O.

Methods to implement:
- read(path: Path) -> WikiArticle
- write(article: WikiArticle) -> None
- append_revision(path: Path, description: str) -> None

Requirements:
- Frontmatter delimiter: --- (triple dash)
- Use PyYAML for parsing (sort_keys=False on write)
- read() must raise ArticleNotFoundError if path does not exist
- read() must raise WikiError with clear message if frontmatter is malformed
- append_revision() must:
  - Use domain.services.next_version() to increment version
  - Set `updated` field to today's date (ISO format)
  - Append to revision list — never overwrite previous entries
- write() must create parent directories if they do not exist
- Translate all pathlib/OSError exceptions to WikiError subtypes

HARNESS rules:
- This is an adapter — I/O is expected here
- Domain errors must be raised (not raw OSError or yaml.YAMLError)
- mypy --strict must pass

Tests in tests/adapters/outbound/filesystem/test_frontmatter_repo.py using tmp_path:
- test_read_valid_frontmatter_returns_wiki_article
- test_read_missing_file_raises_article_not_found
- test_read_malformed_frontmatter_raises_wiki_error
- test_write_and_read_roundtrip
- test_append_revision_increments_version
- test_append_revision_updates_date_to_today
- test_append_revision_preserves_previous_entries
```

**Output:**
```
src/adapters/outbound/filesystem/frontmatter_repo.py
tests/adapters/outbound/filesystem/
└── test_frontmatter_repo.py
```

**Sensors:**
```bash
ruff check . && ruff format --check .
mypy src/ --strict
pytest tests/adapters/outbound/filesystem/test_frontmatter_repo.py -v
pytest --cov=src --cov-fail-under=80
lint-imports
```

---

### Task 2.2 — Implement MarkdownRepository

**Context:** `SPECIFICATION-CLI.md` section 9.2, `HARNESS.md` §3.4

**Prompt:**
```
Implement src/adapters/outbound/filesystem/markdown_repo.py.

This class implements domain.ports.MarkdownRepository.

Methods to implement:
- read_file(path: Path) -> str
- write_file(path: Path, content: str) -> None
- find_articles(root: Path, tag: str | None = None) -> list[Path]
- find_inbound_links(wiki_root: Path, target: Path) -> list[Path]

Requirements:
- write_file() must create parent directories if they do not exist
- find_articles() must filter by frontmatter `tag` field when provided
  (reads frontmatter of each .md file — use FrontmatterRepository or PyYAML directly)
- find_inbound_links() must scan all .md files for relative path strings
  matching the target (string search — no AST)
- Translate OSError to WikiError subtypes (HARNESS §4.3)

Tests in tests/adapters/outbound/filesystem/test_markdown_repo.py using tmp_path:
- test_write_and_read_file_roundtrip
- test_write_file_creates_missing_parent_directories
- test_find_articles_returns_all_md_files_when_tag_is_none
- test_find_articles_filters_by_tag_correctly
- test_find_inbound_links_finds_files_referencing_target
- test_find_inbound_links_returns_empty_for_unreferenced_target
```

**Output:**
```
src/adapters/outbound/filesystem/markdown_repo.py
tests/adapters/outbound/filesystem/
└── test_markdown_repo.py
```

**Sensors:**
```bash
ruff check . && ruff format --check .
mypy src/ --strict
pytest tests/adapters/outbound/filesystem/test_markdown_repo.py -v
pytest --cov=src --cov-fail-under=80
lint-imports
```

---

### Task 2.3 — Implement LogRepository

**Context:** `SPECIFICATION-CLI.md` sections 6 (log.md format), 9.2

**Prompt:**
```
Implement src/adapters/outbound/filesystem/log_repo.py.

This class implements domain.ports.LogRepository.

Method to implement:
- append(operation: str, target: str, details: list[str] | None = None) -> None

Requirements:
- Append to log.md — never overwrite
- Create log.md if it does not exist with correct frontmatter (tag: log)
- Entry format: ## [YYYY-MM-DD] <operation> | <target>
- details rendered as markdown bullet list below the header
- Raise InvalidOperationError if operation is not a valid Operation literal
  (use domain.services.validate_operation())
- Translate OSError to WikiError

Tests in tests/adapters/outbound/filesystem/test_log_repo.py using tmp_path:
- test_creates_log_md_if_missing_with_correct_frontmatter
- test_appends_entry_without_overwriting_previous
- test_appends_entry_with_details_as_bullet_list
- test_two_consecutive_appends_both_appear_in_log
- test_invalid_operation_raises_invalid_operation_error
```

**Output:**
```
src/adapters/outbound/filesystem/log_repo.py
tests/adapters/outbound/filesystem/
└── test_log_repo.py
```

**Sensors:**
```bash
ruff check . && ruff format --check .
mypy src/ --strict
pytest tests/adapters/outbound/filesystem/test_log_repo.py -v
pytest --cov=src --cov-fail-under=80
lint-imports
```

---

### Task 2.4 — Implement IndexRepository

**Context:** `SPECIFICATION-CLI.md` sections 6 (index.md format), 9.2

**Prompt:**
```
Implement src/adapters/outbound/filesystem/index_repo.py.

This class implements domain.ports.IndexRepository.

Methods to implement:
- add_feature(feature: str) -> None
- update_status(article_path: Path, status: str) -> None
- rebuild() -> None

Constructor receives wiki_root: Path.

Requirements:
- add_feature(): append feature section with all three articles as pending
- update_status(): update status inline and recalculate coverage table
- rebuild(): scan all wiki files using MarkdownRepository.find_articles()
  and regenerate index.md from scratch preserving frontmatter
- index.md must always have correct frontmatter (tag: index)
- Coverage table format matches SPECIFICATION.md Concept 12

Tests in tests/adapters/outbound/filesystem/test_index_repo.py using tmp_path:
- test_add_feature_creates_section_with_pending_status
- test_update_status_changes_status_and_recalculates_coverage
- test_rebuild_generates_index_from_actual_wiki_files
- test_index_frontmatter_tag_is_always_index
```

**Output:**
```
src/adapters/outbound/filesystem/index_repo.py
tests/adapters/outbound/filesystem/
└── test_index_repo.py
```

**Sensors:**
```bash
ruff check . && ruff format --check .
mypy src/ --strict
pytest tests/adapters/outbound/filesystem/test_index_repo.py -v
pytest --cov=src --cov-fail-under=80
lint-imports
```

---

## Phase 3 — Adapter: LM Studio

---

### Task 3.1 — Implement prompts.py (pure functions)

**Context:** `SPECIFICATION-CLI.md` sections 6 (LLM output contracts), 9.3

**Prompt:**
```
Implement src/adapters/outbound/lmstudio/prompts.py.

Pure functions that build (system_prompt, user_prompt) tuples.
No I/O — string construction only (HARNESS §3.4 functional micro).

Functions to implement:
- organize(wiki_schema: str, source_content: str, feature_slug: str) -> tuple[str, str]
- distill(wiki_schema: str, index_content: str, article_content: str, related_contents: list[str]) -> tuple[str, str]
- lint(wiki_schema: str, index_content: str, articles_content: list[str]) -> tuple[str, str]

Requirements:
- All functions return (system_prompt, user_prompt)
- System prompt must include the shared base from SPECIFICATION-CLI.md §9.3
- System prompt must embed wiki_schema content
- User prompt must include the full JSON output contract from sections 6.1, 6.2, 6.3
- Instruct LLM: never invent facts, respond ONLY with JSON, no markdown fences
- All functions are pure — mypy --strict must pass, lint-imports must pass

Tests in tests/adapters/outbound/lmstudio/test_prompts.py:
- test_organize_prompt_contains_feature_slug
- test_organize_prompt_contains_source_content
- test_organize_prompt_contains_json_output_contract
- test_distill_prompt_contains_article_content
- test_distill_prompt_contains_all_related_content
- test_lint_prompt_contains_all_articles
- test_all_prompts_embed_wiki_schema
- test_all_system_prompts_contain_no_wikilinks_instruction
```

**Output:**
```
src/adapters/outbound/lmstudio/prompts.py
tests/adapters/outbound/lmstudio/
└── test_prompts.py
```

**Sensors:**
```bash
ruff check . && ruff format --check .
mypy src/ --strict
pytest tests/adapters/outbound/lmstudio/test_prompts.py -v
lint-imports
```

---

### Task 3.2 — Implement LM Studio client

**Context:** `SPECIFICATION-CLI.md` sections 9.3, 12

**Prompt:**
```
Implement src/adapters/outbound/lmstudio/client.py.

This class implements domain.ports.LmStudioPort using httpx for HTTP calls.

Methods:
- organize(wiki_schema, source, feature) -> OrganizeResult
- distill(wiki_schema, index, article, related) -> DistillResult
- lint(wiki_schema, index, articles) -> list[LintFinding]

Internal helper (not public):
- _call(system_prompt, user_prompt) -> dict — sends POST to LM Studio

Requirements:
- Use httpx.Client (sync) with explicit timeout from Config
- Request format exactly as defined in SPECIFICATION-CLI.md §9.3
- response_format: json_object on all requests
- Each public method calls prompts.organize/distill/lint, then _call,
  then maps response dict to domain Value Objects
- Error translation (HARNESS §4.3):
  - ConnectError → LmStudioError("LM Studio is not running at <url>")
  - TimeoutException → LmStudioError("Request timed out after <n>s")
  - JSON decode error → LmStudioError("Response is not valid JSON", raw_response=...)
  - HTTP != 200 → LmStudioError("LM Studio returned HTTP <code>")
- mypy --strict must pass

Tests in tests/adapters/outbound/lmstudio/test_client.py using httpx mock (respx or pytest-httpx):
- test_organize_sends_correct_request_format
- test_organize_parses_response_into_organize_result
- test_distill_parses_response_into_distill_result
- test_lint_parses_response_into_lint_findings
- test_connect_error_raises_lm_studio_error
- test_timeout_raises_lm_studio_error
- test_invalid_json_raises_lm_studio_error_with_raw_response
- test_http_error_status_raises_lm_studio_error
```

**Output:**
```
src/adapters/outbound/lmstudio/client.py
tests/adapters/outbound/lmstudio/
└── test_client.py
```

**Sensors:**
```bash
ruff check . && ruff format --check .
mypy src/ --strict
pytest tests/adapters/outbound/lmstudio/test_client.py -v
pytest --cov=src --cov-fail-under=80
lint-imports
```

---

## Phase 4 — Application Layer

---

### Task 4.1 — Implement OrganizeUseCase

**Context:** `SPECIFICATION-CLI.md` sections 6.1, 8

**Prompt:**
```
Implement src/application/organize.py — OrganizeUseCase.

Following SPECIFICATION-CLI.md section 8 (Application Layer) strictly.

Requirements:
- @dataclass with all dependencies injected via constructor
- Single public method: execute(source_content: str, feature_slug: str) -> OrganizeResult
- Orchestration only — no business rules, no file I/O
- Calls lm_studio.organize() and returns OrganizeResult
- Does NOT write files — the CLI adapter decides what to apply
- mypy --strict must pass
- lint-imports must pass (no adapter imports)

Tests in tests/application/test_organize.py using in-memory stubs:
- test_execute_calls_lm_studio_with_correct_arguments
- test_execute_returns_organize_result_from_lm_studio
- test_execute_raises_lm_studio_error_on_failure
(Use Protocol-compliant stubs — no real I/O, no mocking frameworks needed)
```

**Output:**
```
src/application/organize.py
tests/application/
└── test_organize.py
```

**Sensors:**
```bash
ruff check . && ruff format --check .
mypy src/ --strict
pytest tests/application/test_organize.py -v
lint-imports
```

---

### Task 4.2 — Implement DistillUseCase

**Context:** `SPECIFICATION-CLI.md` sections 6.2, 8

**Prompt:**
```
Implement src/application/distill.py — DistillUseCase.

Requirements:
- @dataclass with all dependencies injected via constructor
- Single public method: execute(article_path: Path) -> DistillResult
- Orchestration:
  1. Read article via frontmatter_repo.read()
  2. Resolve and read all related files via markdown_repo.read_file()
  3. Call lm_studio.distill() with article content + related contents
  4. Return DistillResult — do NOT write files
- mypy --strict must pass
- lint-imports must pass

Tests in tests/application/test_distill.py using in-memory stubs:
- test_execute_reads_article_and_related_files
- test_execute_passes_all_related_content_to_lm_studio
- test_execute_returns_distill_result_from_lm_studio
- test_execute_raises_article_not_found_for_missing_path
```

**Output:**
```
src/application/distill.py
tests/application/
└── test_distill.py
```

**Sensors:**
```bash
ruff check . && ruff format --check .
mypy src/ --strict
pytest tests/application/test_distill.py -v
lint-imports
```

---

### Task 4.3 — Implement LintUseCase

**Context:** `SPECIFICATION-CLI.md` sections 6.3, 8

**Prompt:**
```
Implement src/application/lint.py — LintUseCase.

Requirements:
- @dataclass with all dependencies injected via constructor
- Three public methods:
  - run_static_checks(scope: Path) -> list[LintFinding]
  - run_semantic_checks(scope: Path) -> list[LintFinding]
  - execute(scope: Path) -> list[LintFinding]
- run_static_checks must use ONLY domain.services pure functions — no LLM
  Static checks from SPECIFICATION-CLI.md §6.3:
  - Orphan articles (no inbound links)
  - Missing articles (status: pending)
  - Stale inbox items (mtime > 1 day — use services.is_stale_inbox_item())
  - Frontmatter date mismatch
  - Long-stale revised articles (7+ days)
- run_semantic_checks calls lm_studio.lint()
- execute() = run_static_checks() + run_semantic_checks(), merged
- mypy --strict must pass
- lint-imports must pass

Tests in tests/application/test_lint.py:
- test_static_checks_detect_orphan_articles
- test_static_checks_detect_stale_inbox_items
- test_static_checks_detect_pending_articles
- test_semantic_checks_delegate_to_lm_studio
- test_execute_merges_static_and_semantic_findings
```

**Output:**
```
src/application/lint.py
tests/application/
└── test_lint.py
```

**Sensors:**
```bash
ruff check . && ruff format --check .
mypy src/ --strict
pytest tests/application/test_lint.py -v
lint-imports
```

---

## Phase 5 — CLI Commands

---

### Task 5.1 — Implement main.py and AppContext

**Context:** `SPECIFICATION-CLI.md` sections 9.1, 10

**Prompt:**
```
Implement src/main.py as the Typer entry point and
src/adapters/inbound/cli/context.py as AppContext.

Requirements for main.py:
- Typer app named "wiki" with descriptive help
- Register four subcommands: organize, distill, lint, init
  (import from adapters/inbound/cli/commands/ — stubs returning "Not implemented" are fine)
- Shared callback that:
  - Loads config via load_config()
  - Reads WIKI.md from wiki_root — raises WikiSchemaNotFoundError if missing
  - Reads index.md from wiki_root
  - Instantiates all concrete adapter implementations
  - Stores AppContext in Typer context object
- --wiki-root global option overrides config root
- --version flag prints version from pyproject.toml
- All WikiError subtypes caught at this level → Rich error panel + correct exit code

Requirements for context.py:
- AppContext frozen dataclass with all fields from SPECIFICATION-CLI.md §9.1
- All port fields typed as Protocol interfaces (not concrete implementations)

HARNESS rules:
- main.py is the DI wiring point — the ONLY place where concrete adapters
  are instantiated and injected
- mypy --strict must pass
```

**Output:**
```
src/main.py
src/adapters/inbound/cli/context.py
```

**Sensors:**
```bash
ruff check . && ruff format --check .
mypy src/ --strict
lint-imports
wiki --help
wiki --version
```

---

### Task 5.2 — Implement organize command

**Context:** `SPECIFICATION-CLI.md` sections 6.1, 9.1

**Prompt:**
```
Implement src/adapters/inbound/cli/commands/organize.py.

This is a thin CLI adapter — it translates CLI args to OrganizeUseCase calls
and displays results via Rich. No business logic (HARNESS §7.2).

Requirements:
- Accept: source (Path), --feature (str, optional), --dry-run (flag)
- Infer feature slug using domain.services.infer_feature_slug() if --feature not provided
  Show inferred name and prompt user confirmation before proceeding
- Check if feature already exists in index — Rich warning + confirm if so
- Instantiate OrganizeUseCase from AppContext (dependency already injected)
- Call use_case.execute(source_content, feature_slug)
- Display diff for each proposed article using domain.services.diff_content()
- Rich spinner during LM Studio call
- Prompt [y/n] — skip if --dry-run
- On confirmation:
  - Write three article files via markdown_repo.write_file()
  - Create inbox/ item via markdown_repo.write_file()
  - Call index_repo.add_feature()
  - Call log_repo.append("organize", feature, details)
- LmStudioError → Rich error panel + exit code 3
- OSError / WikiError → Rich error panel + exit code 4

Tests in tests/adapters/inbound/cli/test_organize_command.py using Typer test client
and in-memory stubs:
- test_organize_dry_run_does_not_write_files
- test_organize_confirms_inferred_feature_slug
- test_organize_writes_three_articles_on_confirmation
- test_organize_creates_inbox_item_on_confirmation
- test_organize_shows_error_on_lm_studio_failure
```

**Output:**
```
src/adapters/inbound/cli/commands/organize.py
tests/adapters/inbound/cli/
└── test_organize_command.py
```

**Sensors:**
```bash
ruff check . && ruff format --check .
mypy src/ --strict
pytest tests/adapters/inbound/cli/test_organize_command.py -v
pytest --cov=src --cov-fail-under=80
lint-imports
```

---

### Task 5.3 — Implement distill command

**Context:** `SPECIFICATION-CLI.md` sections 6.2, 9.1

**Prompt:**
```
Implement src/adapters/inbound/cli/commands/distill.py.

Thin CLI adapter for DistillUseCase. No business logic.

Requirements:
- Accept: path (Path), --dry-run (flag)
- Resolve path — single file or folder (collect all feature articles)
- For each article:
  - Call use_case.execute(article_path) via DistillUseCase
  - Display unified diff of original vs proposed (use domain.services.diff_content())
  - Rich spinner per LM Studio call
- Confirmation:
  - Single file: [y/n]
  - Multiple files: [y/n/a] (yes / no / ask per file)
  - --dry-run: display only
- On confirmation per article:
  - Write refined content via markdown_repo.write_file()
  - Call frontmatter_repo.append_revision() with revision_description
  - Advance status draft → revised via index_repo.update_status()
    (do not change revised or validated)
  - Call log_repo.append("distill", article_path, details)
- Rich progress bar when processing multiple articles

Tests in tests/adapters/inbound/cli/test_distill_command.py:
- test_distill_single_file_dry_run_does_not_write
- test_distill_folder_processes_all_articles
- test_distill_advances_status_from_draft_to_revised
- test_distill_does_not_change_validated_status
- test_distill_appends_revision_on_confirmation
```

**Output:**
```
src/adapters/inbound/cli/commands/distill.py
tests/adapters/inbound/cli/
└── test_distill_command.py
```

**Sensors:**
```bash
ruff check . && ruff format --check .
mypy src/ --strict
pytest tests/adapters/inbound/cli/test_distill_command.py -v
pytest --cov=src --cov-fail-under=80
lint-imports
```

---

### Task 5.4 — Implement lint command

**Context:** `SPECIFICATION-CLI.md` sections 6.3, 9.1

**Prompt:**
```
Implement src/adapters/inbound/cli/commands/lint.py.

Thin CLI adapter for LintUseCase. No business logic.

Requirements:
- Accept: path (Path, optional — default wiki root), --dry-run (flag)
- Call use_case.execute(scope) via LintUseCase
- Display findings in Rich table: Severity | Type | Description | Affected
- Summary line: "X errors, Y warnings, Z info"
- Prompt [y/n] to create inbox items for each finding
- --dry-run: display only, no inbox items created
- On confirmation: write one inbox/ file per finding via markdown_repo.write_file()
- Call log_repo.append("lint", target, summary)
- Exit code 0 if no errors, exit code 1 if any errors found

Tests in tests/adapters/inbound/cli/test_lint_command.py:
- test_lint_dry_run_does_not_create_inbox_items
- test_lint_displays_findings_table
- test_lint_creates_inbox_item_per_finding_on_confirmation
- test_lint_exits_0_when_no_errors
- test_lint_exits_1_when_errors_found
```

**Output:**
```
src/adapters/inbound/cli/commands/lint.py
tests/adapters/inbound/cli/
└── test_lint_command.py
```

**Sensors:**
```bash
ruff check . && ruff format --check .
mypy src/ --strict
pytest tests/adapters/inbound/cli/test_lint_command.py -v
pytest --cov=src --cov-fail-under=80
lint-imports
```

---

### Task 5.5 — Implement init command

**Context:** `SPECIFICATION-CLI.md` section 6.4, `SPECIFICATION.md` section 5

**Prompt:**
```
Implement src/application/init_wiki.py (InitWikiUseCase) and
src/adapters/inbound/cli/commands/init_wiki.py (CLI adapter).

InitWikiUseCase.execute(wiki_root: Path) -> None:
- Creates full directory structure from SPECIFICATION.md section 5
- Generates WIKI.md, index.md, log.md with correct frontmatter
- Copies all four article templates from SPECIFICATION.md section 8
- Raises WikiError if wiki_root already exists (unless force=True)

CLI adapter:
- Accept: wiki-root (Path), --force (flag)
- Calls InitWikiUseCase.execute()
- Creates wiki.toml in current working directory pointing to wiki-root
- Displays Rich summary table of all created files
- --force: prompts confirmation before overwriting

Tests:
- test_init_creates_full_directory_structure
- test_init_fails_if_root_exists_without_force
- test_init_overwrites_with_force_flag
- test_init_creates_wiki_toml_in_cwd
```

**Output:**
```
src/application/init_wiki.py
src/adapters/inbound/cli/commands/init_wiki.py
tests/application/test_init_wiki.py
tests/adapters/inbound/cli/test_init_command.py
```

**Sensors:**
```bash
ruff check . && ruff format --check .
mypy src/ --strict
pytest tests/application/test_init_wiki.py tests/adapters/inbound/cli/test_init_command.py -v
pytest --cov=src --cov-fail-under=80
lint-imports
```

---

## Phase 6 — Integration and Polish

---

### Task 6.1 — End-to-end integration tests

**Context:** All previous tasks completed

**Prompt:**
```
Create tests/test_integration.py with end-to-end tests
using Typer test client, tmp_path, and in-memory stubs for LM Studio.

Test scenarios:
1. test_full_init_flow: wiki init creates correct structure
2. test_full_organize_flow: wiki organize → 3 articles + inbox item + log entry
3. test_full_distill_flow: wiki distill → refined content + revision + log entry
4. test_full_lint_static_flow: wiki lint detects orphan + stale inbox without LLM
5. test_full_lint_semantic_flow: wiki lint with mocked LM Studio creates inbox items
6. test_index_updated_after_organize: index.md reflects new feature after organize
7. test_status_advanced_after_distill: status advances draft → revised after distill

All tests use:
- tmp_path fixture for isolated wiki directory
- Protocol-compliant in-memory stubs for LM Studio (no real HTTP)
- Real filesystem adapters against tmp_path (no mocking of file I/O)

All tests must pass without LM Studio running.
Coverage must remain above 80%.
```

**Output:**
```
tests/
└── test_integration.py
```

**Sensors:**
```bash
ruff check . && ruff format --check .
mypy src/ --strict
pytest tests/test_integration.py -v
pytest --cov=src --cov-fail-under=80
lint-imports
```

---

### Task 6.2 — README.md

**Context:** All commands implemented

**Prompt:**
```
Create README.md for wiki-cli covering:

1. What it is (one paragraph — link to SPECIFICATION.md)
2. Prerequisites: Python 3.13, uv, LM Studio with a loaded model
3. Installation:
   git clone <repo> && cd wiki-cli
   uv pip install -e ".[dev]"
4. Configuration: wiki.toml setup with Windows path example
5. Quick start (four commands):
   wiki init ./wiki
   wiki organize src/.../OrderService.java
   wiki distill features/processing-order/
   wiki lint
6. Full command reference: all commands with options and examples
7. CODE workflow diagram as ASCII art
8. Running sensors (HARNESS §5.2):
   ruff check . && ruff format --check .
   mypy src/ --strict
   pytest --cov=src --cov-fail-under=80
   lint-imports
9. Troubleshooting: one section per error category from SPECIFICATION-CLI.md §12

Keep under 200 lines. Use Windows path examples.
```

**Output:**
```
wiki-cli/
└── README.md
```

---

## Implementation Order Summary

```
Phase 0 — Bootstrap
  0.1 → uv project + package skeleton
  0.2 → Config + load_config

Phase 1 — Domain (no I/O, pure)
  1.1 → domain/model.py (all Value Objects)
  1.2 → domain/errors.py (all typed errors)
  1.3 → domain/ports.py (Protocol interfaces)
  1.4 → domain/services.py (pure functions)

Phase 2 — Filesystem Adapters
  2.1 → FrontmatterRepository
  2.2 → MarkdownRepository
  2.3 → LogRepository
  2.4 → IndexRepository

Phase 3 — LM Studio Adapter
  3.1 → prompts.py (pure functions)
  3.2 → client.py (HTTP adapter)

Phase 4 — Application Layer (Use Cases)
  4.1 → OrganizeUseCase
  4.2 → DistillUseCase
  4.3 → LintUseCase

Phase 5 — CLI Commands
  5.1 → main.py + AppContext (DI wiring)
  5.2 → organize command
  5.3 → distill command
  5.4 → lint command
  5.5 → init command

Phase 6 — Integration
  6.1 → end-to-end tests
  6.2 → README.md
```

## Dependency Graph

```
domain/model.py ◄── (no deps — innermost)
domain/errors.py ◄── model.py
domain/ports.py ◄── model.py, errors.py
domain/services.py ◄── model.py

adapters/outbound/filesystem/* ◄── domain/ports.py, model.py, errors.py
adapters/outbound/lmstudio/prompts.py ◄── (pure — no deps)
adapters/outbound/lmstudio/client.py ◄── domain/ports.py, model.py, errors.py, prompts.py

application/organize.py ◄── domain/ports.py, model.py
application/distill.py ◄── domain/ports.py, model.py
application/lint.py ◄── domain/ports.py, model.py, services.py
application/init_wiki.py ◄── domain/model.py, errors.py

adapters/inbound/cli/context.py ◄── domain/ports.py, model.py
adapters/inbound/cli/commands/* ◄── application/*, domain/services.py, context.py

main.py ◄── all commands + all concrete adapters (DI wiring point)
```

---

*Plan version: 2.0 — 2026-04-11 (updated with HARNESS.md)*
