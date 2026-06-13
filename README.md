# Is Open Deep Research Production-Ready? — A Repo-Grounded Due-Diligence Audit

An evidence-cited engineering due-diligence review of [`langchain-ai/open_deep_research`](https://github.com/langchain-ai/open_deep_research), produced from inside **Claude Code** using the **NEO MCP** agent execution layer.

NEO cloned the repository, read every core source file, and wrote a **15-section report where every finding is tied to a real `file:line`** — then the citations were independently re-verified against the clone.

**Commit analyzed:** `68fb7ab` (HEAD on `main`, 2026-06-13)

---

## TL;DR

Open Deep Research is a **clean, well-architected research prototype** — a three-level nested LangGraph `StateGraph` with thoughtful structured-output usage and broad model-provider support. But it is **not production-ready**: no unit tests, no CI gate, near-zero observability, no rate limiting/cost controls, and a critical logic bug that silently ends research on *any* error.

> **Bottom line:** Excellent reference architecture. Productionizing it means investing in testing, observability, error handling, and operational infrastructure.

---

## Headline findings

| # | Finding | Severity | Evidence |
|---|---------|----------|----------|
| 1 | `or True` makes the error path unconditional — any exception silently ends research with partial results | 🔴 Critical | `deep_researcher.py:334` |
| 2 | Zero unit tests — `tests/` holds only eval harnesses | 🔴 Critical | `tests/` |
| 3 | No structured logging or tracing anywhere in the main graph | 🟠 High | `deep_researcher.py` |
| 4 | Four broad `except Exception` blocks can't separate transient vs permanent errors | 🟠 High | `deep_researcher.py:332, 431, 565, 661` |
| 5 | Stale `MODEL_TOKEN_LIMITS` map breaks truncation for unknown models | 🟠 High | `utils.py:788-829` |
| 6 | `asyncio.gather` on researchers has no timeout — one stuck researcher blocks the supervisor | 🟠 High | `deep_researcher.py:300-305` |

Full ranked list of 10 in the report. Risk matrix highest score: **R1 (`or True` bug) = 16/25**.

## Production-readiness scorecard (1–5)

| Dimension | Score | Dimension | Score |
|-----------|:-----:|-----------|:-----:|
| Test Coverage | 1 | Performance | 3 |
| Error Handling | 2 | Security | 3 |
| Observability | 1 | Documentation | 4 |
| CI/CD | 1 | Configuration | 4 |

*(19/40 — prototype tier)*

---

## What's in this repo

| File | What it is |
|------|------------|
| [`OPEN_DEEP_RESEARCH_DUE_DILIGENCE.md`](./OPEN_DEEP_RESEARCH_DUE_DILIGENCE.md) | The full 15-section due-diligence report (architecture deep dive, component & workflow analysis, scalability/reliability/security/maintainability assessments, risk matrix, top 10 findings, recommendations, and a grep-verified evidence index) |
| [`BLOG.md`](./BLOG.md) | A narrative write-up: how Claude Code + NEO produced the audit, and what it found |

## How it was produced

This is a showcase of the **Claude Code + NEO MCP** division of labor:

- **Claude Code** — framed the audit in plain language, dispatched it, and **verified every citation** against the cloned source before publishing.
- **NEO (via MCP)** — ran locally, cloned the repo, read the source end-to-end, and wrote the evidence-based report. Your code never leaves your machine.

Every `file:line` in the report was re-checked against commit `68fb7ab`. Citations that didn't match were corrected; the evidence index reflects the verified locations.

> This is an independent technical review of public, open-source code. The findings are offered constructively — Open Deep Research is a strong prototype, and the gaps noted are the expected distance between "reference implementation" and "production system."

## License

- Report and blog text: [CC BY 4.0](./LICENSE)
- Open Deep Research itself is MIT-licensed by LangChain; this repository only analyzes it and does not redistribute its code.
