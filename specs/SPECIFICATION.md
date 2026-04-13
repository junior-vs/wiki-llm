# Legacy Wiki — Project Specification

> Living knowledge base for legacy Java systems, organized by features,
> maintained in Markdown, and enriched on demand by a local LLM.

---

## Table of Contents

1. [Project Definition](#1-project-definition)
2. [Architecture](#2-architecture)
3. [Core Concepts](#3-core-concepts)
   - 3.1 [Feature](#31-feature)
   - 3.2 [Domain](#32-domain)
   - 3.3 [Feature Articles](#33-feature-articles)
   - 3.4 [Acceptance Criteria](#34-acceptance-criteria)
4. [CODE Workflow](#4-code-workflow)
5. [Repository Structure](#5-repository-structure)
6. [Control Files](#6-control-files)
   - 6.1 [WIKI.md](#61-wikimd)
   - 6.2 [index.md](#62-indexmd)
   - 6.3 [inbox/](#63-inbox)
   - 6.4 [log.md](#64-logmd)
7. [Frontmatter Conventions](#7-frontmatter-conventions)
8. [Article Templates](#8-article-templates)
   - 8.1 [business-rule.md](#81-business-rulemd)
   - 8.2 [data-flow.md](#82-data-flowmd)
   - 8.3 [implementation.md](#83-implementationmd)
   - 8.4 [domain article](#84-domain-article)
9. [CLI Script](#9-cli-script)
10. [LM Studio Integration](#10-lm-studio-integration)
11. [Decisions Log](#11-decisions-log)

---

## 1. Project Definition

A living wiki for documenting a legacy Java system, organized by features,
stored as Markdown files in a Git repository, read in Obsidian, and enriched
on demand by a local LLM via LM Studio.

**Goals:**
- Transform code study sessions into persistent, navigable knowledge
- Document business rules, data flows, and implementation details
- Enable cross-referencing between related features and domain concepts
- Detect ambiguities, gaps, and conflicts in the documented knowledge

**Non-goals:**
- Automated continuous ingestion pipelines
- Cloud-based LLM services
- Dev Services or runtime integrations

---

## 2. Architecture

Three layers, inspired by Karpathy's LLM Wiki pattern:

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

**CODE Workflow** governs how knowledge moves through the layers:

```
Capture → Organize → Distill → Express
                ↑________Lint_________|
```

---

## 3. Core Concepts

### 3.1 Feature

The primary unit of organization. Represents a capability or behavior of the
system from a **business perspective** — not a class, not a package, but
something the system **does**.

Each feature is always documented by exactly three articles:
`business-rule`, `data-flow`, and `implementation`.

**Examples:**
- ✅ `processing-order` — is a feature
- ✅ `freight-calculation` — is a feature
- ❌ `OrderService` — is a class, not a feature
- ❌ `utils` — is a package, not a feature

---

### 3.2 Domain

A business concept shared by two or more features. Lives outside any specific
feature because it belongs to several simultaneously. Analogous to a Bounded
Context in DDD — the wiki reveals where bounded contexts should be, even if
the legacy code does not respect them.

**Rule of thumb:** if a concept appears as an actor or object in more than one
feature, it is a domain.

**Examples:**
- ✅ `order` — appears in `processing-order`, `freight-calculation`,
  `order-cancellation`
- ✅ `customer` — appears in multiple features
- ❌ `freight-calculation` — is a feature, even if others reference it

---

### 3.3 Feature Articles

Each feature is always documented by exactly three articles, each answering
a different question:

| Article | Question | Location |
|---|---|---|
| `business-rule.md` | What and why? | `features/<feature>/business-rule.md` |
| `data-flow.md` | How does data move? | `features/<feature>/data-flow.md` |
| `implementation.md` | Where is it in the code? | `features/<feature>/implementation.md` |

---

### 3.4 Acceptance Criteria

Borrowed from testing discipline. In this context, criteria validate
**understanding**, not code execution.

| Article | Responsibility |
|---|---|
| `business-rule.md` | Validates understanding of the business — what the system **should** do |
| `implementation.md` | Validates code conformance — what the system **actually** does |

The Lint operation cross-checks criteria between the two articles and flags
divergences as `inbox/` capture items.

---

## 4. CODE Workflow

Adapted from Tiago Forte's *Building a Second Brain*:

### Capture
The raw sources layer — **immutable**. Nothing here is ever modified.
Includes: Java source code, existing documents (PDFs, specs, old wikis),
and study notes. The `inbox/` folder is the immediate capture buffer during
study sessions — each item is an individual file with its own frontmatter.

### Organize
LM Studio reads capture sources and generates article skeletons in the
correct feature/domain structure, with placeholders where knowledge needs
to be extracted manually.

### Distill
LM Studio refines writing, suggests relative links between articles, and
compresses knowledge to its essence. Human reviews and validates the diff.
Article `status` advances from `draft` to `revised`.

### Express
Article marked as `validated` — an Intermediate Packet ready for reference
without returning to raw sources.

### Lint
LM Studio scans the wiki for:
- Ambiguities between features (same concept documented differently)
- Domains without a dedicated page
- Orphan articles without inbound links
- Gaps in coverage (features with `pending` articles)
- Conflicts between `business-rule` and `implementation` acceptance criteria
- `inbox/` items with `status: pending` older than one study session

Lint findings become new `inbox/` capture items, closing the cycle.

---

## 5. Repository Structure

```
wiki/
├── WIKI.md                          # schema — LM Studio instructions
├── index.md                         # full catalog with status
├── log.md                           # append-only operation history
├── inbox/                           # capture buffer — one file per capture item
│   ├── 2026-04-11-order-service-study.md
│   ├── 2026-04-10-specification-v1.2.pdf
│   └── 2026-04-09-freight-calculator-notes.md
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

**Rules:**
- Raw sources live outside the wiki directory — they are never modified
- Every feature folder always has exactly three articles
- Domain articles live only in `domains/`
- Links are always relative Markdown links — never wiki-style `[[links]]`

---

## 6. Control Files

### 6.1 WIKI.md

The central schema. The first file LM Studio reads before any operation.
Defines: repository structure, frontmatter conventions, link format,
CODE workflow, and per-operation instructions (organize, distill, lint).

Co-evolved over time as the project matures.

---

### 6.2 index.md

Content-oriented catalog. Lists all articles with their current status.
Includes a coverage summary table. Updated automatically by the CLI on
every `organize` or `express` operation.

LM Studio reads `index.md` before any operation to understand the current
state of the wiki.

---

### 6.3 inbox/

A folder that serves as the immediate capture buffer during study sessions.
Each capture item is an individual file — Markdown notes, PDFs, source files,
or any other raw material. Files are never modified after capture — they are
the raw source of truth for the Organize step.

**Naming convention:** `YYYY-MM-DD-short-description.md` (or original filename
for binary sources like PDFs).

**Rule:** files with `status: pending` must never be older than one study
session. Stale items are flagged by Lint.

Each Markdown capture item uses the `inbox-item.md` template:

```yaml
---
title: [Short description of the capture]
tag: inbox
date: [date]
summary: "Order transitions from pending to processing after payment approval; reserved status triggered when stock is unavailable."
source: [origin — file path, URL, or document name]
destination: [suggested target — features/... or domains/...]
status: pending
---
```

Once processed, the file is deleted from `inbox/` and the operation is
recorded in `log.md`. Binary files (PDFs, etc.) are moved to a `raw/`
folder outside the wiki after processing.

---

### 6.4 log.md

Chronological, append-only record of all wiki operations. Never edit
previous entries — only append new ones.

Entry format:
```
## [YYYY-MM-DD] <operation> | <target>
```

Valid operations:

| Operation | Description |
|---|---|
| `capture` | Sources added to inbox |
| `organize` | Articles created or moved |
| `distill` | LM Studio refined an article |
| `express` | Article marked as validated |
| `lint` | Wiki health check performed |

---

## 7. Frontmatter Conventions

All files use YAML frontmatter. Field names are in English.

### Tag values

| `tag` | File type |
|---|---|
| `inbox` | Capture buffer |
| `index` | Wiki catalog |
| `feature` | Feature article |
| `domain` | Domain article |
| `log` | Operation log |
| `schema` | WIKI.md |

### Type values (feature articles only)

| `type` | Article |
|---|---|
| `business-rule` | Business rule article |
| `data-flow` | Data flow article |
| `implementation` | Implementation article |

### Status values

| `status` | Meaning |
|---|---|
| `pending` | Not yet created |
| `draft` | Skeleton generated, needs manual review |
| `revised` | LM Studio refined, human reviewed |
| `validated` | Intermediate Packet — ready for reference |

### Full frontmatter example

```yaml
---
title: Business Rule — Processing Order
tag: feature
summary: "Order transitions from pending to processing after payment approval; reserved status triggered when stock is unavailable."
feature: processing-order
type: business-rule
status: draft
updated: 2026-04-11
related:
  - [Data Flow](../data-flow.md)
  - [Order](../../domains/order.md)
revision:
  - version: 1.0
    date: 2026-04-11
    description: initial skeleton generated from OrderService.java
  - version: 1.1
    date: 2026-04-11
    description: added acceptance criteria after PDF v1.2 analysis
---
```

**Revision rules:**
- Always append — never edit previous entries
- Mirrors the philosophy of `log.md` at the article level
- Lint flags articles where `updated` diverges from the latest revision date

---

## 8. Article Templates

### 8.1 business-rule.md

```markdown
---
title: Business Rule — [Feature Name]
tag: feature
summary: "Order transitions from pending to processing after payment approval; reserved status triggered when stock is unavailable."
feature: [feature-slug]
type: business-rule
status: draft
updated: [date]
related: []
revision:
  - version: 1.0
    date: [date]
    description: initial skeleton generated from [source]
---

## Context
<!-- What motivated this feature? What problem does it solve? -->

## Business Rules

### Main Flow
<!-- Happy path: what happens when everything works -->

### Exceptions
<!-- What can go wrong and how the system handles it -->

### Restrictions
<!-- Business constraints: limits, validations, invariants -->

## Acceptance Criteria
<!-- Validates understanding of the business behavior -->
<!-- Format: Given / When / Then -->

## Open Questions
<!-- Doubts that still need investigation in the source code -->

## References
<!-- Links to source files, documents, tickets -->
```

---

### 8.2 data-flow.md

```markdown
---
title: Data Flow — [Feature Name]
tag: feature
summary: "Order transitions from pending to processing after payment approval; reserved status triggered when stock is unavailable."
feature: [feature-slug]
type: data-flow
status: draft
updated: [date]
related: []
revision:
  - version: 1.0
    date: [date]
    description: initial skeleton generated from [source]
---

## Input
<!-- What triggers this feature? What data enters? -->

## Processing
<!-- Transformations, validations, business logic applied -->

## Output
<!-- What is persisted, emitted, or returned? -->

## Sequence
<!-- Who calls whom? Textual description or Mermaid diagram -->

## Open Questions
<!-- Doubts that still need investigation -->

## References
<!-- Links to source files, documents, tickets -->
```

---

### 8.3 implementation.md

```markdown
---
title: Implementation — [Feature Name]
tag: feature
summary: "Order transitions from pending to processing after payment approval; reserved status triggered when stock is unavailable."
feature: [feature-slug]
type: implementation
status: draft
updated: [date]
related: []
revision:
  - version: 1.0
    date: [date]
    description: initial skeleton generated from [source]
---

## Key Classes
<!-- Main classes and their responsibilities -->

## Key Methods
<!-- Critical methods and what they do -->

## Technical Decisions
<!-- Why was it implemented this way? Known trade-offs -->

## Divergences
<!-- Where the implementation diverges from the business rule -->

## Acceptance Criteria
<!-- Validates that the implementation matches the business rule -->
<!-- Format: Given / When / Then -->

## Open Questions
<!-- Doubts that still need investigation -->

## References
<!-- Links to source files, documents, tickets -->
```

---

### 8.4 Domain Article

```markdown
---
title: [Domain Name]
tag: domain
summary: "Order transitions from pending to processing after payment approval; reserved status triggered when stock is unavailable."
status: draft
updated: [date]
related: []
revision:
  - version: 1.0
    date: [date]
    description: initial extraction from multiple features
---

## Definition
<!-- What is this concept in business terms? -->

## States
<!-- Possible states and valid transitions -->

## Invariants
<!-- Rules that are always true regardless of the feature -->
<!-- The Lint operation cross-checks these against feature acceptance criteria -->

## Used By
<!-- Features that reference this domain concept -->

## Open Questions
<!-- Doubts that still need investigation -->

## References
<!-- Source files, documents, tickets -->
```

---

## 9. CLI Script

A command-line utility that bridges the study workflow and LM Studio.
Triggered manually on demand — never runs automatically.

### Commands

```bash
# Generate article skeletons from a source file or directory
wiki organize <source>

# Refine an article or an entire feature
wiki distill <path>

# Scan the wiki for ambiguities, gaps, and conflicts
wiki lint [path]
```

### Examples

```bash
# Generate skeletons from a Java source file
wiki organize src/main/java/com/example/OrderService.java

# Distill a single article
wiki distill features/processing-order/business-rule.md

# Distill all articles in a feature
wiki distill features/processing-order/

# Lint the entire wiki
wiki lint

# Lint a single feature
wiki lint features/processing-order/
```

### Behavior (all commands)

1. Read `WIKI.md` before any operation
2. Read `index.md` to understand the current wiki state
3. Call LM Studio API at `http://localhost:1234/v1`
4. Show a diff of proposed changes
5. Wait for `[y/n]` confirmation before applying
6. Append entry to `log.md` after confirmation
7. Update `index.md` coverage table if needed

---

## 10. LM Studio Integration

LM Studio exposes a local OpenAI-compatible API:

```
Base URL: http://localhost:1234/v1
Endpoint: /chat/completions
```

### Context loaded per operation

| Operation | Files loaded into context |
|---|---|
| `organize` | `WIKI.md` + source file |
| `distill` | `WIKI.md` + `index.md` + target article |
| `lint` | `WIKI.md` + `index.md` + all articles in scope |

### Model instructions per operation

**organize:** Generate article skeletons following the templates in WIKI.md.
Fill what can be extracted from the source. Mark unknowns as placeholders.

**distill:** Refine writing. Suggest relative Markdown links in `related`.
Do not invent facts — mark gaps as `Open Questions`. Advance status to `revised`.

**lint:** Scan for ambiguities, orphans, missing domains, stale `inbox/` items,
and conflicts between `business-rule` and `implementation` acceptance criteria.
Output a structured report. Generate new `inbox/` capture items for each finding.

---

## 11. Decisions Log

| # | Decision | Rationale |
|---|---|---|
| 1 | Markdown + Git | Portable, versionable, works in Obsidian and GitHub |
| 2 | Relative Markdown links | Compatible with GitHub rendering and Obsidian |
| 3 | Organized by feature | Business-oriented, not technical |
| 4 | LM Studio (local) | Privacy, no cloud dependency, on-demand |
| 5 | Manual CLI (no automation) | Human stays in control of what gets validated |
| 6 | No Dev Services | Out of scope — wiki documents existing system |
| 7 | English field names | Consistency with technical conventions |
| 8 | Append-only log and revision | Full audit trail without overwriting history |
| 9 | Acceptance Criteria in articles | Bridges business understanding and code conformance |
| 10 | Domains separate from features | Reflects DDD Bounded Context thinking |
| 11 | inbox/ as folder, not file | Each capture item is individual — supports binary files and isolated frontmatter |

---

*Specification version: 1.1 — 2026-04-11*
