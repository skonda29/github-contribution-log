# github-contribution-log

## Issue

Issue: https://github.com/orthogonalhq/nous-core/issues/317

Title: Adapter: vLLM Model Provider (#317)

Status: Phase II Complete

---

## Problem Summary

The nous-core project currently supports multiple LLM providers through a common `IModelProvider` interface, but it does not yet support vLLM deployments. This issue requires implementing a vLLM provider that communicates with OpenAI-compatible `/v1/chat/completions` endpoints while supporting request validation, configuration management, streaming responses, and test coverage.

The provider must implement the existing `IModelProvider` contract without modifying shared interfaces or schemas. Once completed, users will be able to integrate self-hosted vLLM models into the platform using the same abstraction layer as other supported providers.

---

## Why I Chose This Issue

I chose this issue because it aligns with my interests in backend systems, AI infrastructure, and LLM application development. Through my experience working with Spring Boot services, Elasticsearch integrations, and AI-powered applications, I have developed an interest in understanding how model providers are integrated into production systems.

This issue has clear acceptance criteria, active maintainer support, and a well-defined scope. It will also give me exposure to provider abstractions, streaming APIs, and OpenAI-compatible interfaces that are commonly used in modern AI platforms.

I have already introduced myself on the issue thread and received guidance from the maintainer regarding an ongoing refactor that will simplify the implementation.

---

## Initial Understanding

The implementation will likely involve:

- Studying the existing OpenAI, Ollama, and Anthropic providers.
- Understanding the `IModelProvider` contract.
- Implementing `invoke()`, `stream()`, and `getConfig()`.
- Supporting OpenAI-compatible vLLM endpoints.
- Adding test coverage for request handling and streaming behavior.
- Exporting the provider through the provider index.

The maintainer has indicated that a provider refactor is currently underway, so implementation will begin after that work lands.

---

# Phase II — Reproduce and Plan (Week 2)

This phase covers standing up a local development environment, reproducing the
"bug" (for a feature-add issue, this means empirically confirming the missing
adapter and the design blocker), and writing the implementation plan.

A key finding up front: the **provider refactor the maintainer mentioned in
Phase I has now landed**. The package no longer uses a flat
`providers/src/vllm-provider.ts` layout as the issue text describes. It now uses
a **leaf-directory structure** with a code-generation step. The plan below
targets the current structure.

---

## 1. Local Development Environment

I cloned `orthogonalhq/nous-core` and got the relevant package building and
testing cleanly. Two environment issues were worth recording:

### Node version (blocker)

The repo targets **Node 22+** (`pnpm` v10 workspace). On **Node 24**,
`pnpm install` crashes with:

```
FATAL ERROR: invalid array length
... node::crypto::Hash::OneShotDigest ...
```

This is a known incompatibility between pnpm 10.6.2's one-shot `crypto.hash()`
and Node 24. **Fix:** use Node 22 (I used `node-v22.14.0`). After switching,
`pnpm install` completes in ~25s.

### Working command sequence

```bash
git clone https://github.com/orthogonalhq/nous-core.git
cd nous-core
corepack prepare pnpm@10.6.2 --activate
pnpm install

# The provider package depends on three workspace packages. A full repo
# `pnpm build` currently fails in an UNRELATED package (@nous/cortex-core,
# missing sibling dist), so build only the providers runtime closure:
pnpm --filter @nous/shared run build
pnpm --filter @nous/subcortex-inference-runtime run build
pnpm --filter @nous/autonomic-config run build

# Run the provider test suite:
npx vitest run self/subcortex/providers
```

Result: **19 test files, 265 tests — all passing.** Green baseline confirmed.

---

## 2. Reproducing the Gap

Because #317 is a feature-add, "the bug" is the absence of the vLLM adapter.
I confirmed this empirically against the built `dist` artifacts:

```text
Registered provider vendorKeys : [ 'anthropic', 'ollama', 'openai' ]
Has vllm definition?           : false
resolveProviderFactory("vllm") : undefined
ChatCompletionsProvider w/o key: THROWS -> PROVIDER_AUTH_FAILED - OpenAI API key required …
```

Three concrete findings:

1. **No vLLM definition / factory / adapter.** `PROVIDER_DEFINITIONS` and
   `resolveProviderFactory('vllm')` have no vLLM entry.
2. **The issue's target path is stale.** It says create
   `providers/src/vllm-provider.ts`, but the package was refactored into
   **leaf directories** — `src/providers/<vendor>/` each containing
   `definition.ts`, `adapter.ts`, `provider.ts`, `index.ts` — plus a
   generated-aggregate codegen step (`scripts/generate-provider-aggregates.mjs`).
3. **Core design blocker — optional auth.** vLLM is OpenAI-compatible, so the
   natural reuse is the existing `ChatCompletionsProvider` (exactly what the
   `openai` leaf does). But that provider's constructor **throws** when no API
   key is present, while vLLM auth is *optional* (self-hosted). The registry
   calls the factory with `apiKey: undefined` for local/optional-auth
   providers, so a naïve reuse would throw at construction. This is the one real
   design decision to resolve.

---

## 3. Implementation Plan

### Design decisions

- **Reuse the `chat-completions` protocol** (`/v1/chat/completions`,
  OpenAI-compatible) instead of writing a new wire format. vLLM speaks this
  natively.
- **Treat vLLM as local/self-hosted:** `isLocal: true`,
  `providerClass: 'local_text'`, `auth.required: false`. This mirrors how
  Ollama is treated and makes the registry skip the API-key requirement and
  preserve the user's custom endpoint (the registry only overrides endpoints
  for non-local providers).
- **`vendorKey: 'vllm'`** is safe — `ProviderVendor` in `@nous/shared` is an
  open string union, so no shared-package change is needed.
- **Default endpoint** `http://localhost:8000`; **default model** a placeholder
  such as `meta-llama/Llama-3.1-8B-Instruct` (vLLM serves whatever model was
  launched; `defaultModelId` only needs to be a non-empty string).
- **Well-known provider id:** `10000000-0000-0000-0000-000000000004`
  (next after anthropic `…001`, openai `…002`, ollama `…003`).

### Optional-auth handling (the one real code decision)

- **Recommended (Option B):** add a small, backward-compatible option to
  `ChatCompletionsProvider` — `requireApiKey?: boolean` (default `true`) — and
  only send the `Authorization` header when a key exists. The vLLM factory
  passes `requireApiKey: false`. This keeps a single code path, respects the
  scope boundary (only `IModelProvider` and `TextModelInputSchema` are
  off-limits), and the existing "constructor throws when no API key" test still
  passes via the default.
- **Alternative (Option A):** a dedicated `VllmProvider` in
  `providers/vllm/implementation.ts`. More isolated, but duplicates the
  chat-completions logic. I will go with Option B and note A in the PR.

### Files to create

- `self/subcortex/providers/src/providers/vllm/definition.ts` —
  `VLLM_PROVIDER_DEFINITION` (`vendorKey:'vllm'`, `protocol:'chat-completions'`,
  `adapterKey:'chat-completions'`,
  `auth:{ required:false, envVar:'VLLM_API_KEY', purpose:'api_key' }`,
  `isLocal:true`, capabilities `{ streaming:true, nativeToolUse:true }`),
  exported `as providerDefinition`.
- `self/subcortex/providers/src/providers/vllm/adapter.ts` — re-export the
  chat-completions adapter `as providerAdapter` (mirrors the `openai` leaf).
- `self/subcortex/providers/src/providers/vllm/provider.ts` — `providerFactory`
  (`vendorKey:'vllm'`) that builds
  `new ChatCompletionsProvider(config, { apiKey: options?.apiKey, requireApiKey: false })`.
- `self/subcortex/providers/src/providers/vllm/index.ts` — re-export
  `providerAdapter`, `providerDefinition`, `providerFactory`.
- `self/subcortex/providers/src/__tests__/vllm-provider.test.ts` — mirror
  `chat-completions-provider.test.ts`: getConfig, input validation
  (`ValidationError`), invoke happy path, streaming, 401/429/timeout
  classification, **and a new "constructs without an API key" case**.

### Files to modify

- `self/subcortex/providers/src/protocols/openai-api/provider.ts` — add the
  `requireApiKey` option and a conditional `Authorization` header (Option B).
- `self/subcortex/providers/src/index.ts` — export the vLLM provider entry
  (matching the openai leaf pattern).
- **Regenerate aggregates (do not hand-edit):**
  `pnpm --filter @nous/subcortex-providers run generate:providers` updates
  `provider-definitions.ts`, `provider-factories.ts`, and
  `provider-adapters.ts`. `check:generated` (part of `build`) enforces this.

### Tests that hardcode the provider set and must be updated

- `__tests__/provider-codegen.test.ts` → expected list becomes
  `['anthropic','ollama','openai','vllm']`.
- `__tests__/provider-pipeline-integration.test.ts` → add `'vllm'` to the
  expected `vendorKey` list.
- `__tests__/provider-definitions/provider-definitions.test.ts` → add a `vllm`
  row to the expected definitions table and the sorted vendor list.

### Acceptance-criteria mapping (from the issue)

| Criterion | How it's satisfied |
|---|---|
| Implements `IModelProvider` (invoke / stream / getConfig) | Via `ChatCompletionsProvider` reuse |
| Validates input against `TextModelInputSchema` | Inherited from `ChatCompletionsProvider` |
| Handles streaming correctly | Inherited SSE parser |
| Exported from `src/index.ts` | Added export |
| Tests in `src/__tests__/` | New `vllm-provider.test.ts` |
| Scope boundary respected | No edits to `IModelProvider` / `TextModelInputSchema` |

### Verification steps for the PR

```bash
pnpm --filter @nous/subcortex-providers run generate:providers
pnpm --filter @nous/subcortex-providers run check:generated
npx vitest run self/subcortex/providers   # expect green incl. new vllm tests
pnpm lint                                  # oxlint
```

Plus a grep across the wider monorepo for any other hardcoded provider-vendor
lists (router / cortex config) that may also enumerate vendors.

---

## Phase II Outcome

- Local environment stands up and the provider test suite passes (265 tests).
- The missing-adapter gap and the optional-auth blocker are both reproduced
  empirically.
- A concrete, file-by-file implementation plan is ready, aligned to the
  refactored leaf-directory + codegen structure rather than the stale path in
  the original issue text.

Next phase: implement the plan (Option B) and open the PR.
