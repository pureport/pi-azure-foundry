# Streaming Specification

## Purpose

The streaming domain handles all real-time token delivery between Azure AI
Foundry and the pi event stream. It covers two distinct protocol paths — the
Anthropic native Messages API and the OpenAI-compatible chat completions API —
unified behind a single `streamAzureFoundry` dispatcher. It also owns SSE frame
parsing, message format conversion for both paths, and token usage and cost
accounting.

## Requirements

### Requirement: SSE Frame Parsing

The system SHALL parse Server-Sent Events from a `ReadableStream`, yielding only
the data payload of each `data:` line, and SHALL terminate cleanly on a `[DONE]`
sentinel or when the stream ends.

#### Scenario: Normal data frames are yielded

- GIVEN a readable stream containing SSE lines prefixed with `data:`
- WHEN `parseSSE` consumes the stream
- THEN each `data:` payload string is yielded in order

#### Scenario: DONE sentinel terminates the generator

- GIVEN a readable stream that emits `data: [DONE]`
- WHEN `parseSSE` encounters that line
- THEN the generator returns immediately without yielding the sentinel

#### Scenario: Partial chunks across read boundaries are reassembled

- GIVEN a readable stream whose chunks split an SSE line across multiple reads
- WHEN `parseSSE` consumes those chunks
- THEN the incomplete line is buffered and yielded only when the newline arrives

#### Scenario: Non-data lines are ignored

- GIVEN a readable stream containing comment lines (`: ...`) or event-type lines
- WHEN `parseSSE` processes the stream
- THEN those lines are not yielded

### Requirement: OpenAI Message Format Conversion

The system SHALL convert pi's internal `Message[]` format to the OpenAI chat
completions wire format, including system prompt, user messages (text and image),
assistant messages (text and tool calls), and tool result messages.

#### Scenario: System prompt is prepended

- GIVEN a context with a non-empty `systemPrompt`
- WHEN `toOpenAIMessages` is called
- THEN a `{ role: "system", content: systemPrompt }` message appears first

#### Scenario: Plain-text user messages are passed through

- GIVEN a user message whose content is a string
- WHEN `toOpenAIMessages` converts it
- THEN the output is `{ role: "user", content: "<string>" }`

#### Scenario: Multipart user messages with images use image_url

- GIVEN a user message with mixed text and image content blocks
- WHEN `toOpenAIMessages` converts it
- THEN image blocks become
  `{ type: "image_url", image_url: { url: "data:<mime>;base64,<data>" } }`

#### Scenario: Assistant tool calls are serialised to JSON strings

- GIVEN an assistant message containing `toolCall` content blocks
- WHEN `toOpenAIMessages` converts it
- THEN each tool call appears in `tool_calls` with `arguments` as a JSON string

#### Scenario: Tool result messages map to the tool role

- GIVEN a `toolResult` message
- WHEN `toOpenAIMessages` converts it
- THEN the output has `role: "tool"`, `tool_call_id`, and concatenated text content

### Requirement: Anthropic Message Format Conversion

The system SHALL convert pi's internal `Message[]` format to the Anthropic
Messages API wire format, including multipart user content, assistant thinking
blocks, tool use blocks, and tool results wrapped inside user messages.

#### Scenario: Image content uses base64 source format

- GIVEN a user message with an image content block
- WHEN `toAnthropicMessages` converts it
- THEN the image becomes
  `{ type: "image", source: { type: "base64", media_type, data } }`

#### Scenario: Assistant thinking blocks are preserved

- GIVEN an assistant message containing a `thinking` content block
- WHEN `toAnthropicMessages` converts it
- THEN the output includes `{ type: "thinking", thinking, signature }` in the
  content array

#### Scenario: Tool use blocks are emitted

- GIVEN an assistant message containing a `toolCall` content block
- WHEN `toAnthropicMessages` converts it
- THEN the output includes `{ type: "tool_use", id, name, input }` in the
  content array

#### Scenario: Tool results are wrapped in user messages

- GIVEN a `toolResult` message
- WHEN `toAnthropicMessages` converts it
- THEN the output is `{ role: "user", content: [{ type: "tool_result", ... }] }`

#### Scenario: Plain-string user content is passed through

- GIVEN a user message whose content is a plain string
- WHEN `toAnthropicMessages` converts it
- THEN the output is `{ role: "user", content: "<string>" }` (not an array)

#### Scenario: Empty text blocks are omitted from assistant messages

- GIVEN an assistant message with a `text` content block whose text is
  whitespace-only
- WHEN `toAnthropicMessages` converts it
- THEN that block is not included in the output

### Requirement: Stream Start Event

The system SHALL emit a `start` event as the first pi stream event on both the
OpenAI-compatible and Anthropic paths, immediately after receiving a successful
HTTP response and before any content events.

#### Scenario: Start event emitted before content

- GIVEN a successful HTTP response from either upstream path
- WHEN the streaming function begins processing
- THEN `stream.push({ type: "start", partial: output })` is emitted
- AND all subsequent text, tool call, and usage events follow it

### Requirement: OpenAI-Compatible Streaming

The system SHALL stream responses from the OpenAI-compatible endpoint, parsing
SSE chunks to build text, tool call, and usage content incrementally, and SHALL
emit pi stream events at each step.

#### Scenario: Text delta events are emitted

- GIVEN the OpenAI SSE stream emits chunks with `delta.content` strings
- WHEN `streamOpenAI` processes those chunks
- THEN `text_start`, `text_delta`, and `text_end` events are pushed to the pi stream

#### Scenario: Tool call deltas are accumulated and emitted

- GIVEN the OpenAI SSE stream emits `delta.tool_calls` fragments across chunks
- WHEN `streamOpenAI` processes them
- THEN `toolcall_start`, `toolcall_delta`, and `toolcall_end` events are pushed
- AND the final `arguments` object is the result of parsing the accumulated JSON

#### Scenario: Usage is recorded from the usage chunk

- GIVEN an SSE chunk contains a `usage` field with token counts (sent by
  Azure Foundry in response to `stream_options: { include_usage: true }`)
- WHEN `streamOpenAI` processes it
- THEN `output.usage.input`, `output.usage.output`, and `output.usage.totalTokens`
  are populated and `calculateCost` is called

### Requirement: OpenAI stream_options

The system SHALL always include `stream_options: { include_usage: true }` in
the OpenAI-compatible request body to ensure usage data is returned in the
SSE stream.

#### Scenario: stream_options present in every OpenAI request

- GIVEN any streaming request on the OpenAI-compatible path
- WHEN `streamOpenAI` builds the request body
- THEN the body contains `stream_options: { include_usage: true }`

#### Scenario: Stop reason is mapped from finish_reason

- GIVEN the final choice has `finish_reason: "tool_calls"`
- WHEN `streamOpenAI` processes it
- THEN `output.stopReason` is set to `"toolUse"`

### Requirement: Model Identification Per Path

The system SHALL identify the model differently on each API path:

- **OpenAI path**: the model is embedded in the URL as
  `/openai/deployments/<deployment-name>/chat/completions`; no `model` field
  is sent in the request body.
- **Anthropic path**: the model is sent as a `model: <deployment-name>` field
  in the JSON request body; no model appears in the URL path.

#### Scenario: OpenAI route uses deployment name in URL

- GIVEN a deployment with `id: "my-gpt4o-deploy"`
- WHEN `streamOpenAI` sends the request
- THEN the URL contains `/openai/deployments/my-gpt4o-deploy/chat/completions`
- AND the request body contains no `model` field

#### Scenario: Anthropic route sends model in body

- GIVEN a deployment with `id: "my-claude-deploy"`
- WHEN `streamAnthropic` sends the request
- THEN the request body contains `model: "my-claude-deploy"`

### Requirement: Token Limit Parameter Selection

The system SHALL use the `tokenLimit` field from the deployment's resolved
`ApiRoute` to set either `max_tokens` or `max_completion_tokens` in the OpenAI
request body.

#### Scenario: max_completion_tokens used for GPT-5 models

- GIVEN a deployment whose route has `tokenLimit: "max_completion_tokens"`
- WHEN `streamOpenAI` builds the request body
- THEN the body contains `max_completion_tokens: <value>` and no `max_tokens` key

#### Scenario: max_tokens used for standard models

- GIVEN a deployment whose route has `tokenLimit: "max_tokens"`
- WHEN `streamOpenAI` builds the request body
- THEN the body contains `max_tokens: <value>` and no `max_completion_tokens` key

### Requirement: OpenAI Auth Headers

The system SHALL set auth headers on the OpenAI-compatible route based on auth
type:

- `api-key` auth: header `"api-key: <token>"`
- `azure-identity` auth: header `"Authorization: Bearer <token>"`

#### Scenario: api-key auth uses api-key header

- GIVEN the provider auth type is `"api-key"`
- WHEN `streamOpenAI` sends the request
- THEN the request includes an `api-key` header with the token value
- AND no `Authorization` header is set

#### Scenario: azure-identity auth uses Bearer header

- GIVEN the provider auth type is `"azure-identity"`
- WHEN `streamOpenAI` sends the request
- THEN the request includes `Authorization: Bearer <entra-token>`
- AND no `api-key` header is set

### Requirement: Anthropic Streaming

The system SHALL stream responses from the Anthropic Messages API, handling
`content_block_start`, `content_block_delta`, `content_block_stop`,
`message_start`, and `message_delta` events to build text, thinking, and tool
call content incrementally.

#### Scenario: Text content block events are emitted

- GIVEN Anthropic SSE emits `content_block_start` with `type: "text"` followed
  by `text_delta` events
- WHEN `streamAnthropic` processes them
- THEN `text_start`, `text_delta`, and `text_end` events are pushed to the pi stream

#### Scenario: Thinking content blocks are emitted

- GIVEN Anthropic SSE emits `content_block_start` with `type: "thinking"` followed
  by `thinking_delta` and `signature_delta` events
- WHEN `streamAnthropic` processes them
- THEN `thinking_start`, `thinking_delta`, and `thinking_end` events are pushed
- AND the accumulated signature is stored on the thinking block

#### Scenario: Tool use content blocks are emitted

- GIVEN Anthropic SSE emits `content_block_start` with `type: "tool_use"` followed
  by `input_json_delta` events
- WHEN `streamAnthropic` processes them
- THEN `toolcall_start`, `toolcall_delta`, and `toolcall_end` events are pushed
- AND the final `arguments` object is the result of parsing accumulated JSON

#### Scenario: Usage is recorded from message_start and message_delta

- GIVEN Anthropic SSE emits `message_start` with input token counts and
  `message_delta` with output token counts
- WHEN `streamAnthropic` processes them
- THEN `output.usage.input`, `output.usage.output`, cache read/write fields,
  and `totalTokens` are populated and `calculateCost` is called

### Requirement: Anthropic Auth Header

The system SHALL always use `Authorization: Bearer <token>` on the Anthropic
route, regardless of auth type, since Azure Foundry treats API keys as valid
Bearer tokens on this path.

#### Scenario: api-key auth uses Bearer on Anthropic route

- GIVEN the provider auth type is `"api-key"`
- WHEN `streamAnthropic` sends the request
- THEN the request includes `Authorization: Bearer <api-key-value>`

#### Scenario: azure-identity auth uses Bearer on Anthropic route

- GIVEN the provider auth type is `"azure-identity"`
- WHEN `streamAnthropic` sends the request
- THEN the request includes `Authorization: Bearer <entra-token>`

### Requirement: Anthropic Version Header

The system SHALL include the header `anthropic-version: 2023-06-01` on all
requests to the Anthropic Messages API endpoint.

#### Scenario: Version header present on every Anthropic request

- GIVEN any streaming request routed to the Anthropic path
- WHEN `streamAnthropic` sends the request
- THEN the HTTP request includes `anthropic-version: 2023-06-01`

### Requirement: Malformed baseUrl Error Handling

The system SHALL derive the upstream host by calling `new URL(model.baseUrl).origin`.
If `model.baseUrl` is not a valid URL this throws a `TypeError`, which is caught
by the outer try/catch in `streamAzureFoundry` and emitted as an `error` event.

#### Scenario: Malformed baseUrl produces error event

- GIVEN `model.baseUrl` is not a valid URL string
- WHEN `streamAzureFoundry` is called
- THEN a `TypeError` is thrown, caught, and an `error` event is pushed to the
  pi stream with `stopReason: "error"`

### Requirement: Unified Dispatcher Routing

The system SHALL route each streaming request to `streamAnthropic` or
`streamOpenAI` based on the deployment's entry in `apiRouteMap`, defaulting to
`openai-chat-completions` with `max_tokens` when no entry exists.

#### Scenario: Anthropic route dispatched correctly

- GIVEN a model whose `apiRouteMap` entry is `{ kind: "anthropic-messages" }`
- WHEN `streamAzureFoundry` is called
- THEN `streamAnthropic` is invoked

#### Scenario: OpenAI route dispatched correctly

- GIVEN a model whose `apiRouteMap` entry is `{ kind: "openai-chat-completions" }`
- WHEN `streamAzureFoundry` is called
- THEN `streamOpenAI` is invoked with the route's `tokenLimit`

#### Scenario: Missing apiRouteMap entry defaults to OpenAI with max_tokens

- GIVEN a model with no entry in `apiRouteMap`
- WHEN `streamAzureFoundry` is called
- THEN `streamOpenAI` is invoked with `tokenLimit: "max_tokens"`

### Requirement: Error Handling in Streaming

The system SHALL catch any error thrown during streaming and emit an `error`
event on the pi stream, setting `stopReason` to `"aborted"` if the request was
cancelled via `options.signal`, or `"error"` otherwise.

#### Scenario: Network error emits error event

- GIVEN the upstream fetch throws a network error
- WHEN `streamAzureFoundry` processes the request
- THEN an `error` event is pushed to the pi stream with `stopReason: "error"`
- AND the stream is ended

#### Scenario: Aborted request emits aborted reason

- GIVEN `options.signal` is aborted before the request completes
- WHEN `streamAzureFoundry` processes the request
- THEN the `error` event has `stopReason: "aborted"`

### Requirement: HTTP Error Response Handling

The system SHALL throw a descriptive error containing the HTTP status code and
up to 500 characters of the response body when the upstream API returns a
non-2xx status.

#### Scenario: Non-2xx response throws with status and body

- GIVEN the upstream API returns status 429 with a JSON error body
- WHEN `streamOpenAI` or `streamAnthropic` receives the response
- THEN an error is thrown with a message containing `"Azure Foundry 429:"` and
  the first 500 characters of the response body

### Requirement: Tools Forwarding

The system SHALL include the tools list in the request body when `context.tools`
is non-empty, converting to the appropriate wire format for each path.

#### Scenario: Tools forwarded on OpenAI route

- GIVEN a context with one or more tools
- WHEN `streamOpenAI` builds the request body
- THEN the body includes a `tools` array in OpenAI function-calling format

#### Scenario: Tools forwarded on Anthropic route

- GIVEN a context with one or more tools
- WHEN `streamAnthropic` builds the request body
- THEN the body includes a `tools` array in Anthropic `input_schema` format
