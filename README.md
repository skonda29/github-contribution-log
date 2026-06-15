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

For this phase the goal was to actually get the project running on my own
machine, prove to myself that the feature is really missing, and then write
down a plan for how I'm going to add it. Since this issue is about adding a new
provider (not fixing something broken), "reproducing the bug" for me meant
showing that vLLM genuinely isn't there yet and finding out what's standing in
the way.

One thing I noticed right away: the refactor the maintainer told me about in
Phase I has already been merged. So the file path in the original issue
(`providers/src/vllm-provider.ts`) doesn't exist anymore. The providers are now
organized into small folders, one per provider, and there's a script that
auto-generates some files. I made sure my plan matches the new layout instead of
the old issue text.

---

## 1. Setting up the project locally

I cloned `orthogonalhq/nous-core` and tried to install and run the tests. It
took me a couple of tries, and I ran into one annoying problem that I want to
write down so I don't forget it.

### The Node version problem

My machine had Node 24 installed. Every time I ran `pnpm install` it crashed
with this scary error:

```
FATAL ERROR: invalid array length
... node::crypto::Hash::OneShotDigest ...
```

I was confused at first, but after looking into it I figured out that the
project is meant to run on **Node 22**, and the version of pnpm it uses doesn't
play nicely with Node 24. Once I downloaded Node 22 (`node-v22.14.0`) and used
that instead, the install finished in about 25 seconds with no errors. Lesson
learned: check the required Node version first!

### The commands that worked for me

```bash
git clone https://github.com/orthogonalhq/nous-core.git
cd nous-core
corepack prepare pnpm@10.6.2 --activate
pnpm install

# The provider package needs three other workspace packages built first.
# I tried running the full `pnpm build` but it failed in a different package
# (@nous/cortex-core) that has nothing to do with my issue, so I only built
# the parts I actually need:
pnpm --filter @nous/shared run build
pnpm --filter @nous/subcortex-inference-runtime run build
pnpm --filter @nous/autonomic-config run build

# Then I ran the provider tests:
npx vitest run self/subcortex/providers
```

This gave me **19 test files and 265 tests, all passing**. So now I have a
working baseline and I know the existing code is healthy before I start changing
anything.

---

## 2. Proving the gap is real

Since I can't "reproduce" a crash for a missing feature, I instead wrote a tiny
script that imports the built provider code and prints out what's registered.
Here's what it showed:

```text
Registered provider vendorKeys : [ 'anthropic', 'ollama', 'openai' ]
Has vllm definition?           : false
resolveProviderFactory("vllm") : undefined
ChatCompletionsProvider w/o key: THROWS -> PROVIDER_AUTH_FAILED - OpenAI API key required …
```

From this I learned three things:

1. **vLLM really isn't there.** The list of providers only has anthropic,
   ollama, and openai. Asking for a "vllm" provider gives back nothing.
2. **The old file path is gone.** Like I mentioned above, the code is now split
   into per-provider folders (`src/providers/<name>/` with `definition.ts`,
   `adapter.ts`, `provider.ts`, and `index.ts`), and there's a generator script
   that builds the combined lists for me.
3. **There's one tricky part.** vLLM uses the same API format as OpenAI, so my
   first thought was to just reuse the existing OpenAI-style provider. But that
   provider *requires* an API key and throws an error without one. vLLM is
   usually self-hosted, so the API key should be optional. This is the main
   thing I'll need to solve.

---

## 3. My plan for the implementation

### The approach I want to take

- **Reuse the existing OpenAI-compatible provider** instead of writing all the
  HTTP/streaming code from scratch, because vLLM speaks the same
  `/v1/chat/completions` format.
- **Treat vLLM as a local provider** (`isLocal: true`, `auth.required: false`),
  the same way Ollama is treated. This way the app won't force an API key and
  won't overwrite the user's custom server URL.
- **Use `vendorKey: 'vllm'`** — I checked, and the vendor field is just an open
  string, so I don't have to touch the shared package to add a new name.
- **Default settings:** endpoint `http://localhost:8000` and a placeholder model
  id (vLLM serves whatever model you started it with, the field just can't be
  empty).
- **Give it a new ID** `10000000-0000-0000-0000-000000000004` (the next number
  after the existing three providers).

### Fixing the optional API key (the one real decision)

- **My plan (Option B):** add a small `requireApiKey` option to the existing
  provider that defaults to `true`, and only send the auth header when a key is
  actually provided. My vLLM factory will pass `requireApiKey: false`. This is a
  tiny change, doesn't break the existing tests, and stays inside the issue's
  rules (I'm not allowed to change `IModelProvider` or `TextModelInputSchema`,
  and I'm not).
- **Backup idea (Option A):** write a completely separate `VllmProvider` class.
  It's more isolated but copies a lot of code, so I'd rather not. I'll mention
  it in the PR as the alternative.

### New files I'll add

- `src/providers/vllm/definition.ts` — describes the provider (name, protocol,
  default endpoint/model, optional auth via `VLLM_API_KEY`, marked as local).
- `src/providers/vllm/adapter.ts` — re-uses the existing chat-completions
  adapter (same as the openai folder does).
- `src/providers/vllm/provider.ts` — the factory that creates the provider with
  `requireApiKey: false`.
- `src/providers/vllm/index.ts` — exports the three pieces above.
- `src/__tests__/vllm-provider.test.ts` — tests copied/adapted from the existing
  openai provider tests, plus a new test that checks it works **without** an API
  key.

### Existing files I'll need to change

- `src/protocols/openai-api/provider.ts` — add the `requireApiKey` option and
  the conditional auth header.
- `src/index.ts` — export the new vLLM provider.
- **Re-run the generator** (`pnpm --filter @nous/subcortex-providers run
  generate:providers`) so the auto-generated provider lists pick up vLLM. I
  learned I should *not* edit those generated files by hand — there's a check
  that fails the build if they're out of sync.

### Tests that already hardcode the provider list (so they'll break until I fix them)

- `__tests__/provider-codegen.test.ts` — expects `['anthropic','ollama','openai']`,
  needs `'vllm'` added.
- `__tests__/provider-pipeline-integration.test.ts` — same kind of list to update.
- `__tests__/provider-definitions/provider-definitions.test.ts` — needs a vllm
  row added.

### Checking the issue's requirements

| What the issue asks for | How my plan covers it |
|---|---|
| invoke / stream / getConfig | Reused from the OpenAI-compatible provider |
| Validates input | Already handled by the reused provider |
| Streaming works | Already handled by the reused provider |
| Exported from the index | I'll add the export |
| Tests included | New `vllm-provider.test.ts` |
| Don't change the shared interface/schema | My plan doesn't touch them |

### How I'll double-check before opening the PR

```bash
pnpm --filter @nous/subcortex-providers run generate:providers
pnpm --filter @nous/subcortex-providers run check:generated
npx vitest run self/subcortex/providers   # should still be all green
pnpm lint
```

I'll also search the rest of the project for any other place that lists the
providers by name, just in case something else needs updating too.

---

## What I finished in Phase II

- I got the project building and the provider tests passing on my machine (265
  tests green).
- I proved that vLLM is missing and found the one real obstacle (the required
  API key).
- I wrote a clear, step-by-step plan that fits the new code layout instead of
  the outdated path in the issue.

**Next up (Phase III):** actually write the code following this plan and open
the pull request.
