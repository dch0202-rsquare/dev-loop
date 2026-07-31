---
id: platforms-environment-alternate-service-endpoints
domain: platforms
category: environment
applies_to: [general]
confidence: verified
sources:
  - https://docs.litellm.ai/docs/tutorials/claude_non_anthropic_models
  - https://code.claude.com/docs/en/env-vars
  - https://platform.claude.com/docs/en/build-with-claude/context-windows
  - https://docs.aws.amazon.com/sdkref/latest/guide/feature-ss-endpoints.html
  - https://docs.ollama.com/api/openai-compatibility
last_verified: 2026-07-31
related: [platforms-environment-path-resolution, infrastructure-config-environment-config, platforms-processes-background-services]
---

# Pointing a Client at an Alternate Service Endpoint

## When this applies

You are redirecting a CLI or SDK from its default service to another endpoint
via an environment variable — a gateway or proxy, a local emulator, or an
API-compatible reimplementation — and the first request fails (404, 400, an
auth error, or a limit error), or you are setting such a redirect up.

## Do this

Two facts decide almost every failure here: the base URL is a *prefix the client
appends its own path to*, and the client's defaults were sized for the service
it shipped against, not for the one you just pointed it at.

1. **Copy the base-URL form from that client's own documented example.**
   Whether the API version segment belongs to the base URL is a per-client
   convention, not a general rule:

| Client convention | Base URL form |
|-------------------|---------------|
| OpenAI-SDK-compatible clients (Ollama, vLLM, local gateways) | Includes the version segment — `http://localhost:11434/v1` — because the SDK appends `/chat/completions` to it |
| `ANTHROPIC_BASE_URL` (Claude Code → an Anthropic-format gateway such as LiteLLM) | Scheme, host, and port only — `http://0.0.0.0:4000` — because the client appends `/v1/messages` itself |
| AWS SDKs and CLI (`AWS_ENDPOINT_URL`) | Scheme and host, optionally with a path component; `AWS_ENDPOINT_URL_<SERVICE>` overrides the global value for one service |

2. **Set the credential variable alongside the base URL.** Endpoint and identity
   are separate settings: pointing at a gateway without giving it the gateway's
   credential leaves the client authenticating as before, so requests route to
   the new endpoint while billing and limits stay on the old identity.
3. **Re-fit the client's size and limit defaults to the new backend.** A
   maximum-output setting is not spare headroom — it is reserved from the same
   context window the input occupies, so `input + max_output` must fit the new
   backend's window. Read the client's current documented default rather than
   assuming a number (Claude Code's `CLAUDE_CODE_MAX_OUTPUT_TOKENS` default has
   changed across versions), then lower it to fit the smaller model.
4. **Prove the endpoint before configuring the client.** Send one request
   directly to the endpoint with `curl`, in both the non-streaming and the
   streaming form the client uses, and require a 200 on each. A translating
   proxy that implements one shape and not the other otherwise surfaces as a
   client bug.

## Edge cases

| Case | Then |
|------|------|
| 404 on the very first call | The version segment is duplicated or missing — log the full URL the client built and compare it against the vendor's documented example (directive 1) |
| 400 naming the context window on the first call, before any conversation | Input plus the client's default output cap exceeds the backend's window: lower the output cap (directive 3). Anthropic-format APIs return a validation error for this on pre-4.5 models and stop with `model_context_window_exceeded` on newer ones |
| Non-streaming succeeds, streaming hangs or 400s | Test both shapes against the endpoint directly; report the failing one to the proxy, and pin the client to the working shape until it is fixed |
| The client works but features silently disappear | The proxy must forward the headers and blocks the client relies on; check the client's documented gateway requirements rather than reading the loss as a model regression |
| The client produces no output at all and the endpoint's log shows no request | The call may never have been issued: rule out a blocked invocation (a scripted CLI waiting on stdin, a credential prompt) before investigating the endpoint |
| Endpoint override must apply to a service, a team, or CI rather than your shell | Distribute it as configuration for that context — [infrastructure-config-environment-config], and [platforms-environment-path-resolution] for which contexts read which files |

## Instead of

| If you are about to | Do this instead | Why |
|---------------------|-----------------|-----|
| Append `/v1` to a base URL "because that is the API path" | Use the vendor's documented example verbatim for that client | Half of clients append the version segment themselves; guessing produces `/v1/v1/...` and a 404 that looks like a routing failure |
| Read a first-request 400 as "the gateway is broken" | Compare `input + max_output` against the new backend's context window | The client's default cap was sized for a larger model; the request is rejected before the gateway does anything wrong |
| Debug the redirect from inside the client | `curl` the endpoint directly first, streaming and non-streaming | One request separates "the endpoint cannot serve this" from "the client is misconfigured" instead of conflating them |
| Set only the base URL and assume the whole session now routes through the gateway | Set the endpoint and the credential variable together, then verify identity on the gateway's own request log | The endpoint change is visible in the client, the identity change is only visible at the gateway |

## Sources

- https://docs.litellm.ai/docs/tutorials/claude_non_anthropic_models — `export ANTHROPIC_BASE_URL="http://0.0.0.0:4000"` (no version segment) with `ANTHROPIC_AUTH_TOKEN` for the gateway credential
- https://code.claude.com/docs/en/env-vars — `ANTHROPIC_BASE_URL` routes requests through a proxy or gateway; `CLAUDE_CODE_MAX_OUTPUT_TOKENS` sets the response cap (documented default 16000; values above the model's maximum are capped silently)
- https://platform.claude.com/docs/en/build-with-claude/context-windows — the window holds history plus the generated output; when input tokens plus `max_tokens` exceed it, pre-4.5 models return a validation error and 4.5+ models stop with `model_context_window_exceeded`
- https://docs.aws.amazon.com/sdkref/latest/guide/feature-ss-endpoints.html — `AWS_ENDPOINT_URL` / `AWS_ENDPOINT_URL_<SERVICE>`: scheme and host with an optional path component; service-specific overrides the global; explicit client/`--endpoint-url` settings win
- https://docs.ollama.com/api/openai-compatibility — OpenAI-SDK `base_url` includes the version segment (`http://localhost:11434/v1`) because the SDK appends `/chat/completions`
