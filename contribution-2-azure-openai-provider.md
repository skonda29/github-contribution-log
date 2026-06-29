# Contribution 2: Adapter: Azure OpenAI Model Provider

**Contribution Number:** 2

**Student:** Srinitya Kondapally (@skonda29)

**Issue:** https://github.com/orthogonalhq/nous-core/issues/304

**Status:** Phase I — Issue selected, requested to take it up (2026-06-28)

---

## Why I Chose This Issue

After finishing Contribution 1 (the vLLM provider leaf, #317), I already
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

## Understanding the Issue

### Problem Description

nous-core supports several model providers (Anthropic, OpenAI, Ollama, Groq,
llama.cpp, and — pending #417 — vLLM) through the certified provider-leaf system.
It does not yet include **Azure OpenAI**, which is a common deployment target for
teams that consume OpenAI models through Azure's hosting, compliance, and
regional guarantees.

The issue asks for an Azure OpenAI provider implemented against the **current**
provider-leaf contract (the older direct-`IModelProvider` file path is
superseded). Integration target: `feat/contributor-friendly-inference-provider-surface`.

### Expected Behavior (initial reading — to be confirmed during Phase II)

- Azure OpenAI has its own provider leaf under
  `self/subcortex/providers/src/providers/azure-openai/`.
- It reuses the OpenAI-compatible chat-completions request/response shape.
- It authenticates with the Azure `api-key` header and an env var such as
  `AZURE_OPENAI_API_KEY`.
- It targets the Azure deployment-style URL with an `api-version` parameter.
- The built-in provider id derives from `vendorKey` (no hand-authored
  `wellKnownProviderId`).
- Generated catalogs are updated via the generator, not by hand.
- Tests cover the definition, auth header, URL construction, invoke/stream, and
  error mapping.

### Current Behavior

Azure OpenAI is not registered as a provider. The closest references are the
**OpenAI** and **Groq** leaves (OpenAI-compatible, keyed) — but neither uses the
`api-key` header or the deployment/`api-version` URL shape, which is the gap this
issue will need to close.

### Affected Components (anticipated)

- `self/subcortex/providers/src/providers/azure-openai/` (new leaf)
- `self/subcortex/providers/src/protocols/openai-api/` (shared protocol — likely
  needs a central extension for the `api-key` header and Azure URL shape)
- `self/subcortex/providers/src/provider-definitions.ts` / `provider-adapters.ts`
  / `provider-factories.ts` (generated)
- Provider tests under `self/subcortex/providers/src/__tests__/`

---

## Initial Plan (to be refined in Phase II)

Based on what I built for vLLM, the high-level plan is:

1. **Reproduce the gap** — confirm Azure OpenAI isn't registered and prove the
   two concrete obstacles: the shared provider hard-codes `Authorization: Bearer`
   and the `/v1/chat/completions` URL shape.
2. **Decide where the fix lives.** Because the maintainer asked for protocol-level
   gaps to be fixed centrally, I expect the `api-key` header and Azure URL/
   `api-version` handling to belong in the shared `ChatCompletionsProvider`
   (likely behind options driven by the leaf definition), not as a one-off in the
   Azure leaf. I'll flag this in the issue/PR before implementing, per the
   maintainer's standing request.
3. **Author the leaf** under `providers/azure-openai/` mirroring the Groq leaf's
   keyed-auth metadata, with Azure-specific endpoint/`api-version`/deployment
   config.
4. **Regenerate catalogs** via `generate:providers`.
5. **Add tests** mirroring the existing per-provider tests (definition metadata,
   `api-key` header, deployment URL construction, invoke happy path, streaming,
   error mapping) and update the roster-pinning tests.
6. **Verify** with `check:generated`, `typecheck`, `lint`, and the provider
   suite, then open the PR against the integration branch.

> I'll confirm the exact Azure auth/URL contract against the provider-adapter
> docs and the most recently merged provider PRs before writing code — the
> biggest lesson from Contribution 1 was to match accepted precedent first.

---

## Process & Communication

- **2026-06-28** — Commented on #304 requesting to take it up, referencing my
  prior work on the vLLM provider (#317):
  > "Hi @atlamors, I previously worked on Adapter: vLLM Model Provider #317 and
  > would like to take up this issue and get started."
- Awaiting maintainer acknowledgment / assignment before starting Phase II.

---

## Maintainer Feedback Log

| Date | Source | Feedback / Guidance | My Response | Commit / Ref |
|------|--------|---------------------|-------------|--------------|
| 2026-06-18 | @atlamors on #304 (issue body) | Implement as a certified provider leaf under `providers/<vendor>/`; target `feat/contributor-friendly-inference-provider-surface`; leaves use `ProviderDefinitionLeaf` with ids derived from `vendorKey` (don't hand-author `wellKnownProviderId`); the direct-`IModelProvider` path is superseded. | Noted; plan above follows the leaf contract and the integration-branch target. | — |
| 2026-06-28 | — | Requested to take up the issue. | Awaiting assignment. | issue #304 comment |

---

## Resources

- Azure OpenAI issue #304: https://github.com/orthogonalhq/nous-core/issues/304
- Provider adapter docs: https://docs.nue.orthg.nl/docs/development/provider-adapters/quickstart
- Provider leaf anatomy: https://docs.nue.orthg.nl/docs/development/provider-adapters/provider-leaf-anatomy
- Schemas / ABI reference: https://docs.nue.orthg.nl/docs/development/provider-adapters/schemas-abi-reference
- Testing checklist: https://docs.nue.orthg.nl/docs/development/provider-adapters/testing-checklist
- Merged precedent leaves to model on:
  - [#404 — Groq model provider leaf](https://github.com/orthogonalhq/nous-core/pull/404) (keyed OpenAI-compatible)
  - [#403 — llama.cpp provider leaf](https://github.com/orthogonalhq/nous-core/pull/403)
  - My own vLLM leaf: [#417](https://github.com/orthogonalhq/nous-core/pull/417)
- Azure OpenAI REST reference (auth header + `api-version` + deployment URL):
  https://learn.microsoft.com/azure/ai-services/openai/reference
