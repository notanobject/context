# RFC: Improving Context MCP Search in Conservative Phases

**Date**: 2026-03-09
**Status**: Draft
**Repo**: https://github.com/neuledge/context
**Supersedes**: [rfc-context-mcp-search-improvements.original.md](/Users/tom/dev/context/docs/rfc-context-mcp-search-improvements.original.md)

## Summary

Context MCP search is useful today, but it is easy for LLM agents to query it in a way that returns zero results. The first problem to solve is not ranking quality in the abstract; it is that the MCP tool guidance currently teaches agents to use verbose, descriptive phrases even though the underlying search is keyword-based and combines terms strictly.

This RFC proposes a conservative, multi-PR plan:

1. Improve MCP guidance first.
2. Preserve a clear baseline so any later behavioral change can be compared against current behavior.
3. Only then consider a small fallback-based search improvement that activates on zero results.

This sequence is designed to maximize acceptance in a public repo by keeping the first contribution tiny, easy to review, and clearly justified.

## Problem Statement

The `get_docs` tool is a fast local reference for installed docs, but its current parameter guidance encourages query styles that often fail in practice.

Today, the `topic` field is described as:

> `"What you need help with (e.g., 'middleware authentication', 'server components')"`

Those examples are exactly the kind of broad descriptive phrases that currently perform poorly.

The current search implementation in [search.ts](/Users/tom/dev/context/packages/context/src/search.ts#L35) sanitizes the query and passes it to SQLite FTS5 `MATCH`. In this form, whitespace-separated tokens are interpreted with implicit `AND` semantics. In practice, that means adding extra generic words can reduce results to zero even when one core identifier would have succeeded.

## Evidence

The original draft RFC documented failures across libraries such as Hono, Zod, React Router, TanStack Query, Supabase, and the OpenAI Node SDK. The cleaned local search-quality matrix now covers registry-aligned package identities and includes Python libraries as well:

- `hono@latest`
- `zod@4.3.6`
- `openai@6.25.0`
- `@tanstack/react-query@5.90.3`
- `react-router@7.6.2`
- `fastapi@0.115.0`
- `pydantic@2.9.2`

The same broad pattern still holds:

- short API-name queries usually succeed
- descriptive multi-word queries are inconsistent
- natural-language questions often fail entirely

Representative examples from the current matrix:

| Library | Short query | Descriptive query | Natural query |
|---|---|---|---|
| `hono@latest` | `secureHeaders` → 4 | `secureHeaders middleware accept options` → 0 | `what parameters does Hono's secureHeaders middleware accept` → 0 |
| `openai@6.25.0` | `streaming` → 4 | `stream responses api events` → 1 | `how to stream responses with the OpenAI JavaScript SDK` → 0 |
| `@tanstack/react-query@5.90.3` | `useQuery` → 9 | `useQuery hook options staleTime` → 2 | `how to use useQuery hook with staleTime and refetchOnWindowFocus options` → 0 |
| `fastapi@0.115.0` | `Depends` → 11 | `dependency injection dependencies` → 8 | `how to use dependency injection with Depends in FastAPI` → 3 |

Real-world failure observed in OpenAI Codex:

> Asked: "What parameters does Hono's secureHeaders middleware accept?"
>
> 1. `{"library":"hono","topic":"secureHeaders middleware options parameters"}` -> validation error
> 2. `{"library":"hono-docs@main","topic":"secureHeaders middleware accept options"}` -> 0 results
> 3. another verbose retry -> 0 results
> 4. fell back to multiple web searches

This does not mean the current search is bad in general. It means the guidance given to agents does not match the search model they are actually calling.

## Goals

- Improve the chances that agents succeed on their first `get_docs` call.
- Keep the initial contribution small and low-risk.
- Preserve short-query behavior unless a later change can demonstrate improvement.
- Create a repeatable way to compare future search changes against current behavior.

## Non-Goals

- Replacing the current FTS implementation in the first phase.
- Adding embeddings or semantic search.
- Introducing schema migrations or larger indexing changes.
- Trying to make every natural-language query pass in one PR.

## Proposed Plan

### Phase 1: Guidance and Safe Workflow PR

This should be the first PR.

Changes:

- Update the `get_docs` tool description for `topic` to explain how to query effectively.
- Update the `search_packages` tool description so it also nudges agents toward short, package-name-oriented queries.
- Make the zero-results response actionable rather than generic.
- Clarify the intended registry-first workflow for missing packages:
  `search_packages` -> `download_package` -> `get_docs`.
- Add explicit fallback guidance when the registry path is unavailable or low-quality:
  use local sources via `context add` instead of falling straight back to web search.
- Clarify the library parameter or surrounding docs if that helps agents choose valid installed package names.

Proposed `topic` guidance:

```text
Short keyword or API name (for example: 'useQuery', 'cors'). Concise queries work best, so start with one or two words.
```

Proposed no-results guidance:

```json
{
  "results": [],
  "message": "No documentation found. Try a shorter query using just the API or function name, for example 'cors' instead of 'CORS middleware configuration'."
}
```

Proposed missing-package guidance:

```text
If the library is not installed, search the registry with `search_packages`, download it with `download_package`, then retry `get_docs`. If the registry package is unavailable or insufficient, build docs from source with `context add`.
```

Notes on scope:

- This phase should not make `get_docs` auto-download packages.
- This phase should not assume every community-registry package is high quality.
- Live smoke tests show the registry plumbing works, but package quality is uneven across packages, so the first PR should improve guidance and workflow without adding stronger runtime coupling.

Why this first:

- very small surface area
- low regression risk
- directly addresses the observed agent failure mode
- nudges agents toward the intended registry-first path without assuming the registry is always complete or reliable
- easiest public PR to review and accept

### Phase 2: Lightweight Baseline Comparison

Before changing search behavior, make comparison against current behavior explicit.

Changes:

- Preserve the current search path as the baseline behavior in tests.
- Extend the existing search-quality coverage with a few baseline-vs-improved comparisons rather than introducing a large new harness.
- Keep the real package matrix as a reporting tool, not only as a pass/fail gate.

Acceptance criteria:

- short-query behavior does not regress
- any later fallback or preprocessing change can show before/after evidence
- the repo has a documented, reproducible way to evaluate search quality

This phase can be combined with Phase 1 if done lightly, but it should not delay the first guidance PR.

### Phase 3: Conservative Search Fallback

Only after the baseline harness is in place, add a narrow fallback path.

Recommended shape:

- keep current search behavior unchanged as the first attempt
- only if that returns zero results, generate a few simplified variants
- return the first non-empty result set

Possible fallback variants:

- extract code-like identifiers such as `secureHeaders`, `useQuery`, `field_validator`
- remove common filler words such as `how`, `to`, `with`, `options`, `parameters`
- try a shorter phrase or top candidate token

Recommended guardrails:

- do not alter the result path when the original query already returns results
- do not add semantic search or heavier ranking changes in this phase
- keep the variant generator intentionally small and inspectable

This is conservative enough for a public repo because it limits behavioral change to current failure cases.

### Phase 4: Larger Search Rethinks

Defer until there is evidence that the conservative fallback is insufficient.

Examples:

- richer ranking changes
- semantic search
- embeddings
- schema or chunking changes

These may be worth exploring, but they should not be part of the initial contribution plan.

## Recommended PR Schedule

### PR 1: Guidance and Messaging

Scope:

- `get_docs` `topic` description
- `search_packages` guidance
- zero-results message
- missing-package guidance that prefers the registry path first
- explicit fallback guidance to local package building when registry packages are unavailable or inadequate
- tests covering the exposed guidance strings
- one workflow test or mocked integration test covering:
  missing package -> search_packages -> download_package -> get_docs
- brief docs update if helpful

Expected outcome:

- agents are more likely to start with short identifiers
- agents are more likely to use the registry flow before falling back to ad hoc web search
- no search behavior change

### PR 2: Search Evaluation Harness

Scope:

- lightweight baseline-vs-improved search assertions added to existing search-quality coverage
- reporting improvements to the real-world matrix
- explicit regression checks for short queries

Expected outcome:

- future search PRs can prove improvement rather than rely on anecdotes

### PR 3: Zero-Result Fallback

Scope:

- minimal query simplification
- only triggered when baseline search returns zero results
- measured against the PR 2 harness

Expected outcome:

- better descriptive/natural-language recovery without disturbing current successful short queries

## Testing Strategy

### 1. Unit Tests for Guidance

Phase 1 should include tests that assert the exposed MCP tool schema/help text contains:

- examples of short API-name queries
- a warning that concise queries work best
- actionable retry guidance on zero results
- guidance for the registry-first missing-package workflow
- a local-build fallback if the registry path does not work well enough

These tests do not prove agent behavior directly, but they do lock in the contract that agents see.

Phase 1 should also include at least one workflow-level test that does not depend on the live community registry:

- a mocked registry search response
- a mocked package download
- successful availability through `get_docs` after download

This keeps the intended path tested without over-assuming the maturity or consistency of the live registry service.

### 2. Baseline-Preserving Search Tests

For any search behavior change:

- keep a baseline search function or equivalent fixture
- compare baseline and new search on the same inputs

Required assertions:

- short queries are not worse than baseline
- zero-result descriptive queries improve where expected
- no-result cases remain understandable if they still fail

### 3. Real-World Matrix

Use the existing search-quality suite as the real-world comparison matrix across multiple libraries and doc sizes.

Recommended matrix categories:

- framework docs
- API client docs
- frontend/router docs
- validation/schema docs
- Python docs

Recommended query styles:

- short API name
- descriptive phrase
- natural-language question

For acceptance, the matrix should be used primarily as evidence and reporting. It should not force every natural-language query to pass before a conservative PR can land.

### 4. Manual Agent Verification

For the guidance PR, include a small manual appendix or PR note showing:

- one or two before/after agent prompts
- the tool call shape before the guidance change
- the tool call shape after the guidance change

This is especially useful because the first PR changes agent-facing instructions more than server internals.

## Risks and Mitigations

### Risk: Guidance-only changes do not fully solve the issue

Mitigation:

- that is acceptable for Phase 1
- the goal is to reduce obvious agent misuse first, not solve every query shape immediately

### Risk: Search fallback changes cause subtle regressions

Mitigation:

- only apply fallback on zero results
- compare against preserved baseline behavior
- protect short-query performance with explicit tests

### Risk: Public reviewers push back on complexity

Mitigation:

- keep PR 1 tiny
- separate evaluation harness from algorithmic changes
- postpone semantic search and larger redesigns

## Recommendation

Proceed with the phased plan:

1. land guidance and messaging first
2. make before/after measurement explicit
3. only then add a conservative zero-result fallback if needed

This keeps the first contribution well within the kind of change a public repo is likely to accept, while still leaving a clear path for measurable search improvement.

## Reference

The earlier draft is preserved at [rfc-context-mcp-search-improvements.original.md](/Users/tom/dev/context/docs/rfc-context-mcp-search-improvements.original.md) for historical context and raw baseline examples.
