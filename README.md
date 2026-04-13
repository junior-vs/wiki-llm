# wiki-llm

> A living knowledge base for legacy systems — organized by features, stored as Markdown, and enriched on demand by a local LLM.

[![Python](https://img.shields.io/badge/Python-3.11-3776ab?style=flat-square)](https://www.python.org)
[![uv](https://img.shields.io/badge/uv-package%20manager-blueviolet?style=flat-square)](https://github.com/astral-sh/uv)
[![Ruff](https://img.shields.io/endpoint?url=https://raw.githubusercontent.com/astral-sh/ruff/main/assets/badge/v2.json&style=flat-square)](https://github.com/astral-sh/ruff)
[![mypy](https://img.shields.io/badge/mypy-strict-blue?style=flat-square)](https://mypy-lang.org)

Turn code study sessions into a persistent, navigable knowledge base. `wiki-llm` provides a CLI tool (`wiki`) that reads your wiki files, delegates enrichment to a local [LM Studio](https://lmstudio.ai) instance, shows you a diff, and only applies changes after your explicit confirmation.

---

## How it works

Three layers keep raw sources, structured knowledge, and LLM instructions cleanly separated:

```
┌─────────────────────────────────────────┐
│  CAPTURE — Raw Sources (immutable)      │
│  Java source code, PDFs, specs, notes   │
└────────────────────┬────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────┐
│  WIKI — Markdown files (Git repo)       │
│  features/ domains/ + control files    │
└────────────────────┬────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────┐
│  SCHEMA — WIKI.md                       │
│  Instructions for the LM Studio agent  │
└─────────────────────────────────────────┘
```

Knowledge moves through the **CODE workflow** — Capture → Organize → Distill → Express — with a Lint step that closes the feedback loop.

---

## Features

- **Human in control** — no file is ever modified without an explicit `[y/n]` prompt
- **Stateless** — no local database; the wiki files are the single source of truth
- **Offline-first** — all LLM operations run against a local LM Studio instance
- **Feature-oriented structure** — knowledge is organized around what the system *does*, not how it's coded
- **Lint engine** — finds ambiguities, coverage gaps, orphan articles, and criteria conflicts
- **Obsidian-compatible** — the wiki is a standard Markdown folder you can open directly in Obsidian

---

## Core concepts

### Feature

The primary unit of organization. Represents a capability from a **business perspective** — not a class or package, but something the system *does* (e.g. `processing-order`, `freight-calculation`).

Each feature is documented by exactly three articles:

| Article | Question answered |
|---|---|
| `business-rule.md` | What and why? |
| `data-flow.md` | How does data move? |
| `implementation.md` | Where is it in the code? |

### Domain

A business concept shared by two or more features (e.g. `order`, `customer`). Lives in `domains/` and is referenced by features via relative Markdown links.

### CODE Workflow

| Step | What happens |
|---|---|
| **Capture** | Raw sources land in `inbox/` — immutable, never modified |
| **Organize** | LM Studio generates article skeletons from a source file |
| **Distill** | LM Studio refines writing and suggests cross-links; human reviews the diff |
| **Express** | Article is marked `validated` — an Intermediate Packet ready for reference |
| **Lint** | Wiki health check: finds gaps, conflicts, stale items, and orphans |

---

## Installation

**Prerequisites:** Python 3.11, [uv](https://github.com/astral-sh/uv), and a running [LM Studio](https://lmstudio.ai) server.

```bash
git clone <repo> wiki-llm && cd wiki-llm
uv pip install -e ".[dev]"
wiki --help
```

---

## Usage

### Initialize a new wiki

```bash
wiki init ./my-wiki
```

Creates the full directory structure, templates, `WIKI.md`, `index.md`, and `log.md`, and writes a `wiki.toml` configuration file in the current directory.

### Organize a source file into article skeletons

```bash
wiki organize path/to/OrderService.java
wiki organize path/to/spec-v1.2.pdf --feature freight-calculation
```

Reads the source file, infers the feature slug, calls LM Studio to generate the three article skeletons, shows you a diff, and writes the files after confirmation.

> [!TIP]
> Use `--dry-run` on any command to preview proposed changes without writing anything.

### Distill an article or an entire feature

```bash
wiki distill features/processing-order/business-rule.md
wiki distill features/processing-order/
```

Asks LM Studio to refine the writing, suggest relative links, and advance the article status. Shows a unified diff before applying.

### Lint the wiki

```bash
wiki lint                          # entire wiki
wiki lint features/processing-order/   # scoped to one feature
```

Runs two stages:

1. **Static checks** (no LLM): orphan articles, missing articles, stale inbox items, frontmatter date mismatches
2. **Semantic checks** (LM Studio): ambiguities between features, criteria conflicts, missing domain pages

Findings are printed with severity indicators (🔴 error / 🟡 warning / 🔵 info), and critical ones are written to `inbox/` as new capture items.

---

## Wiki structure

```
wiki/
├── WIKI.md                     # schema — LM Studio instructions
├── index.md                    # full catalog with status
├── log.md                      # append-only operation history
├── inbox/                      # capture buffer
│   └── 2026-04-11-order-service-study.md
├── _templates/
│   ├── business-rule.md
│   ├── data-flow.md
│   ├── implementation.md
│   ├── domain.md
│   └── inbox-item.md
├── features/
│   └── processing-order/
│       ├── business-rule.md
│       ├── data-flow.md
│       └── implementation.md
└── domains/
    └── order.md
```

---

## Configuration

The CLI reads `wiki.toml` from the current working directory:

```toml
[wiki]
root = "./my-wiki"

[lmstudio]
base_url = "http://localhost:1234"
model = "your-model-id"
timeout = 120
```

---

## CLI project structure

```
cli/
├── main.py                     # Typer entry point + DI wiring
├── domain/                     # Zero I/O — pure models, ports, services
│   ├── model.py
│   ├── errors.py
│   ├── ports.py
│   └── services.py
├── application/                # Use cases — orchestration only
│   ├── organize.py
│   ├── distill.py
│   ├── lint.py
│   └── init_wiki.py
└── adapters/
    ├── inbound/cli/            # Typer commands
    └── outbound/
        ├── filesystem/         # File I/O adapters
        └── lmstudio/           # LM Studio HTTP client
```

The architecture follows Clean Architecture with strict import boundaries enforced by `lint-imports`:

- **Domain layer** has zero imports from adapters, `httpx`, or `yaml`
- **Application layer** depends only on domain ports and models
- **Adapters** implement domain ports — never the reverse

---

## Development

```bash
# Run all quality sensors
ruff check . && ruff format --check .
mypy wiki/ --strict
pytest --cov=wiki --cov-fail-under=80
lint-imports
```

> [!NOTE]
> All sensors must pass before any task is marked complete. The project follows HARNESS.md principles: fail loudly, functional micro / OO macro, human always in control.
