# Contribution 1: Adapter: vLLM Model Provider

**Contribution Number:** 1

**Student:** Srinitya Kondapally (@skonda29)

**Issue:** https://github.com/orthogonalhq/nous-core/issues/317

**Status:** Phase III Complete

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
- `self/subcortex/providers/src/providers/ollama/` (reference: local provider)
- `self/subcortex/providers/src/providers/openai/` (reference: OpenAI-compatible leaf)
- `self/subcortex/providers/src/protocols/openai-api/` (shared protocol I extended)
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
protocol, and make the API key optional so self-hosted servers work without
credentials. Use **Ollama** as the reference for the local-provider shape and
**OpenAI** as the reference for reusing the chat-completions protocol.

### Implementation Plan

**Understand:** nous-core needs vLLM as a local provider that reuses the
OpenAI-compatible path but doesn't force an API key.

**Match:** reference leaves are `providers/ollama/` (local provider shape) and
`providers/openai/` + `protocols/openai-api/` (OpenAI-compatible behavior).

**Plan:**

- Create the vLLM leaf with `definition.ts`, `adapter.ts`, `provider.ts`, `index.ts`.
- Configure vLLM values: `vendorKey: 'vllm'`, `isLocal: true`,
  `auth.required: false`, env var `VLLM_API_KEY`, endpoint `http://localhost:8000`.
- Reuse the shared chat-completions adapter and provider.
- Add a small `requireApiKey` option to the chat-completions provider so the
  auth header is only sent when a key exists — without touching `IModelProvider`
  or `TextModelInputSchema`.
- Run the catalog generator — don't hand-edit the generated files.
- Add vLLM-specific tests, especially the keyless path.
- Run typecheck and the full provider suite.

**Implement:** branch `feat/vllm-provider-317`.

**Review:**

- ✅ Follows the certified provider leaf structure
- ✅ Added under `self/subcortex/providers/src/providers/vllm/`
- ✅ Reuses the OpenAI-compatible protocol (no custom runtime branches)
- ✅ Optional API key added without changing `IModelProvider` or `TextModelInputSchema`
- ✅ Generated catalogs updated via the generator only
- ✅ Follows the Ollama reference for the local-provider shape
- ✅ Required exports added through `index.ts`
- ✅ Tests added and passing (273)
- ⚠️ `wellKnownProviderId`: I hand-authored it
  (`10000000-0000-0000-0000-000000000004`) to match the current `ollama` /
  `anthropic` leaves on the branch I built against. The issue's 2026-06-18
  update says leaves on the `feat/contributor-friendly-inference-provider-surface`
  branch should *not* hand-author this and should derive it from `vendorKey`. I
  noted this as a reconcile/rebase step before opening the PR against that
  branch (see Pull Request below).

**Evaluate:** verified with `check:generated`, `tsc`, and the full provider test
suite (273 passing).

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
      `chat-completions` protocol/adapter).
    - `adapter.ts` — re-exports the shared chat-completions adapter (same as the
      OpenAI leaf does), since vLLM is OpenAI-compatible.
    - `provider.ts` — factory that wraps `ChatCompletionsProvider` with
      `requireApiKey: false` and `apiKey: process.env.VLLM_API_KEY`.
    - `index.ts` — re-exports the leaf's public surface.
- Added a small `requireApiKey` option (default `true`, so existing OpenAI
  behavior is unchanged) to `protocols/openai-api/provider.ts`, and pulled the
  header building into a `requestHeaders()` helper that only attaches
  `Authorization` when a key is present.
- Ran the catalog generator (`generate:providers`) to register vLLM in the
  generated catalogs — did not hand-edit them.
- Updated the existing tests that hard-coded the old 3-provider roster.
- Added a vLLM-specific test file with 8 tests.

**Key commits:**

- `65d7e30f` — feat(providers): support optional API key in chat-completions protocol
- `0eae635c` — feat(providers): add vLLM provider leaf and register in catalog
- `b113a39e` — test(providers): add vLLM provider unit tests
- `7f9d864a` — test(providers): align adapter-resolver expectation with the vLLM leaf

**Challenges faced:**

- **The required API key (the main one).** The shared chat-completions provider
  throws `PROVIDER_AUTH_FAILED` if there's no key. vLLM is self-hosted, so I
  added a `requireApiKey` option (default `true`) and only send the
  `Authorization` header when a key is actually set. My vLLM factory passes
  `requireApiKey: false`. It's a tiny change and doesn't break any existing
  OpenAI tests.

- **Hard-coded provider rosters in *three* different places.** After adding the
  leaf, I updated the test files I expected to break — but the build still
  failed. `tsc` flagged `provider-definition-types.test.ts`, which uses a
  type-level `Equal/Expect` assertion that the provider-key union is *exactly*
  `'anthropic' | 'openai' | 'ollama'`; adding `vllm` widened the union, so it
  failed at **compile time**, not runtime. Then when I ran the full suite,
  `adapter-resolver.test.ts` failed at **runtime** because vLLM reuses the
  `chat-completions` adapter, so that key now appears once per leaf
  (`['anthropic','ollama','chat-completions','chat-completions','text']`). The
  resolver dedupes by key internally, so it's harmless — but the test pins the
  exact list. Lesson: adding a provider is a multi-place change, and `tsc` and
  `vitest` catch *different* hard-coded lists, so I need to run both.

- **A benign "unknown vendor" log.** Registering vLLM prints
  `stamped with unknown vendor 'vllm' — adapter will fall back to text`, because
  `KNOWN_PROVIDER_VENDORS` in `@nous/shared` only lists the four baseline
  vendors. I traced it and confirmed it only gates that advisory log — provider
  construction goes through `resolveProviderFactory('vllm')` and adapter
  resolution reads the leaf's `adapterKey: 'chat-completions'`, so vLLM resolves
  correctly. Adding `vllm` to that list would modify `@nous/shared` and break a
  shared test that pins its length to four, which the issue rules out, so I left
  it as a documented follow-up instead of widening scope.

- **Branch / `wellKnownProviderId` guidance.** The 2026-06-18 update describes a
  newer provider-leaf surface that derives IDs from `vendorKey`. The code I
  built against still hand-authors `wellKnownProviderId` (like the `ollama`
  leaf), so I matched the actual code rather than text that didn't match it, and
  flagged reconciling against the integration branch as the next step.

---

## Testing Strategy

**Tests added:** `src/__tests__/vllm-provider.test.ts` — 8 tests covering:

- The leaf resolves through `resolveProviderDefinition('vllm')` and
  `resolveProviderFactory('vllm')` with the right metadata (auth optional,
  endpoint `http://localhost:8000`).
- It constructs **without** an API key.
- `invoke()` with no key sends **no** `Authorization` header and returns a
  `ModelResponse`.
- `invoke()` with `VLLM_API_KEY` set **does** send `Authorization: Bearer …`.
- Invalid input is rejected with `ValidationError`.
- `stream()` yields content chunks from an SSE response and a final `done`.
- The factory returns a `ChatCompletionsProvider` instance.
- A 401 surfaces as `PROVIDER_AUTH_FAILED`.

I wrote these to follow the existing patterns — reusing the project's own
`resolveProviderDefinition` / `resolveProviderFactory` helpers and `vi`-mocked
`fetch`, the same way the pipeline tests do.

**Existing tests updated:**

- `src/__tests__/provider-definitions/provider-definitions.test.ts`
- `src/__tests__/provider-definitions/provider-definition-types.test.ts`
- `src/__tests__/provider-codegen.test.ts`
- `src/__tests__/provider-pipeline-integration.test.ts`
- `src/__tests__/adapter-resolver.test.ts`

**Validation performed:**

- **273 provider tests passing** after all changes.
- `check:generated` passing — catalogs are in sync with the leaves.
- `tsc --build` passing (this is what caught the type-level test).
- Also verified the keyless behavior directly against the compiled output with a
  small Node harness: with `VLLM_API_KEY` unset there's no `Authorization`
  header; with it set the header is `Bearer <value>`; empty input throws
  `ValidationError`.

---

## Code Changes

**Branch:** `feat/vllm-provider-317`

**Files created:**

- `self/subcortex/providers/src/providers/vllm/definition.ts`
- `self/subcortex/providers/src/providers/vllm/adapter.ts`
- `self/subcortex/providers/src/providers/vllm/provider.ts`
- `self/subcortex/providers/src/providers/vllm/index.ts`
- `self/subcortex/providers/src/__tests__/vllm-provider.test.ts`

**Files modified:**

- `self/subcortex/providers/src/protocols/openai-api/provider.ts` (optional API key)
- `self/subcortex/providers/src/provider-definitions.ts` (generated)
- `self/subcortex/providers/src/provider-adapters.ts` (generated)
- `self/subcortex/providers/src/provider-factories.ts` (generated)
- `self/subcortex/providers/src/__tests__/provider-definitions/provider-definitions.test.ts`
- `self/subcortex/providers/src/__tests__/provider-definitions/provider-definition-types.test.ts`
- `self/subcortex/providers/src/__tests__/provider-codegen.test.ts`
- `self/subcortex/providers/src/__tests__/provider-pipeline-integration.test.ts`
- `self/subcortex/providers/src/__tests__/adapter-resolver.test.ts`

---

## Pull Request

**PR Target Branch:** `feat/contributor-friendly-inference-provider-surface`
(per the issue's 2026-06-18 integration note)

**Status:** Not yet opened. Before opening the PR I want to rebase onto the
integration branch and reconcile the `wellKnownProviderId` guidance (derive from
`vendorKey` instead of hand-authoring), since the branch I implemented against
still uses the older hand-authored pattern. The implementation and tests are
complete and green; this is the one item left before submitting.

**Planned PR description:** Adds vLLM as a provider for nous-core. vLLM serves an
OpenAI-compatible API, so this reuses the shared chat-completions protocol rather
than custom code, and adds an optional API key so self-hosted servers work
without credentials (without touching `IModelProvider` or `TextModelInputSchema`).
The leaf is registered through the generator, and 273 provider tests pass.

---

## Learnings & Reflections

### Technical Skills Gained

I learned how nous-core structures provider integrations as certified leaves and
how a generator builds the aggregate catalogs from those leaves — and that you
never hand-edit the generated files. I also got a much clearer picture of how an
adapter (the request/response shape) is separate from the provider (the
transport), which is why vLLM can reuse OpenAI's adapter while still being its
own provider.

### Challenges Overcome

The most useful lesson was that "add a provider" touches more places than the
obvious list. One hard-coded roster failed at compile time and only `tsc` caught
it; another failed at runtime and only `vitest` caught it. Now I run both and
grep for every place that enumerates providers or adapters. Tracing the "unknown
vendor" log also taught me to confirm whether a warning is actually functional
before changing shared code to silence it — in this case it wasn't, and changing
it would have broken a shared test and gone out of scope.

### What I'd Do Differently Next Time

I'd run the **full** test suite (not just the files I expected to change) right
after the first implementation pass, to catch hard-coded roster assumptions
sooner. I'd also pin down the exact target branch and its conventions
(`wellKnownProviderId` handling) before writing code, so I don't have to
reconcile it at the end.

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
  - `self/subcortex/providers/src/providers/ollama/` (local provider shape)
  - `self/subcortex/providers/src/providers/openai/`
  - `self/subcortex/providers/src/protocols/openai-api/`
- Maintainer guidance from @atlamors on issue #317
