# Contribution 2: Adapter: Azure OpenAI Model Provider

**Contribution Number:** 2

**Student:** Srinitya Kondapally (@skonda29)

**Issue:** https://github.com/orthogonalhq/nous-core/issues/304

**Status:** Phase II Complete — Reproduced the gap and wrote the implementation plan (2026-07-06)

**Working branch:** `feat/azure-openai-provider-304` (in my fork `skonda29/nous-core`,
cut from the integration branch `feat/contributor-friendly-inference-provider-surface`)

---

## Why I Chose This Issue

After finishing Contribution 1 (the vLLM provider leaf, #317 / PR #417) I already
understand how nous-core structures provider integrations as certified leaves,
how the generated catalogs are produced, and how a leaf reuses the shared
`protocols/openai-api/` chat-completions protocol. Issue #304 — **Azure OpenAI
Model Provider** — is the natural next step: it's the same provider-leaf surface,
but Azure OpenAI is *not* a drop-in OpenAI clone, so it stretches what I learned
on vLLM.

The interesting engineering wrinkle is that Azure OpenAI speaks the same
chat-completions *payload* shape as OpenAI but differs in two ways that the
shared `ChatCompletionsProvider` does not currently handle:

1. **Auth header.** Azure uses an `api-key: <key>` request header, not
   `Authorization: Bearer <key>`.
2. **URL shape.** Azure routes by *deployment*:
   `{endpoint}/openai/deployments/{deployment}/chat/completions?api-version=<ver>`,
   rather than `{base}/v1/chat/completions`.

On vLLM I learned that the maintainer prefers protocol-level gaps to be fixed
*centrally* in the shared protocol rather than worked around per leaf. Azure is a
clean, real example of that, so it's a good issue to apply that lesson to.

---

## Environment Setup

### Approach

I reused the environment I already stood up for Contribution 1 (vLLM), so this
phase was mostly about pointing it at the right branch rather than a fresh
bootstrap. My setup approach for nous-core is **README instructions +
package-script inspection** (there is no dev container in the repo); the provider
package documents its own workflow through the scripts in
`self/subcortex/providers/package.json`:

- `generate:providers` → `node scripts/generate-provider-aggregates.mjs`
- `check:generated` → same script with `--check` (fails if the generated catalog
  is stale)
- `build` / `typecheck` → `check:generated` then `tsc --build`
- tests run through Vitest (`vitest run --pool threads` at the repo root, with a
  package-level `self/subcortex/providers/vitest.config.ts`)

Steps I ran to get to a working, testable state:

1. `git fetch origin feat/contributor-friendly-inference-provider-surface` to pull
   the current integration target.
2. `git checkout -B feat/azure-openai-provider-304 origin/feat/contributor-friendly-inference-provider-surface`
   — this is my issue-named working branch.
3. `node scripts/generate-provider-aggregates.mjs --check` (from
   `self/subcortex/providers/`) — confirms the checked-in generated catalog
   matches the leaves on disk.
4. `npx vitest run self/subcortex/providers/src/__tests__` — establishes the
   green baseline I'll build on.

### Baseline (before any of my changes)

- `check:generated` → exits 0 (catalog is in sync; no Azure leaf is expected yet).
- Provider test suite → **32 files passed / 2 skipped**, **440 tests passed / 4
  skipped**, ~3s. This is my "known good" starting point.

### Challenges encountered and how I resolved them

- **Node version drift.** On Contribution 1, Node 24 broke `pnpm install`
  (a `crypto::Hash::OneShotDigest` OOM during native rebuilds), so I had pinned
  Node 22 for that phase. For Phase II I only needed to switch branches and run
  the already-installed toolchain, and the Vitest suite runs cleanly on the
  currently-installed Node 24 (v24.4.1) + pnpm 10.6.2 against the existing
  `node_modules`. **Resolution:** don't re-run `pnpm install` this phase; reuse
  the working install from Contribution 1. If I hit a native-module failure when
  I do need to reinstall for Phase III, I'll fall back to Node 22 exactly as
  before.
- **Sandbox loses `node_modules` between shell calls.** Running commands in
  isolation intermittently dropped the workspace `node_modules`. **Resolution:**
  chain the setup + command in a single invocation (and run unsandboxed when a
  step legitimately needs the full install), so the dependency tree is present
  for the command that needs it.
- **The integration branch, not `main`, is the source of truth.** `main` still
  has the older direct-`IModelProvider` layout. **Resolution:** I cut my branch
  from `feat/contributor-friendly-inference-provider-surface` and did all
  reproduction there, so everything below reflects the contract I'll actually
  submit against.
- **The branch already contains my vLLM leaf.** `providers/vllm/` is present on
  the integration branch (from #417), which confirms my Contribution 1 work
  landed on the contract I'm targeting and gives me a first-party precedent to
  mirror.

---

## Understanding the Issue

### Problem Description

nous-core supports many model providers through the certified provider-leaf
system. On the integration branch the roster is: `anthropic`, `codex-cli`,
`deepinfra`, `github-copilot-cli`, `groq`, `huggingface-tgi`, `llama-cpp`,
`moonshot`, `ollama`, `openai`, `openclaw`, `openrouter`, `perplexity`, and
`vllm` (my Contribution 1). It does **not** include **Azure OpenAI**, which is a
common deployment target for teams that consume OpenAI models through Azure's
hosting, compliance, and regional guarantees.

The issue asks for an Azure OpenAI provider implemented against the **current**
provider-leaf contract (`ProviderDefinitionLeaf`, ids derived from `vendorKey`,
generated catalogs). Integration target:
`feat/contributor-friendly-inference-provider-surface`.

### Specific files / functions involved

- `self/subcortex/providers/src/protocols/openai-api/provider.ts` — the shared
  `ChatCompletionsProvider`. This is where the real gap lives:
  - `invoke()` sets `Authorization: Bearer ${this.apiKey}` (line 97) and builds
    the URL as `` `${endpoint}${this.completionsPath}` `` (line 92).
  - `stream()` does the same (`Authorization: Bearer` at line 159, URL at
    line 154).
  - the constructor only accepts `{ apiKey, timeoutMs, completionsPath }`
    (lines 62–73); `completionsPath` is a **static** string
    (`DEFAULT_COMPLETIONS_PATH = '/v1/chat/completions'`, line 24) with no query
    string and no per-request templating.
- `self/subcortex/providers/src/schemas/provider-definition.ts` — the contract.
  Notably it **already** models the primitives Azure needs but the shared
  provider ignores them:
  - `ProviderAuthHeaderSchemeSchema = z.enum(['raw', 'bearer'])` (line 34) —
    `raw` means "put the credential in the named header verbatim" (i.e.
    `api-key: <key>`).
  - top-level optional `headers: z.record(...)` (line 157) — static per-request
    headers (Anthropic uses this for `anthropic-version`).
- `self/subcortex/providers/src/providers/anthropic/implementation.ts` — the
  proof that the codebase already knows how to do raw-header auth: it declares
  `header: { name: 'x-api-key', scheme: 'raw' }` and actually sends
  `'x-api-key': this.apiKey` (line 319). Anthropic gets away with a bespoke
  provider because it's a *different* wire protocol; Azure can't, because it must
  reuse the OpenAI chat-completions wire format.
- `self/subcortex/providers/src/provider-identity.ts` — `deriveBuiltInProviderId(vendorKey)`
  derives the provider id from `vendorKey` (v5 UUID). `azure-openai` is not in
  `LEGACY_BUILT_IN_PROVIDER_IDS`, so its id derives cleanly with **no
  hand-authored `wellKnownProviderId`** — the exact lesson from #417.
- Generated catalogs (do not hand-edit): `provider-definitions.ts`,
  `provider-adapters.ts`, `provider-factories.ts`, `adapter-resolver.ts`,
  regenerated by `scripts/generate-provider-aggregates.mjs`.

### Expected vs. Actual Behavior

| | Expected (once #304 is done) | Actual (today, integration branch) |
|---|---|---|
| Registration | `azure-openai` appears in `PROVIDER_DEFINITIONS` and resolves via `resolveProviderDefinition('azure-openai')` | No `azure-openai` leaf exists; the `ProviderVendorKey` union has no such member, and resolving it throws `Provider definition is missing for vendor key 'azure-openai'` |
| Auth header | Request carries `api-key: <AZURE_OPENAI_API_KEY>` | The only chat-completions path hard-codes `Authorization: Bearer <key>` (`provider.ts:97,159`) — Azure would reject this |
| URL shape | `POST {endpoint}/openai/deployments/{deployment}/chat/completions?api-version=<ver>` | URL is `{endpoint}{completionsPath}` with a static path and **no query string**, so `api-version` can't be attached and the deployment segment can't be templated (`provider.ts:92,154`) |
| Provider id | Derived from `vendorKey` via `deriveBuiltInProviderId` | N/A (no leaf) |

---

## Reproduction (numbered, no prior context needed)

Run from the repo root of a clone of `orthogonalhq/nous-core`.

1. **Get on the integration branch:**
   ```bash
   git fetch origin feat/contributor-friendly-inference-provider-surface
   git checkout -B repro/azure-check origin/feat/contributor-friendly-inference-provider-surface
   ```
2. **Confirm there is no Azure leaf on disk:**
   ```bash
   ls self/subcortex/providers/src/providers/
   ```
   *Actual:* the directory lists 14 vendors (anthropic … vllm) and **no
   `azure-openai`**.
3. **Confirm Azure appears nowhere in the provider package:**
   ```bash
   grep -rin "azure" self/subcortex/providers/src
   ```
   *Actual:* no matches.
4. **Confirm the generated catalog is in sync (i.e. Azure genuinely isn't
   expected yet):**
   ```bash
   cd self/subcortex/providers && node scripts/generate-provider-aggregates.mjs --check
   ```
   *Actual:* exits 0 with no diff → the checked-in catalog matches disk, so the
   absence is real, not a stale-generation artifact.
5. **See the auth-header obstacle** — open
   `self/subcortex/providers/src/protocols/openai-api/provider.ts` and look at
   lines 92–98 (`invoke`) and 154–160 (`stream`):
   *Actual:* both hard-code `Authorization: Bearer ${this.apiKey}` and build the
   URL as `` `${endpoint}${this.completionsPath}` ``. There is no branch that
   reads the leaf's declared `auth.header`.
6. **See the URL-shape obstacle** — the constructor's only URL knob is
   `completionsPath` (lines 24, 64, 73), a static string defaulting to
   `/v1/chat/completions`. There is no place to attach the `?api-version=` query
   parameter or to interpolate the Azure `deployment` segment per request.
7. **See that the contract already anticipates the fix** — open
   `self/subcortex/providers/src/schemas/provider-definition.ts` line 34:
   `ProviderAuthHeaderSchemeSchema = z.enum(['raw', 'bearer'])`. The `raw` scheme
   (used by Anthropic's `x-api-key`) is exactly what Azure needs, but the shared
   OpenAI-compatible provider never consumes it.

**Conclusion of reproduction:** the surface symptom ("no Azure provider") sits on
top of a concrete central limitation — the shared `ChatCompletionsProvider`
neither honors a leaf's declared `auth.header` nor supports a query string /
templated deployment in its request URL.

---

## Solution Plan (UMPIRE)

### U — Understand

Azure OpenAI is wire-compatible with OpenAI Chat Completions for the request and
response *bodies*, but differs in transport in exactly two ways: the credential
goes in an `api-key` header (not `Authorization: Bearer`), and the endpoint is
`{resource-endpoint}/openai/deployments/{deployment}/chat/completions?api-version=<ver>`.
The provider must be a certified leaf under the current contract, reuse the
shared chat-completions protocol, and derive its id from `vendorKey`. Success =
an `azure-openai` leaf that (a) sends the right header and URL, (b) round-trips
invoke + stream, (c) maps 401/429/other errors like the other leaves, and (d)
regenerates cleanly with `check:generated` and the full provider suite green.

### M — Match (analogous patterns already in the tree)

- **Groq / DeepInfra leaf shape** — keyed, OpenAI-compatible, custom
  `defaultEndpoint`, reuse `ChatCompletionsProvider` via a 6-line factory. This is
  the skeleton for the Azure leaf's four files (`definition.ts`, `adapter.ts`,
  `provider.ts`, `index.ts`).
- **Perplexity factory** — precedent for a leaf overriding transport details of
  the shared provider (`completionsPath: '/chat/completions'`) and for *failing
  closed* on the OpenAI env-var fallback by resolving its own key explicitly. I'll
  copy that credential-boundary safety for `AZURE_OPENAI_API_KEY`.
- **Anthropic `implementation.ts`** — precedent for `header: { name, scheme: 'raw' }`
  and for static `headers`. It proves the raw-header idea is accepted; my job is
  to make the *shared* provider honor it instead of duplicating a whole provider.
- **My own vLLM leaf (#417)** — first-party precedent on this exact branch for
  adding a leaf, regenerating catalogs, and updating roster-pinning tests without
  hand-authoring an id.

### P — Plan (root cause + specific files to modify)

**Root cause (not symptom):** the shared `ChatCompletionsProvider`
(`protocols/openai-api/provider.ts`) hard-codes `Authorization: Bearer` and can
only build `endpoint + static path` URLs; it ignores the leaf-declared
`auth.header` and has no notion of query params or a templated deployment
segment. The missing Azure leaf is just the visible symptom.

Per the maintainer's standing "fix protocol gaps centrally" guidance, I'll extend
the shared provider minimally and drive the behavior from the leaf definition:

1. **Central protocol change** —
   `self/subcortex/providers/src/protocols/openai-api/provider.ts`:
   - Add a small `authHeader` option (name + scheme) so `raw` produces
     `<name>: <key>` and `bearer` keeps `Authorization: Bearer <key>` (default,
     so existing leaves are unchanged). Factor the header construction into one
     helper used by both `invoke` and `stream`.
   - Generalize URL building so a leaf can supply the full completions path
     *including* a query string, and can request the Azure deployment template
     (`/openai/deployments/{deployment}/chat/completions`) where `{deployment}`
     comes from `config.modelId`. Keep the current default untouched.
   - Merge in static `headers` from the definition (mirrors how Anthropic sends
     `anthropic-version`).
2. **New leaf** — `self/subcortex/providers/src/providers/azure-openai/` with
   `definition.ts` (vendorKey `azure-openai`, `displayName` "Azure OpenAI",
   `protocol`/`adapterKey` `chat-completions`, `auth.header { name: 'api-key',
   scheme: 'raw' }`, `envVar: 'AZURE_OPENAI_API_KEY'`, `isLocal: false`, **no**
   `wellKnownProviderId`), `provider.ts` (factory that resolves the Azure key
   explicitly — fail-closed like Perplexity — and wires the deployment/api-version
   options), `adapter.ts` and `index.ts` re-exporting the shared adapter (mirror
   OpenAI/Groq).
3. **Regenerate catalogs** — `node scripts/generate-provider-aggregates.mjs`
   (updates `provider-definitions.ts`, `provider-adapters.ts`,
   `provider-factories.ts`, `adapter-resolver.ts`). Never hand-edit generated
   files.
4. **Tests** — add `self/subcortex/providers/src/__tests__/azure-openai-provider.test.ts`
   (definition metadata, `api-key` header presence + no `Authorization`, deployment
   + `api-version` URL construction, invoke happy path, streaming, 401/429/error
   mapping), and update the roster-pinning tests that enumerate all vendors
   (`adapter-resolver.test.ts`, the `provider-definitions/` and codegen tests) —
   exactly the set I had to touch for vLLM.

### I — Implement

Land it in reviewable commits mirroring my vLLM PR:
(1) central `ChatCompletionsProvider` header/URL extension + its unit tests,
(2) the `azure-openai` leaf + regenerated catalogs,
(3) the Azure provider tests,
(4) roster-test alignment. I'll open the PR against
`feat/contributor-friendly-inference-provider-surface` and, before writing the
central change, comment on #304 to confirm the maintainer wants the header/URL
generalization in the shared provider (vs. a bespoke Azure provider), since that's
the one design fork with real trade-offs.

### R — Review

Self-review against the provider testing checklist and the ABI/schema reference;
run `check:generated`, `typecheck`, `lint`, and the full provider suite; diff my
leaf against Groq/DeepInfra to make sure I only added Azure-specific metadata; and
confirm no existing leaf's behavior changed (bearer stays the default).

### E — Evaluate

Done = green suite (baseline 440→ +new Azure tests, still 0 failures),
`check:generated` clean, the `api-key`/deployment/`api-version` behavior asserted
by tests, no regression in the other 14 providers, and PR opened against the
integration branch with the design question raised up front.

---

## Investigative Depth Notes

- **git blame on the transport gap.** `git blame` on the integration branch
  attributes both the hard-coded `Authorization: Bearer` (`provider.ts:97`) and
  the `['raw','bearer']` auth-scheme enum (`provider-definition.ts:34`) to the
  provider-surface refactor (Andrew Nelson, 2026-06-30). The same change that
  introduced a *`raw`* header scheme left the shared provider hard-coding bearer —
  so the contract is ahead of the runtime. That's strong evidence the intended
  central design is "shared provider reads `auth.header`," which is exactly the
  fix I'm proposing rather than a workaround.
- **Analogous pattern found.** Anthropic's `implementation.ts` already ships
  `scheme: 'raw'` + `'x-api-key': this.apiKey` and static
  `headers: { 'anthropic-version': ... }`. I'm not inventing a mechanism; I'm
  lifting an accepted one into the shared OpenAI-compatible path so Azure can
  reuse it without a bespoke provider.
- **Edge cases I've already flagged for implementation:**
  - **Credential leakage / fallback.** The shared provider falls back to
    `process.env.OPENAI_API_KEY` when no key is passed. For Azure I'll resolve
    `AZURE_OPENAI_API_KEY` explicitly and fail closed (Perplexity precedent), so
    an OpenAI key can never be sent to an Azure endpoint.
  - **Deployment vs. model id.** Azure's URL segment is the *deployment* name,
    which usually — but not always — equals the model id. I'll source it from
    `config.modelId` and document that assumption in the leaf.
  - **`api-version` is required.** Azure 400s without it; I'll give the leaf a
    sensible default `api-version` and assert it lands in the URL query.
  - **Endpoint normalization.** The shared provider strips a trailing slash
    before appending the path; I'll keep that and make sure it composes correctly
    with the deployment template + query string.
  - **Roster pins.** vLLM taught me that several tests hard-code the full vendor
    roster (compile-time and runtime). I already know which tests fail if I add a
    vendor without updating them, so that won't surprise me this time.
- **Productive escalation planned.** The single genuine design decision — put the
  header/URL generalization in the shared provider vs. write a bespoke Azure
  provider — is exactly the kind of thing the maintainer asked to be raised
  *before* implementation on #317. I'll post that question on #304 with the two
  options and my recommendation (central), rather than guessing.

---

## Process & Communication

- **2026-06-28** — Commented on #304 requesting to take it up, referencing my
  prior work on the vLLM provider (#317 / PR #417):
  > "Hi @atlamors, I previously worked on Adapter: vLLM Model Provider #317 and
  > would like to take up this issue and get started."
- **2026-07-06** — Completed Phase II: cut the issue-named working branch off the
  integration branch, reproduced the gap, and wrote the UMPIRE plan above.
- **Check-in form:** submitted with **"Phase II Complete"** marked.
- **Next:** post the "central vs. bespoke" design question on #304, then begin
  Phase III (implementation) once acknowledged.

---

## Maintainer Feedback Log

| Date | Source | Feedback / Guidance | My Response | Commit / Ref |
|------|--------|---------------------|-------------|--------------|
| 2026-06-18 | @atlamors on #304 (issue body) | Implement as a certified provider leaf under `providers/<vendor>/`; target `feat/contributor-friendly-inference-provider-surface`; leaves use `ProviderDefinitionLeaf` with ids derived from `vendorKey` (don't hand-author `wellKnownProviderId`); the direct-`IModelProvider` path is superseded. | Plan follows the leaf contract and the integration-branch target; id will derive from `vendorKey`. | — |
| 2026-06-28 | — | Requested to take up the issue. | Awaiting assignment; started Phase II reproduction in parallel. | issue #304 comment |
| 2026-06-30 | provider-surface refactor (git blame) | Shared `ChatCompletionsProvider` hard-codes bearer while the schema adds a `raw` header scheme. | Confirms central header/URL handling is the intended home for the fix. | `provider.ts:97`, `provider-definition.ts:34` |

---

## Resources

- Azure OpenAI issue #304: https://github.com/orthogonalhq/nous-core/issues/304
- Provider adapter docs: https://docs.nue.orthg.nl/docs/development/provider-adapters/quickstart
- Provider leaf anatomy: https://docs.nue.orthg.nl/docs/development/provider-adapters/provider-leaf-anatomy
- Schemas / ABI reference: https://docs.nue.orthg.nl/docs/development/provider-adapters/schemas-abi-reference
- Testing checklist: https://docs.nue.orthg.nl/docs/development/provider-adapters/testing-checklist
- Precedent leaves I'm modeling on:
  - [#404 — Groq model provider leaf](https://github.com/orthogonalhq/nous-core/pull/404) (keyed OpenAI-compatible)
  - [#403 — llama.cpp provider leaf](https://github.com/orthogonalhq/nous-core/pull/403)
  - Perplexity leaf (fail-closed key resolution + `completionsPath` override)
  - Anthropic `implementation.ts` (`api-key`/`raw` header + static version header)
  - My own vLLM leaf: [#417](https://github.com/orthogonalhq/nous-core/pull/417)
- Azure OpenAI REST reference (auth header + `api-version` + deployment URL):
  https://learn.microsoft.com/azure/ai-services/openai/reference
