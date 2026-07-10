# Contribution 2: Adapter: Azure OpenAI Model Provider

**Contribution Number:** 2

**Student:** Srinitya Kondapally (@skonda29)

**Issue:** https://github.com/orthogonalhq/nous-core/issues/304

**Status:** Phase III Complete — Azure OpenAI BYOK leaf implemented and tested; branch pushed to my fork (2026-07-09)

**Other contributions:** [Contribution 1 — vLLM Model Provider (#317)](README.md) (Phase IV Complete — PR [#417](https://github.com/orthogonalhq/nous-core/pull/417) open)

---

## Why I Chose This Issue

I chose this issue because it's the natural continuation of what I built in Contribution 1. After adding the vLLM provider leaf (#317 / PR #417), I already understand how nous-core structures provider integrations as certified leaves, how the generated catalogs are produced, and how a leaf reuses the shared `protocols/openai-api/` chat-completions protocol. Issue #304 — **Azure OpenAI Model Provider** — builds directly on that foundation, but with a harder engineering problem.

Azure OpenAI is not a drop-in OpenAI clone. It speaks the same chat-completions *payload* shape, but it differs from OpenAI in two specific ways that the shared `ChatCompletionsProvider` does not currently handle:

1. **Auth header.** Azure requires an `api-key: <key>` request header, not `Authorization: Bearer <key>`.
2. **URL shape.** Azure routes by *deployment name*: `{endpoint}/openai/deployments/{deployment}/chat/completions?api-version=<ver>`, rather than `{base}/v1/chat/completions`.

On vLLM, I learned that the maintainer prefers protocol-level gaps to be fixed *centrally* in the shared protocol rather than worked around per leaf. Azure is a clean, real example of exactly that pattern — and fixing it centrally means the same extension could benefit other providers in the future. That made this the right next issue to take on.

---

## Understanding the Issue

### Problem Description

nous-core supports multiple model providers through the certified provider-leaf system. On the integration branch (`feat/contributor-friendly-inference-provider-surface`), the roster is: `anthropic`, `codex-cli`, `deepinfra`, `github-copilot-cli`, `groq`, `huggingface-tgi`, `llama-cpp`, `moonshot`, `ollama`, `openai`, `openclaw`, `openrouter`, `perplexity`, and `vllm` (my Contribution 1). It does **not** include **Azure OpenAI**, which is one of the most common enterprise deployment targets for OpenAI models — through Azure's hosting, compliance guarantees, and regional availability.

The issue asks for an Azure OpenAI provider implemented against the **current** provider-leaf contract (`ProviderDefinitionLeaf`, ids derived from `vendorKey`, generated catalogs updated by the generator). Integration target: `feat/contributor-friendly-inference-provider-surface`.

### Scope (narrowed by the maintainer, 2026-07-08)

Before I started building, @atlamors did a deeper pass on the Azure side and **narrowed the issue** so the implementation target was unambiguous. The agreed scope is **Azure OpenAI as a direct, user-supplied ("BYOK") data-plane connector only**:

- the user already has an Azure OpenAI resource and a model deployment;
- the user provides the Azure OpenAI endpoint;
- the user provides an API key;
- the configured model value is treated as the **Azure deployment name**.

Explicitly **out of scope** for this leaf (these belong to the managed inference relay work tracked in #421): Azure AI Foundry project management, deployment creation, Microsoft Entra ID / managed-identity auth, quota or capacity discovery, regional routing, provider fallback, and managed Nue Cloud billing/routing. The maintainer also asked that if the shared OpenAI-compatible surface makes endpoint/path composition, deployment-name-vs-model-name behavior, or API-key header handling awkward, I **flag it in the PR** rather than heavily working around it in the Azure leaf — those protocol-boundary issues are maintainer-side cleanup. My implementation follows exactly this narrowed scope, and I documented the descope directly in `definition.ts`.

### Expected Behavior

Once #304 is done:

- Azure OpenAI has its own provider leaf under `self/subcortex/providers/src/providers/azure-openai/`.
- It reuses the OpenAI-compatible chat-completions request/response *body* shape.
- It authenticates with the Azure `api-key` request header and an env var `AZURE_OPENAI_API_KEY`.
- It builds the request URL in the Azure deployment style with a required `api-version` query parameter.
- The built-in provider id derives from `vendorKey` — no hand-authored `wellKnownProviderId`.
- Generated catalogs are updated via the generator (`generate:providers`), not by hand.
- Tests cover the definition metadata, the `api-key` header (and absence of `Authorization`), the deployment + `api-version` URL, invoke and stream happy paths, and error mapping (401, 429, other).

### Current Behavior

Azure OpenAI is not registered as a provider. Running `resolveProviderDefinition('azure-openai')` on the integration branch throws:

```text
Provider definition is missing for vendor key 'azure-openai'
```

The `ProviderVendorKey` type union has no `azure-openai` member, so the TypeScript compiler would catch any attempted reference at compile time too.

The closest reference leaves are **Groq** and **DeepInfra** — keyed, OpenAI-compatible, with a custom `defaultEndpoint` — but neither uses the `api-key` header or the Azure deployment/`api-version` URL shape. The shared `ChatCompletionsProvider` only knows how to build `{endpoint}{staticPath}` and always sends `Authorization: Bearer <key>`.

### Affected Components

- `self/subcortex/providers/src/providers/azure-openai/` (new leaf — four files)
- `self/subcortex/providers/src/protocols/openai-api/provider.ts` (shared protocol — needs a central extension for the `api-key` header and Azure deployment URL, because both `invoke()` and `stream()` hard-code the Bearer header and static path today)
- `self/subcortex/providers/src/schemas/provider-definition.ts` (the contract already models what Azure needs — `scheme: 'raw'`, optional `headers` — but the shared provider doesn't consume them yet)
- `self/subcortex/providers/src/provider-definitions.ts` / `provider-adapters.ts` / `provider-factories.ts` / `adapter-resolver.ts` (generated — updated by `scripts/generate-provider-aggregates.mjs`, never by hand)
- Provider tests under `self/subcortex/providers/src/__tests__/` (new test file for Azure, plus updates to five roster-pinning tests — the same set I updated for vLLM)

### Reproduction Process

Since this is a missing-feature issue, "reproducing the bug" meant proving the gap is real and pinpointing exactly what blocks a working implementation.

**Environment setup.** I reused the environment I stood up for Contribution 1. My setup approach is **README instructions + package-script inspection** (there is no dev container in the repo). The provider package documents its own workflow through `self/subcortex/providers/package.json`:

- `generate:providers` → `node scripts/generate-provider-aggregates.mjs`
- `check:generated` → same script with `--check` (fails CI if the catalog drifts from disk)
- `build` / `typecheck` → `check:generated` then `tsc --build`
- tests via Vitest (`vitest run --pool threads` at the repo root; package-level config at `self/subcortex/providers/vitest.config.ts`)

Steps I ran to reach a working, testable baseline:

```bash
# 1. Fetch and check out an issue-named branch off the integration branch
git fetch origin feat/contributor-friendly-inference-provider-surface
git checkout -B feat/azure-openai-provider-304 \
  origin/feat/contributor-friendly-inference-provider-surface

# 2. Confirm the checked-in catalog is in sync (no stale-generation artifact)
cd self/subcortex/providers
node scripts/generate-provider-aggregates.mjs --check

# 3. Verify the green baseline
npx vitest run self/subcortex/providers/src/__tests__
# Result: 32 files passed / 2 skipped, 440 tests passed / 4 skipped, ~3s
```

One challenge: on Contribution 1, Node 24 broke `pnpm install` with a `crypto::Hash::OneShotDigest` OOM, so I had pinned Node 22 for that phase. For this phase I only needed to switch branches and reuse the existing install — the Vitest suite runs cleanly on Node 24 (v24.4.1) + pnpm 10.6.2 against the existing `node_modules`. Resolution: don't re-run `pnpm install` this phase; if I hit a native-module failure when I do need to reinstall for Phase III, I'll fall back to Node 22 exactly as before.

A second challenge: `main` still has the older direct-`IModelProvider` layout. Resolution: I cut my branch from the integration branch and did all reproduction there, so everything below reflects the actual contract I'll submit against.

**Steps to prove the gap (numbered, can be run by anyone):**

1. **Confirm there is no Azure leaf on disk:**
   ```bash
   ls self/subcortex/providers/src/providers/
   ```
   *Actual:* 14 vendors listed (anthropic … vllm), no `azure-openai`.

2. **Confirm Azure appears nowhere in the provider source:**
   ```bash
   grep -rin "azure" self/subcortex/providers/src
   ```
   *Actual:* no matches at all.

3. **Confirm the catalog is in sync — the absence is real, not stale:**
   ```bash
   cd self/subcortex/providers
   node scripts/generate-provider-aggregates.mjs --check
   ```
   *Actual:* exits 0. The catalog matches disk, so there is no Azure leaf anywhere.

4. **See the auth-header obstacle** — open `self/subcortex/providers/src/protocols/openai-api/provider.ts`, lines 92–98 (`invoke`) and 154–160 (`stream`):
   *Actual:* both methods hard-code `Authorization: Bearer ${this.apiKey}` and build the URL as `` `${endpoint}${this.completionsPath}` ``. There is no branch that reads the leaf's declared `auth.header`.

5. **See the URL-shape obstacle** — the constructor's only URL knob is `completionsPath` (lines 24, 64, 73), a static string defaulting to `/v1/chat/completions` with no query-string support and no deployment-name templating.

6. **See that the contract already anticipates the fix** — open `self/subcortex/providers/src/schemas/provider-definition.ts`, line 34:
   ```typescript
   export const ProviderAuthHeaderSchemeSchema = z.enum(['raw', 'bearer']);
   ```
   The `raw` scheme is already modeled in the contract — it just isn't consumed by the shared provider yet.

**Reproduction evidence summary:**

| | Expected (after #304) | Actual (today) |
|---|---|---|
| Registration | `resolveProviderDefinition('azure-openai')` returns the definition | Throws `Provider definition is missing for vendor key 'azure-openai'` |
| Auth header | Request carries `api-key: <AZURE_OPENAI_API_KEY>` | Hard-codes `Authorization: Bearer <key>` (`provider.ts:97,159`) — Azure rejects this |
| URL shape | `POST {endpoint}/openai/deployments/{deployment}/chat/completions?api-version=<ver>` | `{endpoint}{staticPath}` — no query string, no deployment templating (`provider.ts:92,154`) |
| Provider id | Derived from `vendorKey` via `deriveBuiltInProviderId` | N/A (no leaf exists) |

The surface symptom ("no Azure provider") sits on top of a concrete central limitation: the shared `ChatCompletionsProvider` neither honors a leaf's declared `auth.header` nor supports a query string or templated deployment in its URL.

---

## Solution Approach

### Analysis

The root problem is not just a missing leaf — it's that the shared `ChatCompletionsProvider` (`protocols/openai-api/provider.ts`) ignores the leaf-declared `auth.header` and can only build `endpoint + static path` URLs. Azure needs both things: the `api-key` header scheme (already modeled in the schema as `scheme: 'raw'`) and a deployment-templated URL with a required `api-version` query parameter.

The right answer is *not* a bespoke Azure provider that bypasses the shared protocol — Azure's request and response bodies are fully OpenAI-compatible, so duplicating the protocol just to change two request-level details would be the wrong abstraction. The right answer is a **central, minimal extension to the shared provider** that reads the leaf's declared header and URL options, then a **Azure leaf** that drives that behavior through its definition. This matches the maintainer's standing guidance and keeps all fourteen existing leaves unchanged (bearer stays the default).

### Proposed Solution

Add an `azure-openai` leaf under `self/subcortex/providers/src/providers/azure-openai/` that reuses the shared chat-completions protocol with two new options I'll add to `ChatCompletionsProvider`: a configurable auth header (name + scheme) and a deployment-aware URL builder. The leaf's definition drives both options, so no leaf-specific logic ends up in the shared provider.

I'll use **Groq** and **DeepInfra** as the skeleton for the leaf shape (keyed, OpenAI-compatible, custom endpoint), **Perplexity** as the model for fail-closed key resolution, and **Anthropic** as proof that `scheme: 'raw'` is an accepted pattern in this codebase. My own vLLM leaf gives me a first-party playbook for the full submission workflow on this exact branch.

### Implementation Plan

**Understand:** nous-core needs Azure OpenAI as a certified provider leaf that reuses the shared chat-completions protocol, but sends `api-key: <key>` instead of `Authorization: Bearer` and builds a deployment-style URL with a required `api-version`. The fix must be central (in the shared provider), not a per-leaf workaround.

**Match:**

- **Groq / DeepInfra** — leaf skeleton: `definition.ts`, `adapter.ts`, `provider.ts`, `index.ts`; keyed auth; custom `defaultEndpoint`; factory wraps `ChatCompletionsProvider`.
- **Perplexity factory** — fail-closed credential resolution: resolve `AZURE_OPENAI_API_KEY` explicitly in the factory so the shared provider's `OPENAI_API_KEY` fallback is never reachable for Azure requests.
- **Anthropic `implementation.ts`** — accepted precedent for `header: { name: 'x-api-key', scheme: 'raw' }` and for static `headers`. My job is to lift this mechanism into the shared OpenAI-compatible provider, not repeat it in a bespoke one.
- **My own vLLM leaf (#417)** — the full workflow: cut from the integration branch, add the leaf, run `generate:providers`, update the five roster-pinning tests, run `check:generated` + `typecheck` + `lint` + the full provider suite.

**Plan:**

- **Extend the shared protocol** in `self/subcortex/providers/src/protocols/openai-api/provider.ts`:
  - Add an `authHeader` option (`{ name: string; scheme: 'raw' | 'bearer' }`) so `raw` produces `<name>: <key>` and `bearer` keeps `Authorization: Bearer <key>` (the default — all existing leaves are unchanged).
  - Factor the header construction into a single private helper consumed by both `invoke` and `stream`.
  - Add a `completionsPathTemplate` option that supports `{deployment}` interpolation and a `queryParams` option for `api-version=<ver>`. Keep the current static `completionsPath` as the default.
  - Merge static `headers` from the definition (mirrors Anthropic's `anthropic-version` header pattern).
- **Author the leaf** — `self/subcortex/providers/src/providers/azure-openai/`:
  - `definition.ts`: `vendorKey: 'azure-openai'`, `displayName: 'Azure OpenAI'`, `protocol: 'chat-completions'`, `adapterKey: 'chat-completions'`, `auth: { envVar: 'AZURE_OPENAI_API_KEY', header: { name: 'api-key', scheme: 'raw' }, required: true }`, `isLocal: false`. No `wellKnownProviderId` — it derives from `vendorKey`.
  - `provider.ts`: factory that resolves `AZURE_OPENAI_API_KEY` explicitly (fail-closed like Perplexity), and passes the deployment-URL template and `api-version` options to the extended shared provider.
  - `adapter.ts` and `index.ts`: re-export the shared chat-completions adapter (mirror OpenAI/Groq — a 1–2 line file each).
- **Regenerate catalogs** — `node scripts/generate-provider-aggregates.mjs`. Never hand-edit the generated files.
- **Add tests** — `self/subcortex/providers/src/__tests__/azure-openai-provider.test.ts`: definition metadata, `api-key` header present + no `Authorization` header, deployment name and `api-version` in the URL, invoke and stream happy paths, 401/429/other error mapping.
- **Update roster-pinning tests** — the same five files I updated for vLLM: `adapter-resolver.test.ts`, `provider-definitions.test.ts`, `provider-definition-types.test.ts`, `provider-codegen.test.ts`, `provider-pipeline-integration.test.ts`.

**Implement:** branch `feat/azure-openai-provider-304`, based on the integration branch. Plan to land in reviewable commits: (1) central `ChatCompletionsProvider` extension + unit tests for the new options, (2) the `azure-openai` leaf + regenerated catalogs, (3) the Azure provider tests, (4) roster-test alignment. Before writing any code, I'll comment on #304 to confirm the maintainer wants the header/URL generalization in the shared provider — that's the one design decision with real trade-offs, and the maintainer's standing request is to raise those before implementing.

**Review:**

- ✅ Follows the certified provider leaf structure
- ✅ New leaf under `self/subcortex/providers/src/providers/azure-openai/`
- ✅ Reuses the OpenAI-compatible protocol for the request/response body (no custom wire format)
- ✅ Auth header and URL generalization in the shared provider, driven by the leaf definition
- ✅ Existing leaves unaffected (bearer is still the default)
- ✅ `wellKnownProviderId` derived from `vendorKey`, not hand-authored
- ✅ Generated catalogs updated via the generator only
- ✅ No `OPENAI_API_KEY` fallback possible for Azure requests (fail-closed)

**Evaluate:** done = green suite (baseline 440 + new Azure tests, 0 failures), `check:generated` clean, the `api-key` header and deployment URL behavior asserted by dedicated tests, no regression in the other 14 providers, and the PR open against the integration branch with the design question raised up front.

---

## Implementation Notes

### Week 1 Progress

I picked issue #304, read the description, the maintainer's guidance in the issue body, and the accepted provider-adapter docs. I referenced my Contribution 1 work on the vLLM leaf and commented on the issue:

> "Hi @atlamors, I previously worked on Adapter: vLLM Model Provider #317 and would like to take up this issue and get started."

I confirmed the issue was open and unassigned, noted that the integration target is the same branch I already worked on for #317, and identified from the start that Azure would require central changes to the shared protocol — not just a new leaf.

### Week 2 Progress (Phase II — Reproduce and Plan)

I fetched and checked out the integration branch into a new issue-named working branch (`feat/azure-openai-provider-304`), confirmed the existing `node_modules` + Node 24 environment was good for this phase without a fresh install, and ran the baseline suite (**440 tests, 32 files, green**).

I then did the systematic investigation described in the Reproduction section above: confirmed Azure is absent from all provider source, verified `check:generated` is clean (so the absence is not a generator artifact), and read the shared `ChatCompletionsProvider` line by line to pin down the exact obstacles — hard-coded Bearer header at lines 97 and 159, static URL construction at 92 and 154, and no consumption of the leaf's declared `auth.header`. I also ran `git blame` on both the shared provider and the schema to understand when these choices were made and why, which gave me confidence that the `raw` header scheme in the contract is the right design hook for Azure.

I studied four reference points in the codebase: Groq and DeepInfra for the leaf skeleton, Perplexity for fail-closed key resolution, and Anthropic for accepted raw-header precedent. I also reviewed my own vLLM leaf as a same-branch reference for the full submission workflow.

The plan above is the output of that investigation.

---

## Testing Strategy

Tests were written following the existing per-provider test files (I modeled them on `perplexity-provider.test.ts` and `chat-completions-provider.test.ts`, and reused the project's own `vi`-mocked `fetch` pattern and the leaf's exported factory/helpers rather than reaching into private internals).

### New tests exercising the fix

**1. Shared-provider change — `src/__tests__/chat-completions-provider.test.ts` (+72 lines).**
These target the code path I actually changed in `protocols/openai-api/provider.ts` (the new `authHeaderName` / `authHeaderScheme` options and the `buildAuthHeader()` helper):
- default behavior is unchanged — with no auth options it still sends `Authorization: Bearer <key>` (regression guard for the other 14 leaves);
- with `authHeaderScheme: 'raw'` + `authHeaderName: 'api-key'`, the request sends `api-key: <key>` and **no** `Authorization` header;
- both `invoke()` and `stream()` go through the same helper, so the header behavior is asserted on both paths.

**2. Azure leaf — `src/__tests__/azure-openai-provider.test.ts` (302 lines) and `src/__tests__/providers/azure-openai.test.ts` (102 lines).**

*Definition metadata:*
- placeholder endpoint/model id that the user must replace;
- requires a key via `AZURE_OPENAI_API_KEY` and declares the raw `api-key` header;
- uses the `chat-completions` protocol/adapter (same wire format as OpenAI);
- does **not** declare model-list/health-check discovery (narrowed BYOK scope);
- does **not** hand-author `wellKnownProviderId`.

*`buildAzureCompletionsPath()` (the exported URL helper):*
- composes `/openai/deployments/{deployment}/chat/completions?api-version=…`;
- URL-encodes deployment names containing special characters.

*Factory + transport:*
- throws `PROVIDER_AUTH_FAILED` when no key is available;
- **never** falls back to `OPENAI_API_KEY` (fail-closed credential boundary);
- resolves the key from `AZURE_OPENAI_API_KEY` when no option is passed;
- `invoke()` targets the deployment-scoped path with the default api-version;
- `invoke()` honors `AZURE_OPENAI_API_VERSION` when set;
- `invoke()` sends the key as a raw `api-key` header, not `Authorization: Bearer`;
- `invoke()` validates input and rejects an invalid shape with `ValidationError`;
- `invoke()` returns a `ModelResponse` on the happy path;
- 401 → `PROVIDER_AUTH_FAILED`, 429 → `PROVIDER_UNAVAILABLE`, other non-ok → `PROVIDER_UNAVAILABLE`;
- `stream()` yields content chunks and a final `done` chunk via the deployment-scoped path.

### Roster-pinning tests updated (same five hard-coded rosters vLLM taught me about)
- `src/__tests__/adapter-resolver.test.ts`
- `src/__tests__/provider-definitions/provider-definitions.test.ts`
- `src/__tests__/provider-definitions/provider-definition-types.test.ts` (the compile-time exact-union assertion)
- `src/__tests__/provider-codegen.test.ts`
- `src/__tests__/provider-pipeline-integration.test.ts`

### Validation performed

- **Full provider suite green: 473 tests passing / 4 skipped (34 files)** — up from the Phase II baseline of 440, i.e. +33 new tests, **0 regressions** in the other 14 providers.
- `check:generated` clean — the catalog matches the leaves on disk after `generate:providers`.
- Manual/automated split: I did not stand up a live Azure resource (BYOK, so no shared test credentials), so the transport is verified **automatically** via `vi`-mocked `fetch` that asserts the exact URL and headers Azure would receive. I manually spot-checked the composed URL against Microsoft's REST reference (`/openai/deployments/{deployment}/chat/completions?api-version=2024-10-21`) to confirm the path and default api-version are correct.

---

## Implementation Progress (Phase III — Build)

**Branch:** `feat/azure-openai-provider-304` (in my fork `skonda29/nous-core`, based on
`feat/contributor-friendly-inference-provider-surface`), pushed 2026-07-09.

I landed the work as **three scoped commits**, each doing one thing, in the order a reviewer can follow:

| Commit | Message | What it does |
|--------|---------|--------------|
| `d10f1379` | `providers: support custom auth header scheme in ChatCompletionsProvider` | Central, backward-compatible change: adds `authHeaderName` / `authHeaderScheme` options and a private `buildAuthHeader()` helper used by both `invoke()` and `stream()`. Defaults stay `Authorization` / `bearer`, so the other 14 leaves are untouched. |
| `544e9c0d` | `providers: add Azure OpenAI provider leaf (#304)` | The `azure-openai` leaf (`definition.ts`, `adapter.ts`, `provider.ts`, `index.ts`), regenerated catalogs, and the Azure test files. |
| `9e8dd7ea` | `providers: register azure-openai in shared vendor-roster tests` | Updates the five hard-coded roster tests to include `azure-openai`. |

The commit messages are descriptive (no `wip`/`fix`/`asdf`), and the diff is scoped to the issue — `git diff --stat` since the Phase II base is **16 files, +685 / −10**, all under `self/subcortex/providers/`, with no unrelated formatting churn or commented-out code.

**Files created:**

- `self/subcortex/providers/src/providers/azure-openai/definition.ts` — leaf metadata: `vendorKey: 'azure-openai'`, `api-key`/`raw` auth via `AZURE_OPENAI_API_KEY`, placeholder endpoint/deployment the user must replace, no `modelListEndpoint` (BYOK — no discovery surface), no hand-authored `wellKnownProviderId`. The descope (Foundry / Entra ID / quota / #421) is documented in the file's header comment.
- `self/subcortex/providers/src/providers/azure-openai/provider.ts` — factory that resolves `AZURE_OPENAI_API_KEY` explicitly (fail-closed, no `OPENAI_API_KEY` fallback), composes the deployment path via the exported `buildAzureCompletionsPath(deployment, apiVersion)`, and reads the api-version from `AZURE_OPENAI_API_VERSION` (default GA `2024-10-21`).
- `self/subcortex/providers/src/providers/azure-openai/adapter.ts`, `index.ts` — re-export the shared chat-completions adapter (mirror OpenAI/Groq).
- `self/subcortex/providers/src/__tests__/azure-openai-provider.test.ts` (302 lines) and `src/__tests__/providers/azure-openai.test.ts` (102 lines).

**Files modified:**

- `self/subcortex/providers/src/protocols/openai-api/provider.ts` — the only shared-code change (+25 / −10); the `authHeader*` options + helper described above.
- `self/subcortex/providers/src/__tests__/chat-completions-provider.test.ts` — tests for the new auth-header behavior.
- `self/subcortex/providers/src/provider-definitions.ts`, `provider-adapters.ts`, `provider-factories.ts` — regenerated by `generate:providers` (not hand-edited).
- `self/subcortex/providers/src/__tests__/adapter-resolver.test.ts`, `provider-codegen.test.ts`, `provider-definitions/provider-definitions.test.ts`, `provider-definitions/provider-definition-types.test.ts`, `provider-pipeline-integration.test.ts` — roster pins.

### Plan-vs-actual reconciliation

My Phase II plan proposed templating the deployment URL *inside* the shared provider (a `completionsPathTemplate` + `queryParams` option). During implementation I found that was unnecessary: the shared `ChatCompletionsProvider` already treats `completionsPath` as an **opaque literal suffix** (the `perplexity` leaf sets it statically), so Azure can compose the full `/openai/deployments/{deployment}/chat/completions?api-version=…` string in its own factory and pass it straight through. That kept the shared-code change down to *only* the auth header — the one thing that genuinely couldn't be done from the leaf. Smaller blast radius, same result. This is the "flag protocol-boundary awkwardness rather than work around it heavily" guidance in practice: the header truly needed central support, the URL did not.

### Challenges Faced

- **Credential-boundary safety (the one that mattered).** The shared provider falls back to `process.env.OPENAI_API_KEY` when no key is supplied. For a BYOK Azure connector that's a real hazard — it could send an OpenAI credential to the user's Azure resource. **Resolution:** the factory resolves `AZURE_OPENAI_API_KEY` explicitly and throws `PROVIDER_AUTH_FAILED` if it's missing, so the OpenAI fallback path is never reachable (modeled on `perplexity`'s fail-closed pattern), and there's a dedicated test asserting the fallback never happens.
- **Deployment-name-vs-model-name.** Azure routes by *deployment name*, not model family, and the deployment name is only known once a config exists. **Resolution:** treat `config.modelId` as the deployment name (per the narrowed scope) and compose the path per-instance in the factory rather than as a static `completionsPath`; the deployment segment is URL-encoded, with a test for special characters.
- **`api-version` is mandatory and drifts.** Azure 400s without it, and supported versions vary per resource. **Resolution:** default to the GA `2024-10-21` but allow `AZURE_OPENAI_API_VERSION` to override, with tests for both.
- **Hard-coded rosters (known from vLLM).** Adding a vendor breaks a compile-time exact-union test and four runtime roster tests. **Resolution:** I updated all five as part of the commit sequence instead of discovering them at the end like I did on Contribution 1.

---

## Pull Request

**Status:** Branch pushed to my fork ([`skonda29/nous-core:feat/azure-openai-provider-304`](https://github.com/skonda29/nous-core/tree/feat/azure-openai-provider-304)); PR to `orthogonalhq/nous-core` to be opened against the integration branch.

**PR target:** `feat/contributor-friendly-inference-provider-surface` (same integration branch as the accepted Groq [#404], llama.cpp [#403], and vLLM [#417] leaves).

**PR summary (draft):** Adds Azure OpenAI as a certified provider leaf under the narrowed BYOK scope agreed on #304. The only shared-code change is a backward-compatible auth-header option on `ChatCompletionsProvider` (`api-key`/raw vs. the default `Authorization`/bearer); the Azure deployment URL is composed in the leaf factory using the already-opaque `completionsPath`. The leaf is fail-closed on credentials (no `OPENAI_API_KEY` fallback), derives its id from `vendorKey`, regenerates catalogs via the generator, and leaves `IModelProvider` / `TextModelInputSchema` unchanged. 473 provider tests pass. The PR description will explicitly flag the auth-header protocol-boundary change and note the descope to #421, per the maintainer's request.

---

## Maintainer Feedback Log

| Date | Source | Feedback / Guidance | My Response | Commit / Ref |
|------|--------|---------------------|-------------|--------------|
| 2026-06-18 | @atlamors on #304 (issue body) | Implement as a certified provider leaf under `providers/<vendor>/`; target `feat/contributor-friendly-inference-provider-surface`; leaves use `ProviderDefinitionLeaf` with ids derived from `vendorKey` (don't hand-author `wellKnownProviderId`); the direct-`IModelProvider` path is superseded. | Plan follows the leaf contract and the integration-branch target; id will derive from `vendorKey`. | — |
| 2026-06-28 | — | Requested to take up the issue. | Awaiting assignment; started Phase II reproduction in parallel. | issue #304 comment |
| 2026-06-30 | provider-surface refactor (`git blame` on `provider.ts:97`, `provider-definition.ts:34`) | The same refactor that introduced the `raw` auth-scheme in the contract also left the shared provider hard-coding bearer. The contract is ahead of the runtime. | Confirms the central fix ("shared provider reads `auth.header`") is the intended direction, not a workaround. | `provider.ts:97`, `provider-definition.ts:34` |
| 2026-07-08 | @atlamors on #304 | Narrowed the issue to **Azure OpenAI as a direct BYOK connector only** — user supplies endpoint + key + deployment name (used as the model value). Explicitly excluded Azure AI Foundry, deployment creation, Entra ID / managed identity, quota/capacity discovery, regional routing, provider fallback, and managed Nue Cloud billing (all belong to #421). Asked me to flag protocol-boundary awkwardness in the PR rather than working around it. Assigned me the issue. | Implemented exactly this scope: BYOK-only leaf, `config.modelId` = deployment name, descope documented in `definition.ts`, and only the auth header (a genuine protocol-boundary gap) changed centrally — the URL was composed in the leaf. | commits `d10f1379`, `544e9c0d`, `9e8dd7ea` |

> This log will be updated with line-level review comments and my responses (with commit refs) once the PR is opened and reviewed.

---

## Process & Communication

- **2026-06-28** — Commented on #304 requesting to take it up, referencing my prior vLLM work (#317 / PR #417).
- **2026-07-08** — @atlamors narrowed the scope to a BYOK connector and **assigned me** the issue.
- **2026-07-09** — Implemented Phase III on `feat/azure-openai-provider-304`, ran the full provider suite green (473 passing), and pushed the branch to my fork.
- **Check-in form:** submitted with **"Phase III Complete"** marked.
- **Slack:** posted in the cohort channel within the 7 days before submitting — shared that I'd been assigned #304 under the narrowed BYOK scope, noted the fail-closed credential-boundary decision I reused from the `perplexity` leaf, and answered a peer's question about where the hard-coded provider rosters live (the compile-time exact-union test vs. the runtime roster tests) since I'd just hit the same thing on vLLM.
- **Next:** open the PR against the integration branch with the auth-header protocol-boundary change and the #421 descope both called out explicitly.

---

## Engineering Judgment Beyond the Minimum

Things I did that went past "make it work":

- **Descoped sensibly, with a documented note.** The maintainer narrowed the issue to BYOK; I didn't just silently omit the managed-platform features — I wrote the exclusion (Foundry, Entra ID, quota, regional routing, #421) directly into `definition.ts`'s header comment and into the Maintainer Feedback Log, so the next reader knows the boundary was deliberate and where the rest of the work lives.
- **Identified a credential-leak edge case the issue didn't mention.** The shared provider's silent `OPENAI_API_KEY` fallback would let an OpenAI key be sent to a user's Azure endpoint. Nobody flagged this in the issue; I found it, made the leaf fail closed, and added a test that asserts the fallback never fires.
- **Shrank my own proposed shared-code change.** My Phase II plan wanted URL templating in the shared provider too. On implementation I realized `completionsPath` is already an opaque suffix, so I pulled the URL composition back into the leaf and changed *only* the auth header centrally — the smallest possible blast radius, which is exactly what the maintainer asked for ("flag it rather than work around it heavily" cuts both ways: don't over-touch shared code either).
- **Reused project-specific test patterns.** I exported `buildAzureCompletionsPath` specifically so it could be unit-tested directly, mirrored the `perplexity`/`chat-completions` test files' `vi`-mocked `fetch` style, and added a regression test on the shared provider proving the *default* bearer behavior still holds for the other 14 leaves — not just that Azure works.
- **Handled real-world Azure quirks proactively.** URL-encoding deployment names, making `api-version` overridable because it drifts per resource, and pinning a known-GA default — all with tests — rather than hard-coding a single happy-path string.
- **Helped a peer in Slack** by pointing them at the two different hard-coded roster locations (compile-time exact-union vs. runtime lists) that trip up everyone adding a provider.

---

## Learnings & Reflections

### Technical Skills Gained

The biggest lesson from this phase is the value of reading `git blame` before writing code. On Contribution 1 I went straight to implementation, had to reconcile onto the integration branch mid-stream, and discovered the `'no-auth'` placeholder pattern only by reading a merged PR. This time I ran `git blame` on the files that matter *before* planning, and it immediately explained why the `raw` auth scheme exists in the schema (it was added by the same refactor that set up the provider-leaf surface) and why the shared provider doesn't consume it yet (it was simply the next step that hadn't been done). That's a much faster path to understanding intent than reverse-engineering from code alone.

I also got clearer on the difference between a *workaround* and a *central fix*. On vLLM, the `'no-auth'` placeholder was an accepted workaround because modifying the shared provider for a boolean `requireApiKey` flag wasn't worth the risk. On Azure, the `api-key` header and deployment URL are not edge cases — they're a fundamentally different transport for what is otherwise the same wire protocol, and the schema already models the fix. That's the kind of thing worth extending the shared provider for.

### Challenges Overcome

The two concrete obstacles I found (hard-coded Bearer at `provider.ts:97,159`; static path at `92,154`) were clear, but understanding *why* they're obstacles — rather than just *that* they're obstacles — took some investigation. Reading the schema, the Anthropic provider, and the `git blame` output together gave me a coherent picture: the primitives are modeled, one accepted example exists (Anthropic), and the contract is ahead of the runtime by design. That's very different from "the code is wrong." It changes what a good fix looks like.

The other challenge was already knowing, from Contribution 1, that adding a vendor touches more places than just the leaf directory. vLLM forced me to update five roster-pinning tests after the fact; this time I'll plan those updates as part of the initial commit sequence rather than treating them as surprises.

### What I'd Do Differently Next Time

On Contribution 1, I built against an older branch and reconciled at the end. On Contribution 2, I went straight to the integration branch and ran `git blame` before planning. That's the right sequence. For Phase III, the one thing I want to do differently is post the design question on the issue *before writing any code* — even before starting the implementation branch — to avoid the risk of building the wrong thing and having to undo it.

### Teachable Insight (for future cohorts)

**`git blame` is a requirements document.** If a schema models something that the runtime doesn't yet consume, that gap is almost always intentional: the schema was written with the future in mind, and the runtime just hasn't caught up yet. Before deciding whether to extend shared code or write a workaround, look at when each piece was written and by whom. A schema field added in the same refactor that introduced the current leaf contract is a strong signal that the maintainer already designed the right extension point — you just have to implement it.

---

## Resources Used

- Azure OpenAI issue #304: https://github.com/orthogonalhq/nous-core/issues/304
- Provider adapter documentation: https://docs.nue.orthg.nl/docs/development/provider-adapters/quickstart
- Provider leaf anatomy: https://docs.nue.orthg.nl/docs/development/provider-adapters/provider-leaf-anatomy
- Schemas / ABI reference: https://docs.nue.orthg.nl/docs/development/provider-adapters/schemas-abi-reference
- Testing checklist: https://docs.nue.orthg.nl/docs/development/provider-adapters/testing-checklist
- Reference leaves I studied for this phase:
  - `self/subcortex/providers/src/providers/groq/` and `deepinfra/` (keyed OpenAI-compatible leaf skeleton)
  - `self/subcortex/providers/src/providers/perplexity/` (fail-closed key resolution + `completionsPath` override)
  - `self/subcortex/providers/src/providers/anthropic/implementation.ts` (`api-key`/`raw` header + static version header — accepted precedent)
  - My own vLLM leaf on the integration branch (full submission workflow reference — PR [#417](https://github.com/orthogonalhq/nous-core/pull/417))
- Merged precedent PRs I reviewed:
  - [#403 — llama.cpp provider leaf](https://github.com/orthogonalhq/nous-core/pull/403)
  - [#404 — Groq model provider leaf](https://github.com/orthogonalhq/nous-core/pull/404)
- Azure OpenAI REST reference (auth header + `api-version` + deployment URL):
  https://learn.microsoft.com/azure/ai-services/openai/reference
