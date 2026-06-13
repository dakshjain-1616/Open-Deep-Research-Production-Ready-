# We Pointed Claude Code + NEO at a Popular Agent Framework and Asked: "Is It Production-Ready?"

> One request, inside Claude Code. NEO cloned [`langchain-ai/open_deep_research`](https://github.com/langchain-ai/open_deep_research), read every source file, and wrote a 15-section due-diligence report — every finding tied to a real `file:line`. Then we made it bulletproof by re-verifying each citation against the actual code.

**⏱ 7-min read · For:** engineering leads, AI platform teams, anyone evaluating an agent framework before they build on it

---

## TL;DR

- We worked entirely **inside Claude Code** and asked it to run a repo-grounded production-readiness audit of Open Deep Research.
- Claude dispatched the heavy lifting to **NEO via MCP**. NEO cloned the repo locally, read the source end-to-end, and produced an **evidence-cited, 15-section report**.
- The verdict: **a clean, well-architected research prototype — but not production-ready.** Zero tests, no CI, almost no observability, no cost controls, and a one-line bug that silently kills research on any error.
- The part that makes it trustworthy: **Claude verified every citation against the clone.** Several line numbers were wrong on the first pass; we caught and fixed them before publishing.

---

## Why audit a framework you didn't write?

Open Deep Research is a popular reference implementation for agentic "deep research" — a supervisor that spins up parallel researcher sub-agents, each running search tools, compressing findings, and feeding a final report. If you're about to build a product on top of it (or borrow its patterns), you want to know two things: *how is it actually built*, and *what breaks when you put it under load?*

That's a due-diligence question. And the honest version of it is tedious: read the whole codebase, trace the graph, find the error handling, check the CI, map the dependencies — then translate all of it into risk.

So we tried it the lazy way, without leaving the editor:

> **From inside Claude Code, ask it to run a repo-grounded production-readiness audit of Open Deep Research — and let NEO do the investigation.**

---

## The setup: one request

We were in a normal Claude Code session and asked, in plain language, for a repository-**first** audit: clone the repo, read the real source, cite specific files and line numbers for every claim, and answer the questions that matter — how it's architected, how the research workflow runs, where it bottlenecks, what's risky, and what it would take to ship it.

Claude recognized this as exactly the kind of long-running, file-producing investigation to hand to **NEO via its installed MCP**. It pointed NEO at the repo and kept us posted in the same chat.

The sections it asked NEO to produce: Executive Summary, Repository Overview, Architecture Deep Dive, Component Analysis, Workflow Analysis, Strengths, Weaknesses, Scalability, Reliability, Security, Maintainability, Production Readiness, Risk Matrix, Top 10 Findings, and Prioritized Recommendations.

---

## What NEO did (while Claude drove)

NEO didn't summarize the README. Streaming back into the session, it:

- 🧬 **Cloned the repo** (`git clone` into a local workspace) and walked the full tree.
- 📖 **Read every core file end-to-end** — `deep_researcher.py`, `configuration.py`, `state.py`, `prompts.py`, `utils.py`, plus `security/auth.py`, the legacy implementations, the tests, `pyproject.toml`, `langgraph.json`, and the CI workflows.
- 🧩 **Traced the actual graph** — the three-level nested `StateGraph`: a main graph (clarify → research brief → supervisor → final report) wrapping a supervisor subgraph that delegates to parallel researcher subgraphs.
- 🔎 **Grepped to confirm line numbers** for each piece of evidence before writing it down.

The result: a **15-section report** where every finding points to a concrete `file:line`.

---

## What it found

A clean architecture with real, shippable-blocking gaps. The highlights — each grounded in the actual code:

### 🔴 The `or True` bug that silently ends research
In `supervisor_tools`, the error handler reads:

```python
if is_token_limit_exceeded(e, configurable.research_model) or True:
    # ... end the research phase, return partial results
```

That `or True` makes the branch **unconditional**. Any exception — a transient network blip, a tool error, anything — immediately ends the research phase and returns whatever was collected so far, with no retry and no error surfaced. It's one line (`deep_researcher.py:334`), and it's the single highest-scoring risk in the report (16/25).

### 🔴 Zero unit tests
The `tests/` directory looks reassuring until you open it — it's all LangSmith evaluation harnesses and benchmarks. There is **no unit test** for any function, state reducer, graph node, or tool. Refactoring safely is essentially impossible.

### 🟠 Almost no observability
`deep_researcher.py` has **zero logging calls**. Debugging a production failure means editing source and redeploying. There's no trace context across the three-level graph.

### 🟠 No timeouts on parallel research
The supervisor fans researchers out with `asyncio.gather(...)` and **no timeout** (`deep_researcher.py:300-305`). One researcher hitting a hanging API blocks the whole supervisor indefinitely.

### 🟠 No cost controls
`max_concurrent_research_units` defaults to 5 and `max_react_tool_calls` to 10 — so a single run can fan out to 50+ LLM calls and 50+ search calls, with **no budget enforcement** anywhere.

### 🟡 And a few more
A stale hardcoded `MODEL_TOKEN_LIMITS` map that breaks truncation for unknown models; four broad `except Exception` blocks; unbounded note accumulation in state; CI that runs only AI review bots (no test gate); and legacy code bundled into the production package.

NEO rolled these into a **production-readiness scorecard**:

| Dimension | Score (1–5) | Dimension | Score (1–5) |
|-----------|:-----------:|-----------|:-----------:|
| Test Coverage | 1 | Performance | 3 |
| Error Handling | 2 | Security | 3 |
| Observability | 1 | Documentation | 4 |
| CI/CD | 1 | Configuration | 4 |

…plus a ranked top-10 findings list and a prioritized, file-specific recommendation set.

> **The verdict:** an excellent research prototype and reference architecture — *not* a production system. The gaps are exactly the distance between "reference implementation" and "something you'd run unattended for paying users."

---

## The part that makes it trustworthy

Here's where this stops being a parlor trick. We didn't just publish what the agent wrote — **Claude verified it.**

Repo-grounded claims are easy to check: open the file, go to the line, confirm the code says what the report says. So we did, against commit `68fb7ab`. And we found something worth flagging: on the first pass, **several line numbers were wrong** — the `or True` bug cited a line or two off; a couple of `utils.py` functions were attributed to the wrong ranges entirely; one even pointed at a docstring instead of the error handler it described.

The *findings* were real — the bug, the missing tests, the swallowed errors all exist. But the citations have to be exact, or the evidence index is worthless. So Claude re-checked every `file:line` against the clone and corrected them. The published report's evidence index now matches the source line-for-line.

That's the loop that turns "fast" into "shippable": **NEO investigates and writes; Claude verifies against ground truth.**

---

## Why this combo works: Claude Code + NEO MCP

- **🏠 It runs locally.** NEO clones and reads the code on *your* machine. For due diligence on someone else's repo, that's just `git clone` in your own workspace — nothing leaves your environment.
- **🎯 It's task-oriented.** You give it an objective ("audit this repo, cite everything") and it plans the multi-step investigation itself.
- **📄 It produces artifacts.** The output is a committable report, not a chat transcript — ready to drop into a repo, review, and act on.
- **✅ It's verifiable.** Because every claim is a `file:line`, a second pass (human or agent) can confirm each one. That's what we did.

---

## Try it yourself

Install the NEO MCP into Claude Code, open a repo you're evaluating, and ask for a repo-grounded due-diligence audit — clone, read the source, cite every finding.

Then do the one thing that makes it bulletproof: **have Claude verify the citations against the actual code** before you trust the report.

---

*This is an independent technical review of public, open-source code, offered constructively. Open Deep Research is a strong prototype; the gaps noted here are the expected work of productionizing any reference implementation. Full report with all 15 sections and the verified evidence index: [`OPEN_DEEP_RESEARCH_DUE_DILIGENCE.md`](./OPEN_DEEP_RESEARCH_DUE_DILIGENCE.md).*

**Powered by NEO MCP + Claude Code.**
