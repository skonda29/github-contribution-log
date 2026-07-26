# Contribution 2: Adapter: Azure OpenAI Model Provider

**Contribution Number:** 2

**Student:** Srinitya Kondapally (@skonda29)

**Issue:** https://github.com/orthogonalhq/nous-core/issues/304

**Status:** Phase IV — PR [#425](https://github.com/orthogonalhq/nous-core/pull/425) open; addressed maintainer change request (endpoint preservation), awaiting re-review

**Other contributions:** [Contribution 1 — vLLM Model Provider (#317)](README.md) (Phase IV Complete — PR [#417](https://github.com/orthogonalhq/nous-core/pull/417) open)

---

## Why I Chose This Issue

I picked this one because it's a direct follow-up to my first contribution. After doing the vLLM leaf (#317 / PR #417), I already knew how nous-core organizes providers as "leaves," how the generated catalogs work, and how a leaf reuses the shared `protocols/openai-api/` chat-completions code. So #304 felt like a good next step — same general area, but a harder problem.

The reason it's harder is that Azure OpenAI isn't just OpenAI with a different URL. The request/response JSON is the same, but two things are different:

1. Azure sends the key in an `api-key:` header instead of `Authorization: Bearer`.
2. Azure's URL is deployment-based: `{endpoint}/openai/deployments/{deployment}/chat/completions?api-version=<ver>`, not `/v1/chat/completions`.

From vLLM I also learned the maintainer would rather fix protocol-level gaps in the shared code than have every leaf hack around them, so I figured Azure would be a good place to actually apply that.

---

## Understanding the Issue

### Problem Description

nous-core already supports a bunch of providers as leaves. On the integration branch (`feat/contributor-friendly-inference-provider-surface`) the list is: `anthropic`, `codex-cli`, `deepinfra`, `github-copilot-cli`, `groq`, `huggingface-tgi`, `llama-cpp`, `moonshot`, `ollama`, `openai`, `openclaw`, `openrouter`, `perplexity`, and `vllm` (mine). There's no Azure OpenAI, which is a pretty common way for companies to use OpenAI models (through Azure's hosting/compliance/regions).

The issue wants an Azure OpenAI provider built the current way — a `ProviderDefinitionLeaf`, id derived from `vendorKey`, catalogs updated by the generator. Target branch: `feat/contributor-friendly-inference-provider-surface`.

### Scope (the maintainer narrowed it on 2026-07-08)

Before I started coding, @atlamors looked deeper at the Azure side and narrowed the issue so I'd know exactly what to build. The agreed scope is **Azure OpenAI as a direct "bring your own key" (BYOK) connector only**:

- the user already has an Azure OpenAI resource and a deployment;
- the user gives the endpoint;
- the user gives an API key;
- the model value is treated as the Azure **deployment name**.

He also listed what's explicitly **not** part of this issue (all of it belongs to the managed inference relay work in #421): Azure AI Foundry, creating deployments, Entra ID / managed-identity auth, quota/capacity discovery, regional routing, provider fallback, and managed Nue Cloud billing. And he asked that if the shared OpenAI-compatible code makes the endpoint/path, deployment-vs-model, or api-key header stuff awkward, I should flag it in the PR instead of hacking around it in the leaf. I stuck to this scope and wrote the descope right into `definition.ts` so it's obvious later.

### Expected Behavior

Once this is done:

- there's an `azure-openai` leaf under `self/subcortex/providers/src/providers/azure-openai/`;
- it reuses the OpenAI chat-completions request/response shape;
- it uses the `api-key` header and `AZURE_OPENAI_API_KEY`;
- it builds the deployment-style URL with the required `api-version`;
- the id comes from `vendorKey` (no hand-authored `wellKnownProviderId`);
- catalogs are updated by the generator;
- tests cover the metadata, the header, the URL, invoke/stream, and errors.

### Current Behavior (before my change)

Azure isn't registered. On the integration branch, `resolveProviderDefinition('azure-openai')` throws:

```text
Provider definition is missing for vendor key 'azure-openai'
```

The `ProviderVendorKey` type union doesn't have `azure-openai` either, so TypeScript would reject any reference to it. The closest existing leaves are Groq and DeepInfra (keyed, OpenAI-compatible, custom endpoint), but none of them use the `api-key` header or the Azure URL shape — the shared `ChatCompletionsProvider` only builds `{endpoint}{path}` and always sends `Authorization: Bearer`.

### Affected Components

- `self/subcortex/providers/src/providers/azure-openai/` (new leaf)
- `self/subcortex/providers/src/protocols/openai-api/provider.ts` (shared code — needs the api-key header support)
- generated catalogs (`provider-definitions.ts`, `provider-adapters.ts`, `provider-factories.ts`) — via the generator only
- tests under `self/subcortex/providers/src/__tests__/`

### Reproduction Process

Since this is a "missing feature," reproducing it meant proving Azure really isn't there and finding what would block a working version.

**Environment setup.** I reused the setup from Contribution 1. There's no dev container, so my approach is reading the README + the package scripts in `self/subcortex/providers/package.json`:

- `generate:providers` → runs the catalog generator
- `check:generated` → same thing with `--check` (fails if the catalog is out of date)
- tests via Vitest at the repo root

Steps to get a baseline:

```bash
git fetch origin feat/contributor-friendly-inference-provider-surface
git checkout -B feat/azure-openai-provider-304 \
  origin/feat/contributor-friendly-inference-provider-surface

cd self/subcortex/providers
node scripts/generate-provider-aggregates.mjs --check   # in sync

npx vitest run self/subcortex/providers/src/__tests__
# 32 files / 440 tests passing (my starting point)
```

One thing I remembered from Contribution 1: Node 24 broke `pnpm install` with a `crypto::Hash::OneShotDigest` crash, so I'd pinned Node 22 back then. This phase I didn't need a reinstall — the tests run fine on the existing `node_modules` with Node 24. If I do need to reinstall later, I'll go back to Node 22.

**Steps to prove the gap (anyone can run these):**

1. `ls self/subcortex/providers/src/providers/` → 14 vendors, no `azure-openai`.
2. `grep -rin "azure" self/subcortex/providers/src` → nothing.
3. `node scripts/generate-provider-aggregates.mjs --check` → exits 0, so the catalog matches disk and Azure genuinely isn't there.
4. Open `self/subcortex/providers/src/protocols/openai-api/provider.ts` — `invoke()` and `stream()` both hard-code `Authorization: Bearer ${this.apiKey}`. Nothing reads a per-leaf header.
5. The only URL knob is `completionsPath`, a plain string defaulting to `/v1/chat/completions`.

So the real blocker isn't just "no leaf" — it's that the shared provider always sends a Bearer header, which Azure will reject.

---

## Solution Approach

### Analysis

The missing leaf is the symptom. The actual blocker is the shared `ChatCompletionsProvider`: it always sends `Authorization: Bearer` and doesn't know about a per-leaf header. Azure needs the `api-key` header. The URL part turned out to be less of a problem than I first thought (more on that below).

I didn't want to write a whole separate Azure provider, because the request/response bodies are identical to OpenAI — copying the protocol just to change a header would be the wrong call. So the plan was: make one small, backwards-compatible change to the shared provider for the header, and put everything Azure-specific in the leaf.

References I leaned on: Groq/DeepInfra for the leaf skeleton, Perplexity for resolving the key explicitly (so it can't fall back to an OpenAI key), and my own vLLM leaf for the whole "add leaf, regenerate, fix roster tests" workflow.

### What I actually built

- Added `authHeaderName` / `authHeaderScheme` options to `ChatCompletionsProvider`, with a small `buildAuthHeader()` helper used by both `invoke()` and `stream()`. Defaults stay `Authorization` / `bearer`, so nothing else changes.
- Added the `azure-openai` leaf that sets `api-key` / `raw` and composes the Azure URL in its own factory.
- Regenerated the catalogs and fixed the roster tests.

### Plan vs. what changed

In Phase II I planned to also add URL templating to the shared provider. When I actually looked at it, I realized `completionsPath` is already treated as an opaque string (that's how `perplexity` overrides it), so Azure can just build the full `/openai/deployments/{deployment}/chat/completions?api-version=…` path in its own factory and pass it in. That meant the only shared-code change I needed was the header — which is the one thing that genuinely couldn't be done from the leaf. Smaller change, same result.

---

## Implementation Notes

### Week 1 Progress

I picked #304, read the issue and the maintainer's notes, and commented asking to take it up (referencing my vLLM work). I confirmed it targets the same integration branch I already worked on, and noted early that Azure would probably need a shared-code change, not just a new leaf.

### Week 2 Progress (Phase II — Reproduce and Plan)

I cut the `feat/azure-openai-provider-304` branch off the integration branch, got the baseline suite green (440 tests), and did the investigation above to prove Azure was missing and find the Bearer-header blocker. I also used `git blame` on the shared provider and the schema, which showed the schema already has a `raw` header scheme even though the shared provider doesn't use it yet — good sign that the header fix was the intended direction.

### Week 3 Progress (Phase III — Build)

This is where I actually wrote the code (see the Implementation Progress section below for the file-by-file details and commit hashes). Short version: one small shared-provider change for the header, the Azure leaf, regenerated catalogs, tests, and roster fixes. Full suite ended green at 473 tests.

### Week 4 Progress (Phase IV — PR)

Opened PR [#425](https://github.com/orthogonalhq/nous-core/pull/425) against the integration branch. The first open showed merge conflicts because other provider leaves had landed since I branched, so I rebased, updated the roster expectations to include both `azure-openai` and the new vendors, force-pushed, and got the PR mergeable.

### Maintainer review + follow-up fix

On 2026-07-21 @atlamors left a **Request changes** review. The scope looked good (BYOK only, no Foundry/#421 stuff), and the deployment path / `api-key` header / fail-closed key handling were fine — but he caught a **shared-runtime** bug that made Azure unusable through the normal registry path:

`ProviderRegistry.normalizeRemoteConfig()` was always overwriting the configured endpoint with `definition.defaultEndpoint`. For Azure, that default is just a placeholder (`https://your-resource.openai.azure.com`), so a real BYOK host like `https://acme-resource.openai.azure.com` got wiped before the factory ran. My factory/unit tests missed it because they call the factory directly and skip the registry.

I replied on the PR agreeing it was a shared-runtime gap, then fixed it so normalization only falls back to the definition default when `config.endpoint` is absent, and added a registry-level Azure test that asserts the invoked URL uses the custom resource host (not the placeholder). Also updated two openai/anthropic proxy-endpoint tests that had been asserting the old overwrite behavior.

---

## Implementation Progress (Phase III — Build)

**Branch:** `feat/azure-openai-provider-304` (my fork `skonda29/nous-core`, based on `feat/contributor-friendly-inference-provider-surface`).

I split the work into reviewable commits. After rebasing onto the latest integration tip (other leaves had landed — gemini, mistral, qwen-code, xai), plus the review follow-up, the hashes are:

| Commit | Message | What it does |
|--------|---------|--------------|
| `6c703ab8` | `providers: support custom auth header scheme in ChatCompletionsProvider` | Adds `authHeaderName` / `authHeaderScheme` + a `buildAuthHeader()` helper used by `invoke()` and `stream()`. Defaults unchanged, so the other leaves aren't affected. |
| `113d3b91` | `providers: add Azure OpenAI provider leaf (#304)` | The `azure-openai` leaf (`definition.ts`, `adapter.ts`, `provider.ts`, `index.ts`), regenerated catalogs, and the Azure tests. |
| `a928b084` | `providers: register azure-openai in shared vendor-roster tests` | Adds `azure-openai` to the hard-coded roster tests (merged with the newly landed vendors). |
| `67681d25` | `providers: preserve configured endpoint through registry normalization (#425)` | Review follow-up: `normalizeRemoteConfig()` keeps an explicit `config.endpoint` (only falls back to `definition.defaultEndpoint` when absent). Adds a registry-level Azure test for a custom resource host; updates openai/anthropic tests that assumed the old overwrite. |

The PR diff is scoped to the issue — all under `self/subcortex/providers/`, no random formatting changes or commented-out code.

**Files I created:**

- `providers/azure-openai/definition.ts` — the metadata: `vendorKey: 'azure-openai'`, `api-key` / `raw` auth via `AZURE_OPENAI_API_KEY`, a placeholder endpoint + deployment name the user has to replace, no `modelListEndpoint` (BYOK, so no "list my deployments" surface), and no hand-authored `wellKnownProviderId`. I wrote the out-of-scope list (Foundry / Entra / quota / #421) in the file's comment.
- `providers/azure-openai/provider.ts` — the factory. It resolves `AZURE_OPENAI_API_KEY` itself and throws if it's missing (so it can't fall back to an OpenAI key), builds the path with `buildAzureCompletionsPath(deployment, apiVersion)`, and reads the api-version from `AZURE_OPENAI_API_VERSION` (default GA `2024-10-21`).
- `providers/azure-openai/adapter.ts`, `index.ts` — re-export the shared adapter (same as OpenAI/Groq).
- `__tests__/azure-openai-provider.test.ts` (302 lines) and `__tests__/providers/azure-openai.test.ts` (102 lines).

**Files I modified:**

- `protocols/openai-api/provider.ts` — shared auth-header options + helper (`authHeaderName` / `authHeaderScheme`).
- `runtime/provider-runtime.ts` — review follow-up: preserve an explicit configured endpoint in `normalizeRemoteConfig()` instead of always overwriting with `definition.defaultEndpoint`.
- `__tests__/chat-completions-provider.test.ts` — tests for the new header behavior.
- `__tests__/provider-registry.test.ts` — registry-level Azure custom-endpoint test + openai/anthropic expectation updates.
- `provider-definitions.ts`, `provider-adapters.ts`, `provider-factories.ts` — regenerated (not edited by hand).
- `__tests__/adapter-resolver.test.ts`, `provider-codegen.test.ts`, `provider-definitions/provider-definitions.test.ts`, `provider-definitions/provider-definition-types.test.ts`, `provider-pipeline-integration.test.ts` — roster updates.

### Challenges I ran into

- **Not sending an OpenAI key to Azure by accident.** The shared provider falls back to `OPENAI_API_KEY` if you don't give it a key. For a BYOK Azure connector that's actually dangerous — you could send someone's OpenAI key to their Azure endpoint. I fixed it by resolving `AZURE_OPENAI_API_KEY` in the factory and throwing if it's missing (same idea as the `perplexity` leaf), and I wrote a test that checks the fallback never happens.
- **Deployment name vs. model name.** Azure routes by deployment name, not model, and you only know it once a config exists. So I treat `config.modelId` as the deployment name and build the path per-instance (not as a static `completionsPath`), and I URL-encode it with a test for weird characters.
- **api-version is required and changes.** Azure returns a 400 without it, and supported versions differ per resource. I defaulted to `2024-10-21` but made `AZURE_OPENAI_API_VERSION` override it, with tests for both.
- **The hard-coded roster tests (already knew this one from vLLM).** Adding a vendor breaks one compile-time exact-union test and four runtime roster tests. This time I updated all five as part of the commits instead of getting surprised at the end.
- **Rebase conflicts before the PR was mergeable.** By the time I opened [#425](https://github.com/orthogonalhq/nous-core/pull/425), the integration branch had moved a lot (xAI, Mistral, Gemini, Qwen Code, etc.). GitHub showed conflicts in the same roster/catalog files. I rebased onto the latest tip, merged `azure-openai` into the updated vendor lists (kept gemini/mistral/qwen-code/xai), force-pushed, and the PR went mergeable.
- **Registry overwrote the Azure resource endpoint (maintainer review).** My leaf tests only covered the factory path, so I missed that `normalizeRemoteConfig()` replaced every remote provider's endpoint with the definition default. For Azure that default is a placeholder, so the real BYOK host never reached the provider. Fixed centrally in the registry and added a registry-level test that registers Azure with `https://acme-resource.openai.azure.com` and asserts the invoked URL uses that host.

---

## Testing Strategy

I followed the existing test files (mostly `perplexity-provider.test.ts` and `chat-completions-provider.test.ts`) and used the same `vi`-mocked `fetch` approach instead of poking at private internals.

**Test that covers the actual fix — `__tests__/chat-completions-provider.test.ts` (+72 lines):**
- with no options it still sends `Authorization: Bearer <key>` (so I know I didn't break the other leaves);
- with `authHeaderScheme: 'raw'` + `authHeaderName: 'api-key'` it sends `api-key: <key>` and no `Authorization`;
- checked on both `invoke()` and `stream()`.

**Azure leaf tests — `azure-openai-provider.test.ts` + `providers/azure-openai.test.ts`:**
- definition: placeholder endpoint/model, requires `AZURE_OPENAI_API_KEY`, declares the raw `api-key` header, uses `chat-completions`, no discovery endpoint, no hand-authored id;
- `buildAzureCompletionsPath()`: builds the deployment path with api-version, and URL-encodes special characters;
- factory/transport: throws when no key; never falls back to `OPENAI_API_KEY`; reads the key from the env var; targets the deployment path with the default api-version; honors `AZURE_OPENAI_API_VERSION`; sends the raw `api-key` header (not Bearer); rejects invalid input with `ValidationError`; returns a `ModelResponse` on success; maps 401 → `PROVIDER_AUTH_FAILED`, 429 → `PROVIDER_UNAVAILABLE`, other errors → `PROVIDER_UNAVAILABLE`; `stream()` yields chunks + a final done chunk.

**Roster tests updated:** `adapter-resolver`, `provider-codegen`, `provider-definition-types` (the compile-time one), `provider-definitions`, `provider-pipeline-integration`.

**Registry-level test added after review — `__tests__/provider-registry.test.ts`:**
- registers an Azure provider with a custom resource endpoint (`https://acme-resource.openai.azure.com`);
- asserts `provider.getConfig().endpoint` keeps that host (not the placeholder);
- asserts `invoke()` hits `https://acme-resource.openai.azure.com/openai/deployments/.../chat/completions?api-version=2024-10-21`.

**What passed:**
- Full provider suite was green at Phase III (**473 passing / 4 skipped**); after the registry fix I re-ran the Azure + registry tests covering the new path.
- `check:generated` clean (maintainer also confirmed this on review).
- I didn't spin up a real Azure resource (it's BYOK, there's no shared test key), so the transport is checked automatically with a mocked `fetch` that asserts the exact URL and headers. I did manually compare the URL I build against Microsoft's REST docs to make sure the path and default api-version are right.

---

## Pull Request

**PR:** [#425 — feat(providers): add Azure OpenAI BYOK provider leaf (#304)](https://github.com/orthogonalhq/nous-core/pull/425)

**PR target:** `feat/contributor-friendly-inference-provider-surface` (same integration branch as the accepted Groq [#404], llama.cpp [#403], and my vLLM [#417] leaves — not `dev` or `main`).

**Status:** Open. Opened 2026-07-17 against upstream (`orthogonalhq/nous-core`) from my fork (`skonda29/nous-core:feat/azure-openai-provider-304`). @atlamors requested changes on 2026-07-21 (preserve configured endpoint through registry normalization); I pushed the fix (`67681d25`) and replied on the PR. Awaiting re-review.

**Summary:** Adds Azure OpenAI as a certified BYOK provider leaf under the narrowed scope from #304. Shared changes: (1) backwards-compatible auth-header options on `ChatCompletionsProvider` (`api-key`/raw vs. default bearer), and (2) after review, `ProviderRegistry.normalizeRemoteConfig()` preserves an explicit configured endpoint so BYOK hosts aren't overwritten by the leaf's placeholder default. The Azure deployment URL is built in the leaf factory using the already-opaque `completionsPath`. Fail-closed on credentials, id derived from `vendorKey`, catalogs regenerated via the generator. Foundry / Entra / quota / routing / billing left to #421.

---

## Maintainer Feedback Log

| Date | Source | Feedback / Guidance | My Response | Commit / Ref |
|------|--------|---------------------|-------------|--------------|
| 2026-06-18 | @atlamors on #304 (issue body) | Build it as a certified leaf under `providers/<vendor>/`, target the integration branch, use `ProviderDefinitionLeaf` with ids from `vendorKey`, don't hand-author `wellKnownProviderId`, the old `IModelProvider` path is dead. | Followed all of it. | — |
| 2026-06-28 | — | Asked to take the issue. | Started reproduction while waiting. | #304 comment |
| 2026-06-30 | `git blame` on `provider.ts:97`, `provider-definition.ts:34` | The refactor that added the `raw` scheme to the schema left the shared provider hard-coding bearer. | Told me the header fix in the shared provider was the intended direction. | — |
| 2026-07-08 | @atlamors on #304 | Narrowed it to a **BYOK connector only** (endpoint + key + deployment name as the model), excluded Foundry / Entra / quota / routing / fallback / billing (those go to #421), and asked me to flag protocol-boundary issues in the PR. Assigned me. | Built exactly that scope; `config.modelId` = deployment name; descope written into `definition.ts`; only the header changed centrally, URL stayed in the leaf. | `6c703ab8`, `113d3b91`, `a928b084` |
| 2026-07-10 | @atlamors on #304 | Acknowledged my update — "Looking forward to the PR." | Confirmed I'd started under the narrowed scope and had checked sibling leaf PRs (#424, #419, #420) for review issues to avoid (doubled `/v1/v1` path, adapterKey collisions); neither applies here. | #304 comment |
| 2026-07-17 | PR #425 opened | Opened against the integration branch; rebased after merge conflicts from newly landed leaves. | Flagged the auth-header protocol-boundary change and the #421 descope in the PR description. | PR [#425](https://github.com/orthogonalhq/nous-core/pull/425) |
| 2026-07-21 | @atlamors on PR #425 (**Request changes**) | Scope looks right (BYOK only). Deployment path, `api-key` header, and fail-closed key handling look good. **Blocker:** `ProviderRegistry.normalizeRemoteConfig()` overwrites every remote provider's configured endpoint with `definition.defaultEndpoint`, so Azure's placeholder default replaces the user's real resource host before the factory runs. Fix by only falling back to the definition default when `config.endpoint` is absent, and add a registry-level Azure test that asserts the invoked URL uses the custom host. Treated as shared-runtime debt, not an Azure leaf mistake — but still required before merge. CI roster churn from other merges is not a contributor blocker. | Agreed it's a shared-runtime gap; fixed normalization and added the registry-level Azure custom-endpoint test; updated openai/anthropic tests that asserted the old overwrite. | `67681d25` |
| 2026-07-26 | me on PR #425 | — | Replied thanking him for the review and summarizing the fix before/while pushing. | PR [#425](https://github.com/orthogonalhq/nous-core/pull/425) comment |

> This log will be updated with further review comments and my responses (with commit refs) as re-review happens.

---

## Process & Communication

- **2026-06-28** — Commented on #304 asking to take it, referencing my vLLM work.
- **2026-07-08** — @atlamors narrowed the scope and assigned me.
- **2026-07-09** — Built Phase III, got the suite green (473), pushed the branch.
- **2026-07-10** — Replied on [#304](https://github.com/orthogonalhq/nous-core/issues/304) confirming I'd started with the narrowed scope in mind, and that I'd checked the other in-flight leaf PRs to avoid repeating issues already caught in their reviews:

  > Hi @atlamors, thank you for the guidance. I started working on this already keeping your suggestions in mind. Also checked the other leaf PRs (#424, #419, #420) to dodge issues already caught in review there (doubled /v1/v1 path, adapterKey collisions) — neither applies here, wanted good to confirm.

  @atlamors replied: *"Awesome, thanks @skonda29. Looking forward to the PR."*

- **2026-07-17** — Opened PR [#425](https://github.com/orthogonalhq/nous-core/pull/425) against `feat/contributor-friendly-inference-provider-surface`. Rebased onto the latest integration tip to clear catalog/roster conflicts from newly merged leaves (xAI, Mistral, Gemini, Qwen Code), force-pushed, and confirmed the PR is mergeable.
- **2026-07-21** — @atlamors requested changes on #425: preserve the user-supplied Azure resource endpoint through `ProviderRegistry.normalizeRemoteConfig()` (don't overwrite with the placeholder default), and add a registry-level Azure test.
- **2026-07-26** — Replied on the PR and pushed the fix (`67681d25`):

  > Hi @atlamors, Thanks for the detailed review. You're right that this is a shared-runtime gap rather than an Azure-specific one. normalizeRemoteConfig() was unconditionally overwriting the configured endpoint with definition.defaultEndpoint, which breaks any per-resource BYOK host. Fixing it now so it only falls back to the definition default when config.endpoint is absent, and adding the registry-level Azure test you asked for (custom resource endpoint → asserts the invoked URL, not the placeholder). Will push shortly.

- **Check-in form:** submitted with **"Phase IV Complete"** / PR opened as required.
- **Next:** wait for re-review on #425.

---

## Engineering Judgment Beyond the Minimum

A few things I did that weren't strictly required:

- **Wrote down the descope instead of silently dropping it.** The maintainer narrowed the scope, so I put the excluded stuff (Foundry, Entra, quota, #421) right in `definition.ts` and in the feedback log, so it's clear the boundary was on purpose.
- **Caught a credential-leak edge case nobody mentioned.** The `OPENAI_API_KEY` fallback could leak an OpenAI key to an Azure endpoint. I made the leaf fail closed and added a test for it.
- **Made my own shared-code change smaller.** My Phase II plan wanted URL templating in the shared provider too; once I saw `completionsPath` is already opaque, I moved the URL into the leaf and only changed the header centrally.
- **Used the project's own test patterns.** I exported `buildAzureCompletionsPath` so it could be unit-tested directly, copied the mocked-`fetch` style from the existing tests, and added a regression test proving the default Bearer behavior still works for the other leaves.
- **Handled real Azure quirks.** URL-encoded deployment names, overridable api-version, GA default — all with tests.
- **Checked the other in-flight leaf PRs first.** Before opening mine I read the sibling provider PRs (#424, #419, #420) to see what reviewers had already flagged — a doubled `/v1/v1` path and adapterKey collisions. Neither applies to Azure, but confirming that up front means I'm not re-introducing a bug the maintainer already had to point out on someone else's PR.
- **Fixed the shared-runtime endpoint overwrite as a general BYOK fix, not an Azure hack.** When the maintainer pointed out `normalizeRemoteConfig()` wiping configured endpoints, I treated it as provider-surface debt (his framing) and fixed the shared path with a small `config.endpoint ?? definition.defaultEndpoint` change, plus a registry-level test so the next BYOK leaf doesn't hit the same hole.

---

## Learnings & Reflections

### What I learned

The biggest thing was using `git blame` before writing any code. On my first contribution I jumped in and had to reconcile onto the right branch later. This time I checked the history first, saw the schema already had a `raw` header scheme the runtime wasn't using, and that basically told me the header fix belonged in the shared provider.

I also got better at telling a workaround apart from a real fix. On vLLM, the `'no-auth'` placeholder was a fine workaround. On Azure, the header genuinely needed shared support — but the URL didn't, so I only changed what actually had to change. Figuring out where that line is was the useful part.

### What I'd do differently

Post the design decision on the issue before writing code, not after. It worked out here because the maintainer narrowed the scope first, but I'd rather confirm the "change shared code vs. keep it in the leaf" call up front every time.

### Something for future cohorts

Read `git blame` like it's a requirements doc. If the schema models something the runtime doesn't use yet, that gap is usually on purpose — the maintainer left an extension point. Finding that first saved me from over-building.

---

## Resources Used

- Azure OpenAI issue #304: https://github.com/orthogonalhq/nous-core/issues/304
- Provider adapter docs: https://docs.nue.orthg.nl/docs/development/provider-adapters/quickstart
- Provider leaf anatomy: https://docs.nue.orthg.nl/docs/development/provider-adapters/provider-leaf-anatomy
- Schemas / ABI reference: https://docs.nue.orthg.nl/docs/development/provider-adapters/schemas-abi-reference
- Testing checklist: https://docs.nue.orthg.nl/docs/development/provider-adapters/testing-checklist
- Leaves I looked at: `groq/` and `deepinfra/` (skeleton), `perplexity/` (fail-closed key), `anthropic/implementation.ts` (raw header), and my own `vllm/` leaf
- Merged PRs I referenced: [#403 llama.cpp](https://github.com/orthogonalhq/nous-core/pull/403), [#404 Groq](https://github.com/orthogonalhq/nous-core/pull/404)
- Azure OpenAI REST reference: https://learn.microsoft.com/azure/ai-services/openai/reference
