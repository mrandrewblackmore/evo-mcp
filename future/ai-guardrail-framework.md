# AI Guardrail Framework for `evo-mcp`

> Generated: March 5, 2026
> Purpose: Define guardrails to preserve design, architecture, code style, quality, and security
> when AI agents update the repository (e.g., adapting to Evo API changes).

---

## Problem Statement

The `evo-mcp` repository was written by humans and follows specific design patterns,
architecture boundaries, and coding conventions. As the Evo API evolves, we want to
use AI agents to update this codebase — but we must ensure the AI preserves the
original quality, style, and architectural integrity.

---

## Current State Audit

### What the repo already has:
- **CODEOWNERS** — global fallback to `@SeequentEvo/evo-mcp-maintainers`
- **`.github/copilot-instructions.md`** — basic project overview for AI
- **PR template, issue templates, CONTRIBUTING.md** — governance docs
- **SPDX license headers** on all source files
- **`reuse` tool** in dev dependencies for license compliance

### What's completely absent:
- No CI/CD pipeline (no `.github/workflows/`)
- No linter or formatter (no ruff, flake8, black, mypy)
- No pre-commit hooks
- No type checking
- No tests
- No security scanning
- No architecture documentation beyond the basic copilot-instructions.md
- No automated quality gates of any kind

**Conclusion:** Right now there are **zero automated guardrails** — an AI (or a human)
can push anything and nothing enforces consistency.

---

## Guardrail Framework: 7 Layers

### Layer 1 — AI Context Files (Tell the AI what the rules are)

The existing `.github/copilot-instructions.md` is too thin. It describes *what* the
project is but not *how* to write code for it. An AI agent needs explicit guidance.

| File | Purpose | Status |
|---|---|---|
| `.github/copilot-instructions.md` | Expanded architecture guide, patterns, conventions, do/don't rules | **Exists but needs major expansion** |
| `ARCHITECTURE.md` | Module boundaries, dependency flow, class responsibilities, design decisions | **Missing** |
| `CLAUDE.md` / `AGENTS.md` | Agent-specific instructions (applies to Claude Code, Codex, Copilot Workspace) | **Missing** |

**What to capture in these files:**
- The tool registration pattern (`register_*_tools(mcp)` with inner `async def` closures)
- EvoContext singleton pattern and why it exists
- The builder pattern (`BaseObjectBuilder` → `subclass.build()`)
- SDK client lifecycle (transport → authorizer → connector → client)
- Error handling conventions (return error dicts vs. raise exceptions)
- Naming conventions (snake_case functions, `_private` helpers, docstring style)
- Security rules (no credentials in code, env vars only, token caching approach)

---

### Layer 2 — Code Style Enforcement (Catch style drift automatically)

| Tool | What It Enforces | Config File |
|---|---|---|
| **Ruff** (linter + formatter) | PEP 8, import ordering, unused imports, code smells, consistent formatting | `ruff.toml` |
| **mypy** (strict mode) | Type safety — catches API contract breaks when SDK types change | `mypy.ini` or `pyproject.toml [tool.mypy]` |
| **pre-commit** | Runs ruff + mypy + reuse-lint locally before every commit | `.pre-commit-config.yaml` |

This is the **most impactful missing guardrail**. When an AI generates code, ruff and mypy catch:
- Wrong import style
- Inconsistent string quoting
- Missing type annotations
- API signature mismatches after SDK updates
- Unused variables/imports the AI left behind

---

### Layer 3 — Test Suite (Catch behavioral regressions)

Refers to the companion document: [`docs/unit-test-plan.md`](unit-test-plan.md)

Key additions beyond basic unit tests:

| Mechanism | Purpose |
|---|---|
| **Unit tests** (~155 cases from the test plan) | Verify each function's behavior in isolation |
| **Coverage threshold** (e.g., 80%) | Prevent AI from adding untested code |
| **Snapshot/contract tests** | Lock down the shape of objects sent to Evo API — if the API changes, these break first |
| **Schema validation tests** | Verify builder output matches `evo-schemas` types — catches schema version drift |

The schema validation tests are particularly important: when the Evo API changes,
the AI updates the builders, and these tests confirm the output still matches the
expected schema.

---

### Layer 4 — CI/CD Pipeline (Enforce gates on every PR)

**This is the single biggest gap.** Without `.github/workflows/`, nothing is enforced.

Required pipeline (every step must pass before merge):

```
PR opened/updated
  ├── ruff check + ruff format --check     (style)
  ├── mypy --strict                        (types)
  ├── pytest --cov --cov-fail-under=80     (tests + coverage)
  ├── reuse lint                           (license compliance)
  ├── pip-audit / safety check             (dependency security)
  └── CODEOWNERS → require maintainer approval
```

---

### Layer 5 — Architecture Boundary Enforcement

Prevent AI from crossing module boundaries or introducing new patterns:

| Mechanism | What It Prevents |
|---|---|
| **Import linter rules** (ruff `I` + `INP` rules) | Tools importing from other tools directly; circular dependencies |
| **Forbidden patterns in CI** (grep-based checks) | Direct `requests`/`urllib` usage (must use `AioTransport`); hardcoded URLs; `print()` instead of `logger` |
| **Module structure validation** | New tools must go in `src/evo_mcp/tools/` and follow `register_*_tools()` pattern |
| **README architecture validation** | Verify that the architecture described in `README.md` still matches the actual codebase (see below) |
| **ADRs** (Architecture Decision Records) in `docs/adr/` | Document *why* design choices were made so AI doesn't undo them |

#### README Architecture Validation

The `README.md` contains a Mermaid architecture diagram and a key components table
that describe the system's structure. These must stay in sync with the code. An AI
that adds a new tool category, changes the transport mechanism, or restructures
`EvoContext` could silently make the README inaccurate.

**What the README currently documents (as of March 2026):**

1. **Mermaid diagram** — Shows: MCP Clients → (stdio / streamable HTTP) → Evo MCP Server → (HTTPS) → Evo APIs
2. **Server subgraph** — Three components: Tool Modules (General · Admin · Data · Filesystem), MCP_TOOL_FILTER, EvoContext (OAuth · Tokens)
3. **API subgraph** — Three services: Discovery, Workspace, Object
4. **Key components table** — 5 rows: MCP clients, FastMCP server, Tool modules, EvoContext, Evo APIs

**Automated checks to add in CI:**

```bash
# 1. Verify every tool category mentioned in the README diagram actually exists
#    as a register_*_tools() function in the codebase
for category in general admin data filesystem; do
  grep -q "register_${category}_tools" src/evo_mcp/tools/*.py || \
    echo "WARN: README mentions '${category}' tools but no register_${category}_tools() found"
done

# 2. Verify every Evo API mentioned in the README is actually imported/used
for api in Discovery Workspace Object; do
  grep -rq "${api}APIClient" src/evo_mcp/ || \
    echo "WARN: README mentions ${api} API but ${api}APIClient not found in code"
done

# 3. Verify transport modes mentioned in README match code
grep -q "stdio" src/mcp_tools.py || echo "WARN: README mentions stdio but not found in mcp_tools.py"
grep -q "http" src/mcp_tools.py || echo "WARN: README mentions HTTP but not found in mcp_tools.py"

# 4. Verify EvoContext still exists and handles what the README says it does
grep -q "class EvoContext" src/evo_mcp/context.py || echo "WARN: EvoContext class not found"
grep -q "OAuth\|oauth\|authorizer\|access_token" src/evo_mcp/context.py || \
  echo "WARN: README says EvoContext handles OAuth but no OAuth code found in context.py"
```

**When to trigger this check:**
- On every PR that modifies files in `src/` (code changed → does README still match?)
- On every PR that modifies `README.md` (README changed → does it still match code?)

**What to do when it fails:**
- If code changed but README wasn't updated → PR must also update the README diagram/table
- If README changed but doesn't match code → reject the PR

Example forbidden-pattern checks for CI:

```bash
# Fail if anyone (including AI) uses print() instead of logger
grep -rn "^\s*print(" src/evo_mcp/ && exit 1 || true

# Fail if raw HTTP calls bypass the SDK transport
grep -rn "import requests\|import urllib\|import httpx" src/ && exit 1 || true
```

---

### Layer 6 — Security Guardrails

| Mechanism | What It Catches |
|---|---|
| **pip-audit** in CI | Known vulnerabilities in dependencies |
| **gitleaks** or **trufflehog** pre-commit hook | Secrets/tokens accidentally committed |
| **Dependency pinning** (`uv.lock`) | Prevents supply chain attacks from unpinned transitive deps (already in place) |
| **SPDX/reuse lint** (already in dev deps) | Missing license headers on new files |
| **No `eval()`, `exec()`, `subprocess` scan** | AI-generated code sometimes introduces unsafe patterns |

---

### Layer 7 — Human Review Gate

Even with all automated checks passing, AI-generated changes need a human review step:

| Mechanism | Purpose |
|---|---|
| **CODEOWNERS** (already exists) | Require maintainer approval for all PRs |
| **Branch protection rules** | Require CI pass + 1 approval; no direct push to `main` |
| **PR size limits** | Flag PRs over ~500 lines for extra scrutiny |
| **AI-generated label** | Auto-label PRs from AI agents so reviewers know to look more carefully |

---

## Priority Implementation Order

| Priority | Guardrail | Effort | Impact |
|---|---|---|---|
| **1** | Ruff config + `pyproject.toml` integration | 1 hour | Catches 80% of style issues |
| **2** | CI workflow (`.github/workflows/ci.yml`) | 2 hours | Enforces everything on every PR |
| **3** | Expand `copilot-instructions.md` + create `ARCHITECTURE.md` | 2-3 hours | Guides AI to produce correct patterns |
| **4** | Pre-commit hooks (ruff + reuse + gitleaks) | 30 min | Catches issues before they reach CI |
| **5** | Unit tests (priority 1-3 from test plan: pure logic tests) | 4-6 hours | ~45 tests covering core logic |
| **6** | mypy strict config | 1-2 hours | Type safety for API contract changes |
| **7** | Security scanning (pip-audit, gitleaks) in CI | 1 hour | Supply chain + secret protection |
| **8** | ADRs for key design decisions | 2-3 hours | Prevents AI from undoing architecture |
| **9** | Schema contract tests | 2-3 hours | Catches Evo API version drift |
| **10** | Branch protection rules on GitHub | 15 min | Enforces the whole chain |

---

## The Key Insight

The AI context files (Layer 1) tell the AI **what to do**. Everything else
(Layers 2-7) **verifies it actually did it correctly**. You need both:

- An AI with great instructions but no CI will eventually drift.
- A CI with great checks but no instructions will reject most AI-generated code, wasting cycles.

---

## Related Documents

- [Unit Test Plan](unit-test-plan.md) — Comprehensive test plan for ~155 test cases across the repo
