---
id: backend-common-llm-response-completeness-validation
domain: backend
category: llm
applies_to: [openai-compatible-apis]
confidence: verified
sources:
  - https://developers.openai.com/api/docs/api-reference/chat/object
  - https://developers.openai.com/api/docs/guides/reasoning
  - https://docs.vllm.ai/en/latest/features/reasoning_outputs/
last_verified: 2026-07-31
related: [backend-common-reliability-timeouts-and-retries, qa-process-release-gates]
---

# Accepting an LLM Chat Completion as Usable Output

## When this applies

Application code calls an OpenAI-compatible `/v1/chat/completions` endpoint and
writes the returned text somewhere durable or user-visible (a document, a ticket,
a notification, another API call); and which model answers is decided by
configuration or gateway routing rather than by this call site.

## Do this

1. Treat HTTP 200 as "the call completed", not "the text is usable". Before using
   the text, branch on `choices[0]`:

| Case | Do |
|------|----|
| `finish_reason == "stop"` and `message.content` non-blank | Use the text |
| `finish_reason == "tool_calls"` | Not a final answer — run the tools, send the results in the next round, and cap the number of rounds |
| `finish_reason == "length"` | Fail the step. The reply was cut at the token limit, so the text is a fragment even when it reads like prose |
| `finish_reason == "content_filter"` | Fail the step and report the filter. Content was omitted by the provider, so resending the same messages returns the same result |
| `message.content` blank and `message.reasoning` (or `message.reasoning_content`) non-empty | Fail with "the routed model is a reasoning model and spent the output budget before emitting an answer" |
| `message.content` blank and no reasoning field present | Fail with the raw `finish_reason` and `usage` in the message |

2. Read both chain-of-thought field names, `reasoning` and `reasoning_content`.
   vLLM served this as `reasoning_content` (DeepSeek's name) and renamed it to
   `reasoning`, so a gateway fronting several servers returns either; neither is
   declared by the official OpenAI client types, so read them off the raw JSON or
   the client's extra-attributes escape hatch.
3. Put the distinguishing evidence in the failure message: `response.model` (the
   model actually served, which differs from the alias you asked for), the
   `finish_reason`, `usage.completion_tokens`, and the character counts of content
   versus the reasoning field. Those four values separate "reasoning model routed
   in" from "prompt too long" from "provider filtered it" without another round trip.
4. Size the token budget for reasoning plus answer. Reasoning tokens are billed and
   counted as output tokens, so a budget sized for the answer alone is consumed
   before any visible text appears — OpenAI's guidance is to reserve at least
   25,000 tokens for reasoning plus output when working with these models.
5. Cover the two silent shapes as test cases at your client wrapper: a 200 whose
   `content` is `""`, and a 200 whose `finish_reason` is `"length"`. Both must
   surface as failures with the model id in the message.

## Edge cases

| Case | Then |
|------|------|
| Streaming responses | Reasoning arrives as `delta.reasoning` and the stop condition as the final chunk's `finish_reason`; accumulate both and apply the same table after the stream closes |
| A configured alias resolves to a different model than the one you tested | Compare `response.model` against the expected id and log the mismatch — routing changes on the server, with no change in your request |
| The output is parsed as JSON or another structured format | A parse failure right after `finish_reason == "length"` is truncation, not a malformed model reply; report the truncation so the fix is budget or prompt size |
| Prompt plus context sits near the model's context window | Empty content also occurs when the context limit is hit rather than `max_tokens`; separate them with `usage.prompt_tokens` against the model's window |
| The endpoint is a local or self-hosted server | `reasoning`/`reasoning_content` appear only when the server was started with a reasoning parser enabled, so a served reasoning model can return blank content and no reasoning field at all — rely on `finish_reason` plus `usage` there |

## Instead of

| If you are about to | Do this instead | Why |
|---------------------|-----------------|-----|
| Read `message.content` and default a blank value to `""` | Branch on `finish_reason` and the reasoning field, and raise on every non-usable case | The `""` default writes an empty document or notification and reports success |
| Retry the identical request after `finish_reason == "length"` | Shorten the input, raise the token budget, or route to a non-reasoning model | The same request hits the same limit; only the budget or the routing changes the outcome |
| Log "LLM call failed" and move on | Log `response.model`, `finish_reason`, `usage`, content and reasoning lengths | Without the served model id, a routing change and a prompt-size bug look identical in the logs |

## Sources

- https://developers.openai.com/api/docs/api-reference/chat/object — `finish_reason` values: `stop` (natural stop point), `length` (the request's maximum token count was reached), `tool_calls`, `content_filter`
- https://developers.openai.com/api/docs/guides/reasoning — reasoning tokens "occupy space in the model's context window and are billed as output tokens"; hitting the limit yields an incomplete response with reason `max_output_tokens`, and "this might occur before any visible output tokens are produced"; reserve at least 25,000 tokens for reasoning and output
- https://docs.vllm.ai/en/latest/features/reasoning_outputs/ — `"reasoning" used to be called "reasoning_content"`; the field sits on the response message (`delta` when streaming), requires `--reasoning-parser` on the server, and is not officially supported by the OpenAI Python client's types
