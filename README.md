# Contribution 1: Adapter: vLLM Model Provider

**Contribution Number:** 1

**Student:** Srinitya Kondapally (@skonda29)

**Issue:** https://github.com/orthogonalhq/nous-core/issues/317

**Status:** Phase IV Complete — PR [#417](https://github.com/orthogonalhq/nous-core/pull/417) open against upstream

---

## Why I Chose This Issue

I chose this issue because it lines up directly with my interest in AI
infrastructure and how applications talk to model providers. The issue is about
adding **vLLM** as a supported model provider in nous-core, which meant I'd get
to learn how the system connects to a model server, handles (or skips)
authentication, validates input, streams responses, and exposes a provider
through the project's provider system.

It also fit my learning goals because it's a real open-source contribution with
clear acceptance criteria and an active maintainer. vLLM serves an
OpenAI-compatible `/v1/chat/completions` API, so I could focus on understanding
the project's existing provider patterns instead of writing a brand-new
protocol. The one twist that made it interesting: vLLM is usually self-hosted,
so unlike OpenAI it shouldn't require an API key — and figuring out how to
support that cleanly was the real engineering part of the issue.

---

## Understanding the Issue

### Problem Description

nous-core supports multiple model providers (Anthropic, OpenAI, Ollama) through
a common provider system, but it does not yet include a vLLM provider. vLLM is a
self-hosted inference server that exposes an OpenAI-compatible
`/v1/chat/completions` API. For nous-core to use it, the project needs a
certified provider leaf that declares vLLM's metadata, exports the adapter and
provider factory, points at the right endpoint, and becomes discoverable through
the generated provider catalogs.

The original issue asked for an `IModelProvider` implementation at a path like
`self/subcortex/providers/src/<vendor>-provider.ts`. A maintainer update on
2026-06-18 noted that that path is superseded and providers are now organized as
per-vendor leaves under `self/subcortex/providers/src/providers/<vendor>/`.

### Expected Behavior

nous-core should recognize vLLM as a provider and route to it through the shared
OpenAI-compatible protocol. Because vLLM is self-hosted, it should work
**without** an API key by default (pointing at `http://localhost:8000`), but
still send an `Authorization: Bearer` header if the user sets `VLLM_API_KEY`.

Expected behavior includes:

- vLLM has its own provider leaf under `self/subcortex/providers/src/providers/vllm/`.
- vLLM metadata is declared correctly (local provider, optional auth).
- vLLM reuses the OpenAI-compatible protocol instead of adding custom one-off code.
- The provider is exported through the leaf's `index.ts`.
- Generated provider catalogs are updated through the generator, not by hand.
- Tests verify the definition, the adapter/factory behavior, catalog generation,
  and — importantly — the keyless behavior.
- The shared `IModelProvider` interface and `TextModelInputSchema` are **not**
  modified.

### Current Behavior

vLLM is not available as a provider. Other leaves (Anthropic, OpenAI, Ollama)
exist as references, but there's no vLLM leaf. The closest reference for what I
needed is **Ollama**, because it's the existing *local* provider — it's marked
`isLocal: true` with `auth.required: false`, which is exactly the shape vLLM
needs.

### Affected Components

- `self/subcortex/providers/src/providers/vllm/` (new leaf)
- `self/subcortex/providers/src/providers/llama-cpp/` (primary reference: local OpenAI-compatible server, merged in #403)
- `self/subcortex/providers/src/providers/groq/` (reference: optional/keyed OpenAI-compatible leaf, merged in #404)
- `self/subcortex/providers/src/protocols/openai-api/` (shared protocol I reuse, unmodified)
- `self/subcortex/providers/src/provider-definitions.ts` (generated)
- `self/subcortex/providers/src/provider-adapters.ts` (generated)
- `self/subcortex/providers/src/provider-factories.ts` (generated)
- Provider tests under `self/subcortex/providers/src/__tests__/`

**Important note:** the generated catalog files have a
`// @generated ... do not edit by hand` header and a `check:generated` script
that fails the build if they drift. So they only get updated by running the
generator.

### Reproduction Process

Since this is a "missing feature" and not a crash, "reproducing the bug" for me
meant proving vLLM genuinely isn't there and finding what stands in the way.

**Environment setup.** I cloned nous-core and checked out a working branch
`feat/vllm-provider-317`. One thing that bit me first: my machine had Node 24,
and `pnpm install` kept crashing with `FATAL ERROR: invalid array length ...
node::crypto::Hash::OneShotDigest`. The project expects **Node 22**. Once I
switched to `node-v22.14.0`, install finished in ~25 seconds.

```bash
git clone https://github.com/orthogonalhq/nous-core.git
cd nous-core
corepack prepare pnpm@10.6.2 --activate
pnpm install

# the provider package needs a few workspace packages built first
pnpm --filter @nous/shared run build
pnpm --filter @nous/subcortex-inference-runtime run build
pnpm --filter @nous/autonomic-config run build

# baseline provider tests
npx vitest run self/subcortex/providers
```

**Steps to reproduce the gap:**

1. List the certified provider leaves: `ls self/subcortex/providers/src/providers`
2. Search for vLLM: `grep -R "vllm" -n self/subcortex/providers/src || true`
3. Print what's actually registered with a small probe script.

**Reproduction evidence.** My probe printed:

```text
Registered provider vendorKeys : [ 'anthropic', 'ollama', 'openai' ]
Has vllm definition?           : false
resolveProviderFactory("vllm") : undefined
ChatCompletionsProvider w/o key : THROWS -> PROVIDER_AUTH_FAILED - OpenAI API key required …
```

So three findings: (1) vLLM really isn't registered, (2) the code is now
per-vendor leaves with a generator, and (3) the OpenAI-compatible provider
*throws* without an API key — which is the one real obstacle, because vLLM
shouldn't need one.

---

## Solution Approach

### Analysis

The root problem is that there's no vLLM leaf. vLLM speaks the same
`/v1/chat/completions` format as OpenAI, so the right answer is **not** a custom
protocol or a one-off runtime branch — it's a provider leaf that reuses the
shared `protocols/openai-api/` implementation. The only thing standing in the
way is that this shared provider hard-requires an API key.

### Proposed Solution

Add vLLM as a provider leaf under
`self/subcortex/providers/src/providers/vllm/`, reuse the OpenAI-compatible
protocol, and support the keyless self-hosted case **without modifying the
shared provider**. Use **llama.cpp** as the reference for the local
OpenAI-compatible shape (it is the closest analog to vLLM) and **Groq** as the
reference for the optional API-key metadata.

### Implementation Plan

**Understand:** nous-core needs vLLM as a local provider that reuses the
OpenAI-compatible path but works without an API key by default.

**Match:** the reference leaves are `providers/llama-cpp/` (local
OpenAI-compatible server, the direct analog) and `providers/groq/` +
`protocols/openai-api/` (OpenAI-compatible behavior + auth metadata).

**Plan:**

- Create the vLLM leaf with `definition.ts`, `adapter.ts`, `provider.ts`, `index.ts`.
- Configure vLLM values: `vendorKey: 'vllm'`, `isLocal: true`,
  `auth.required: false`, env var `VLLM_API_KEY`, endpoint `http://localhost:8000`.
- Reuse the shared chat-completions adapter and provider.
- Handle the keyless path the way the merged llama.cpp leaf does — pass a
  `'no-auth'` placeholder key to the factory so the shared
  `ChatCompletionsProvider` key guard is satisfied, and forward a real bearer
  token when `VLLM_API_KEY` is set. **No change** to `ChatCompletionsProvider`,
  `IModelProvider`, or `TextModelInputSchema`.
- Let the built-in provider id derive from `vendorKey` (do **not** hand-author
  `wellKnownProviderId`).
- Run the catalog generator — don't hand-edit the generated files.
- Add vLLM-specific tests, especially the keyless path.
- Run typecheck and the full provider suite.

**Implement:** branch `feat/vllm-provider-317-leaf`, based on the integration
branch `feat/contributor-friendly-inference-provider-surface`.

**Review:**

- ✅ Follows the certified provider leaf structure
- ✅ Added under `self/subcortex/providers/src/providers/vllm/`
- ✅ Reuses the OpenAI-compatible protocol (no custom runtime branches)
- ✅ Keyless path handled via the `'no-auth'` placeholder (same as the merged
  llama.cpp leaf) — `IModelProvider`, `TextModelInputSchema`, and the shared
  `ChatCompletionsProvider` are all unchanged
- ✅ Generated catalogs updated via the generator only
- ✅ Follows the llama.cpp reference for the local-provider shape
- ✅ Required exports added through `index.ts`
- ✅ `wellKnownProviderId` derived from `vendorKey` (not hand-authored)
- ✅ Tests added and passing (378 provider tests)

**Evaluate:** verified with `check:generated`, `typecheck` (which enforces the
exact provider-key union), `lint`, and the full provider test suite (378
passing).

---

## Implementation Notes

### Week 1 Progress

I picked issue #317, read the description, acceptance criteria, and the
maintainer's notes, and introduced myself on the thread. I confirmed the issue
was ready to work on and noted the maintainer's update about the provider
refactor that changed where providers live.

### Week 2 Progress (Phase II — Reproduce and Plan)

I set up the project locally (the Node 22 lesson above), got the baseline
provider tests green (**265 tests** at that point), and wrote the probe that
proved vLLM was missing. I found the one real obstacle — the required API key —
and wrote a plan that matches the new per-leaf layout instead of the outdated
path in the original issue.

### Week 3 Progress (Phase III — Implement and Test)

**What I built:**

- Created the vLLM provider leaf under
  `self/subcortex/providers/src/providers/vllm/` with 4 files:
    - `definition.ts` — vLLM metadata (`vendorKey: 'vllm'`,
      `providerClass: 'local_text'`, `isLocal: true`, `auth.required: false`,
      env var `VLLM_API_KEY`, endpoint `http://localhost:8000`, reuses the
      `chat-completions` protocol/adapter). No hand-authored
      `wellKnownProviderId` — it derives from `vendorKey`.
    - `adapter.ts` — re-exports the shared chat-completions adapter (same as the
      llama.cpp and OpenAI leaves), since vLLM is OpenAI-compatible.
    - `provider.ts` — factory that wraps `ChatCompletionsProvider`, passing
      `apiKey: process.env.VLLM_API_KEY ?? 'no-auth'`.
    - `index.ts` — re-exports the leaf's public surface.
- Ran the catalog generator (`generate:providers`) to register vLLM in the
  generated catalogs — did not hand-edit them.
- Updated the existing tests that pin the provider roster.
- Added a vLLM-specific test file with 15 tests.

> **Phase IV reconciliation note.** My Phase II plan was written against an
> older branch and proposed adding a `requireApiKey` option to the shared
> `ChatCompletionsProvider` and hand-authoring `wellKnownProviderId`. When I
> rebased onto the actual PR target
> (`feat/contributor-friendly-inference-provider-surface`) I found the merged
> llama.cpp leaf (#403) had already established the convention for keyless local
> servers: pass a `'no-auth'` placeholder key and leave the shared provider
> untouched, with ids derived centrally from `vendorKey`. I dropped the
> shared-provider change and matched that accepted pattern instead. The sections
> above reflect what was actually built and submitted.

**Key commits:**

- `69f9914a` — feat(providers): add vLLM provider leaf and register in catalog
- `873dfd06` — test(providers): add vLLM unit tests and align provider rosters

**Challenges faced:**

- **The keyless self-hosted path (the main one).** The shared chat-completions
  provider throws `PROVIDER_AUTH_FAILED` if there's no key. vLLM is self-hosted
  and usually keyless. My first instinct (Phase II) was to add a `requireApiKey`
  flag to the shared provider, but the merged llama.cpp leaf showed the
  project's accepted answer: pass a `'no-auth'` placeholder key from the leaf
  factory so the guard is satisfied without modifying shared code, and forward a
  real bearer token when `VLLM_API_KEY` is set. This keeps the change contained
  to the leaf and matches precedent the maintainer already approved.

- **Hard-coded provider rosters in several places.** After adding the leaf, the
  build still failed until I updated every test that enumerates the roster.
  `provider-definition-types.test.ts` uses a type-level `Equal/Expect` assertion
  that the provider-key union is *exact*, so adding `vllm` failed at **compile
  time**; `provider-codegen`, `provider-definitions`, `provider-pipeline-integration`,
  and `adapter-resolver` pin the list at **runtime**. Lesson: adding a provider
  is a multi-place change, and `tsc` and `vitest` catch *different* hard-coded
  lists, so I run both.

- **`nativeToolUse` ambiguity across leaves.** The merged llama.cpp leaf
  advertises `capabilities.nativeToolUse: true`, but the merged Groq leaf
  intentionally omits it pending the shared native tool-use bridge. I matched
  llama.cpp (the direct local analog) and flagged the inconsistency in the PR
  description so the maintainer can decide — both PRs already note this is a
  project-side `protocol`/`adapterKey`-vs-capabilities contract issue, not a
  fault in the contribution.

---

## Testing Strategy

**Tests added:** `src/__tests__/vllm-provider.test.ts` — 15 tests covering:

- The definition metadata: local endpoint `http://localhost:8000`,
  `isLocal: true`, `providerClass: 'local_text'`, optional auth with
  `envVar: 'VLLM_API_KEY'`, the `chat-completions` protocol/adapter, and that it
  does **not** hand-author `wellKnownProviderId`.
- The factory creates a `ChatCompletionsProvider` and constructs **without** an
  API key.
- `invoke()` with no key sends `Authorization: Bearer no-auth`; with
  `VLLM_API_KEY` set it sends `Authorization: Bearer <key>`.
- Requests target `config.endpoint` (`localhost:8000`), not the OpenAI default.
- Invalid input is rejected with `ValidationError`.
- A 401 maps to `PROVIDER_AUTH_FAILED` and a non-ok response to
  `PROVIDER_UNAVAILABLE`.
- `invoke()` returns a `ModelResponse` on the happy path.
- `stream()` yields content chunks from an SSE response and a final `done`.

I wrote these to follow the existing patterns — mirroring the merged
`llama-cpp-provider.test.ts` and reusing the project's own
`resolveProviderDefinition` / `resolveProviderFactory` helpers and `vi`-mocked
`fetch`.

**Existing tests updated** (to include `vllm` in the pinned roster):

- `src/__tests__/provider-definitions/provider-definitions.test.ts`
- `src/__tests__/provider-definitions/provider-definition-types.test.ts`
- `src/__tests__/provider-codegen.test.ts`
- `src/__tests__/provider-pipeline-integration.test.ts`
- `src/__tests__/adapter-resolver.test.ts`

**Validation performed:**

- **378 provider tests passing** after all changes (27 files, 2 skipped).
- `check:generated` passing — catalogs are in sync with the leaves.
- `typecheck` passing — this is what enforces the type-level exact provider-key
  union in `provider-definition-types.test.ts`.
- `lint` (`oxlint self/`) — 0 errors.

---

## Code Changes

**Branch:** `feat/vllm-provider-317-leaf` (based on
`feat/contributor-friendly-inference-provider-surface`)

**Files created:**

- `self/subcortex/providers/src/providers/vllm/definition.ts`
- `self/subcortex/providers/src/providers/vllm/adapter.ts`
- `self/subcortex/providers/src/providers/vllm/provider.ts`
- `self/subcortex/providers/src/providers/vllm/index.ts`
- `self/subcortex/providers/src/__tests__/vllm-provider.test.ts`

**Files modified:**

- `self/subcortex/providers/src/provider-definitions.ts` (generated)
- `self/subcortex/providers/src/provider-adapters.ts` (generated)
- `self/subcortex/providers/src/provider-factories.ts` (generated)
- `self/subcortex/providers/src/__tests__/provider-definitions/provider-definitions.test.ts`
- `self/subcortex/providers/src/__tests__/provider-definitions/provider-definition-types.test.ts`
- `self/subcortex/providers/src/__tests__/provider-codegen.test.ts`
- `self/subcortex/providers/src/__tests__/provider-pipeline-integration.test.ts`
- `self/subcortex/providers/src/__tests__/adapter-resolver.test.ts`

The shared `protocols/openai-api/provider.ts` is **not** modified — the keyless
path is handled entirely inside the vLLM leaf factory via the `'no-auth'`
placeholder, matching the merged llama.cpp leaf.

---

## Pull Request

**PR:** [#417 — feat(providers): add vLLM provider leaf (#317)](https://github.com/orthogonalhq/nous-core/pull/417)

**PR Target Branch:** `feat/contributor-friendly-inference-provider-surface`
(the integration branch the maintainer designated on #317, and the same base the
merged llama.cpp [#403] and Groq [#404] leaves used — not `dev` or `main`).

**Status:** Open, awaiting maintainer review. Opened 2026-06-28 against the
upstream repo (`orthogonalhq/nous-core`) from my fork
(`skonda29/nous-core:feat/vllm-provider-317-leaf`). The maintainer (@atlamors)
was @mentioned on both the PR and issue #317.

**Summary:** Adds vLLM as a certified provider leaf. vLLM serves an
OpenAI-compatible API, so the leaf reuses the shared chat-completions
protocol/adapter rather than custom code. It is local (`isLocal: true`,
`providerClass: 'local_text'`) and works without an API key by default: the
factory passes a `'no-auth'` placeholder (the pattern the maintainer explicitly
accepted on the llama.cpp PR) and forwards a real bearer token when
`VLLM_API_KEY` is set. The built-in id derives from `vendorKey` (not
hand-authored), catalogs are regenerated via the generator, and the shared
`ChatCompletionsProvider` / `IModelProvider` / `TextModelInputSchema` are
unchanged. 378 provider tests pass; typecheck and lint are green.

---

## Maintainer Feedback Log

| Date | Source | Feedback / Guidance | My Response | Commit / Ref |
|------|--------|---------------------|-------------|--------------|
| 2026-06-11 | @atlamors on #317 | The original `IModelProvider` file path is superseded; implement as a certified provider leaf under `providers/<vendor>/`, target `feat/contributor-friendly-inference-provider-surface` (not `dev`/`main`), reuse `protocols/openai-api`, don't hand-edit generated catalogs. | Implemented the leaf under `providers/vllm/`, reused the shared protocol, regenerated catalogs via `generate:providers`, and opened the PR against the integration branch. | PR #417, commit `69f9914a` |
| 2026-06-18 | @atlamors on #317 | Integration branch updated: leaves use `ProviderDefinitionLeaf`, built-in ids derive from `vendorKey`, leaves should **not** hand-author `wellKnownProviderId`. | Rebased onto the updated integration branch and removed the hand-authored id; the id now derives from `vendorKey`. | commit `69f9914a` |
| 2026-06-23 | Merged llama.cpp PR [#403](https://github.com/orthogonalhq/nous-core/pull/403) (maintainer merge note) | Maintainer accepted the `'no-auth'` placeholder key as the keyless local-provider approach, since the shared provider has no clean no-auth mode yet; flagged `nativeToolUse` capability ambiguity as a project-side follow-up. | Adopted the same `'no-auth'` placeholder pattern instead of modifying the shared provider; matched `nativeToolUse: true` to llama.cpp and flagged the Groq inconsistency in my PR description for the maintainer to decide. | commit `69f9914a`, PR #417 description |
| 2026-06-28 | PR #417 opened | Awaiting review — @mentioned @atlamors on the PR and issue #317. | — | PR #417 |

> The dates above reflect the issue-thread guidance and merged-PR precedent I
> incorporated. This log will be updated with line-level review comments and my
> responses (with commit refs) as the PR is reviewed.

---

## Learnings & Reflections

### Technical Skills Gained

I learned how nous-core structures provider integrations as certified leaves and
how a generator builds the aggregate catalogs from those leaves — and that you
never hand-edit the generated files. I also got a much clearer picture of how an
adapter (the request/response shape) is separate from the provider (the
transport), which is why vLLM can reuse OpenAI's adapter while still being its
own provider. Reconciling onto the integration branch taught me the difference
between `ProviderDefinitionLeaf` (metadata only, id derived centrally from
`vendorKey`) and the older hand-authored definition, and why centralizing id
derivation removes a whole class of copy-paste mistakes.

### Challenges Overcome

The most useful lesson was that "add a provider" touches more places than the
obvious list. One hard-coded roster failed at compile time and only `tsc` caught
it (the type-level exact-union assertion); others failed at runtime and only
`vitest` caught them. Now I run both, plus `check:generated`, and grep for every
place that enumerates providers or adapters. The harder, more valuable challenge
was *not over-engineering the fix*: my Phase II plan was to modify the shared
`ChatCompletionsProvider` to make auth optional, but reading the merged
llama.cpp PR showed the project had already settled on a contained `'no-auth'`
placeholder at the leaf level. Choosing the established pattern over my own
shared-code change kept the PR small, low-risk, and consistent with precedent.

### What I'd Do Differently Next Time

I'd pin down the exact target branch and study the most recently merged PRs in
that area *before* writing code, instead of building against an older branch and
reconciling at the end. Reading #403 and #404 first would have pointed me
straight at the `'no-auth'` pattern and the `vendorKey`-derived id, saving a
rework pass. I'd also run the **full** test suite right after the first
implementation pass to surface hard-coded roster assumptions sooner.

### Teachable Insight (for future cohorts)

**Before implementing, find the most recently merged PR that did the same kind
of thing and copy its shape — including its accepted imperfections.** In a young,
fast-moving repo, the "right" pattern is whatever the maintainer last merged, not
necessarily what the docs or the original issue say. The merged llama.cpp leaf
told me three things the issue text didn't: keyless local servers use a
`'no-auth'` placeholder (not a shared-provider change), ids derive from
`vendorKey`, and a known capability ambiguity (`nativeToolUse`) was acceptable to
the maintainer as a tracked follow-up. Matching an accepted PR is the single
highest-signal way to get your own PR merged — and where you intentionally
diverge, say so explicitly in the description so the reviewer can decide quickly.

---

## Resources Used

- vLLM issue #317: https://github.com/orthogonalhq/nous-core/issues/317
- Provider adapter documentation:
  https://docs.nue.orthg.nl/docs/development/provider-adapters/quickstart
- Provider leaf anatomy:
  https://docs.nue.orthg.nl/docs/development/provider-adapters/provider-leaf-anatomy
- Testing checklist:
  https://docs.nue.orthg.nl/docs/development/provider-adapters/testing-checklist
- vLLM OpenAI-compatible server docs: https://docs.vllm.ai/
- Existing provider references:
  - `self/subcortex/providers/src/providers/llama-cpp/` (local OpenAI-compatible server — merged PR #403)
  - `self/subcortex/providers/src/providers/groq/` (keyed OpenAI-compatible leaf — merged PR #404)
  - `self/subcortex/providers/src/protocols/openai-api/`
- Merged precedent PRs I modeled the contribution on:
  - [#403 — llama.cpp provider leaf](https://github.com/orthogonalhq/nous-core/pull/403)
  - [#404 — Groq model provider leaf](https://github.com/orthogonalhq/nous-core/pull/404)
- Maintainer guidance from @atlamors on issue #317
