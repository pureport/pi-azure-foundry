# Deployment Discovery Specification

## Purpose

Deployment discovery queries the Azure AI Foundry deployments API at startup,
filters the results to chat-capable deployments, and resolves each deployment's
API route and model capability defaults (context window, max output tokens,
reasoning flag, input modalities, and token limit parameter). The resolved state
is stored in module-level maps (`apiRouteMap`) that are consulted at request
time by the streaming layer.

## Requirements

### Requirement: API Endpoint Construction

The system SHALL construct the deployments API URL from `resourceId` and
`projectId` using the pattern:
`https://<resourceId>.services.ai.azure.com/api/projects/<projectId>/deployments?api-version=v1`

#### Scenario: URL built from config fields

- GIVEN a config with `resourceId: "my-resource"` and `projectId: "my-project"`
- WHEN the extension initializes
- THEN the deployments fetch is made to
  `https://my-resource.services.ai.azure.com/api/projects/my-project/deployments?api-version=v1`

### Requirement: Authenticated Fetch

The system SHALL authenticate the deployments API request using an
`Authorization: Bearer` header with the token obtained from the configured
auth provider.

#### Scenario: Token used for discovery request

- GIVEN auth is configured (either api-key or azure-identity)
- WHEN the deployments endpoint is fetched
- THEN the HTTP request carries `Authorization: Bearer <token>`

### Requirement: HTTP Error Handling

The system SHALL throw a descriptive error and halt initialization if the
deployments API returns a non-2xx response.

#### Scenario: Non-2xx response from deployments API

- GIVEN the deployments API returns a 401 or 5xx status
- WHEN the extension processes the response
- THEN an error is thrown containing the HTTP status code and up to 200
  characters of the response body
- AND the extension does not proceed to register any models

### Requirement: Missing API Response Value

The system SHALL treat an API response with no `value` field as an empty
deployment list, using `data.value ?? []`. This results in the
"No chat-capable deployments found" error being thrown.

#### Scenario: API returns response without value field

- GIVEN the deployments API returns a JSON body with no `value` key
- WHEN the extension processes the response
- THEN the deployment list is treated as empty
- AND an error with the message `"No chat-capable deployments found"` is thrown

### Requirement: Chat-Capability Filter

The system SHALL include only deployments where
`capabilities.chat_completion === "true"`. All other deployments SHALL be
excluded from model registration.

#### Scenario: Mixed deployment list

- GIVEN the API returns three deployments, two with `chat_completion: "true"`
  and one without
- WHEN the extension filters the list
- THEN only the two chat-capable deployments are registered as models

#### Scenario: No chat-capable deployments

- GIVEN the API returns deployments but none have `chat_completion: "true"`
- WHEN the extension filters the list
- THEN an error is thrown with the message `"No chat-capable deployments found"`
- AND the extension does not call `pi.registerProvider`

### Requirement: Model Name Resolution

The system SHALL use `modelName` as the display name and defaults lookup key
when present; otherwise it SHALL fall back to the deployment `name`.

#### Scenario: Deployment with explicit modelName

- GIVEN a deployment with `name: "my-deploy"` and `modelName: "gpt-4o"`
- WHEN the deployment is mapped to a model
- THEN the model's `name` field is `"gpt-4o"` and defaults are looked up
  under `"gpt-4o"`

#### Scenario: Deployment without modelName

- GIVEN a deployment with `name: "my-deploy"` and no `modelName`
- WHEN the deployment is mapped to a model
- THEN the model's `name` field is `"my-deploy"` and defaults are looked up
  under `"my-deploy"`

### Requirement: Known Model Defaults

The system SHALL apply pre-configured capability defaults from `MODEL_DEFAULTS`
when the resolved model name matches a known entry. The known entries are:

| Model name        | Context window | Max tokens | Reasoning | Input        |
|-------------------|----------------|------------|-----------|--------------|
| claude-sonnet-4-5 | 200,000        | 16,384     | true      | text + image |
| claude-sonnet-4-6 | 200,000        | 16,384     | true      | text + image |
| claude-haiku-4-5  | 200,000        | 16,384     | false     | text + image |
| claude-opus-4-5   | 200,000        | 32,000     | true      | text + image |
| gpt-5.4-nano      | 128,000        | 16,384     | false     | text + image |
| gpt-4o            | 128,000        | 4,096      | false     | text + image |
| gpt-4o-mini       | 128,000        | 4,096      | false     | text + image |
| Kimi-K2.5         | 131,072        | 8,192      | false     | text only    |
| Kimi-K2.6         | 131,072        | 8,192      | false     | text only    |

#### Scenario: Known model gets explicit defaults

- GIVEN a deployment resolves to model name `"gpt-4o"`
- WHEN `deploymentToModel` is called
- THEN the registered model has `contextWindow: 128000`, `maxTokens: 4096`,
  `reasoning: false`, and `input: ["text", "image"]`

### Requirement: Model ID Is Always Deployment Name

The system SHALL set the registered model's `id` to the deployment `name`
(never to `modelName`). Two deployments of the same model type but with
different deployment names will receive distinct `id` values.

#### Scenario: Deployment name used as model ID

- GIVEN a deployment with `name: "my-deploy"` and `modelName: "gpt-4o"`
- WHEN `deploymentToModel` is called
- THEN the registered model has `id: "my-deploy"`

### Requirement: Zero-Cost Registration

The system SHALL register all models with a zero-cost structure:
`{ input: 0, output: 0, cacheRead: 0, cacheWrite: 0 }`. No per-token pricing
is configured.

#### Scenario: Model registered with zero cost

- GIVEN any deployment
- WHEN `deploymentToModel` is called
- THEN the registered model has `cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 }`

### Requirement: Unknown Model Fallback

The system SHALL apply a fallback of 128K context, 4K max output, no reasoning,
and text-only input for any deployment whose model name is not in `MODEL_DEFAULTS`.

#### Scenario: Unknown model uses fallback defaults

- GIVEN a deployment with `modelName: "some-unknown-model"`
- WHEN `deploymentToModel` is called
- THEN the registered model has `contextWindow: 128000`, `maxTokens: 4096`,
  `reasoning: false`, and `input: ["text"]`

### Requirement: Unknown Model Logging

The system SHALL log a warning for each deployment whose model name is not in
`MODEL_DEFAULTS`, including the deployment name, model name, and inferred route.

#### Scenario: Unknown model emits a warning

- GIVEN a deployment with `modelName: "some-unknown-model"`
- WHEN the extension completes discovery
- THEN a log line is printed containing the deployment name, `"no explicit defaults for"`,
  the model name, and the inferred API route description

### Requirement: API Route Resolution

The system SHALL determine the API route for each deployment at discovery time
and store it in `apiRouteMap` keyed by deployment name.

- Deployments with `modelPublisher === "Anthropic"` SHALL be assigned route
  `anthropic-messages`.
- All other deployments SHALL be assigned route `openai-chat-completions` with
  a token limit parameter of either `max_tokens` or `max_completion_tokens`.

Note: `resolveApiRoute` returns the route but does not write to `apiRouteMap`.
It is `deploymentToModel` that calls `apiRouteMap.set(d.name, resolveApiRoute(d))`
as a side effect during model construction.

#### Scenario: Anthropic deployment gets Messages API route

- GIVEN a deployment with `modelPublisher: "Anthropic"`
- WHEN `deploymentToModel` is called
- THEN `apiRouteMap` contains `{ kind: "anthropic-messages" }` for that deployment

#### Scenario: OpenAI deployment gets chat completions route

- GIVEN a deployment with `modelPublisher: "Microsoft"` and `modelName: "gpt-4o"`
- WHEN `deploymentToModel` is called
- THEN `apiRouteMap` contains
  `{ kind: "openai-chat-completions", tokenLimit: "max_tokens" }` for that deployment

### Requirement: Token Limit Parameter Inference

The system SHALL use `max_completion_tokens` for GPT-5 and o-series models on
the OpenAI-compatible route, and `max_tokens` for all others.

Inference rules (applied in order):

1. If the model name has an explicit `openaiTokenLimit` in `MODEL_DEFAULTS`, use it.
2. If the model name matches `/^(gpt-5|o[1-9])([-.]|$)/i`, use
   `max_completion_tokens`.
3. Otherwise use `max_tokens`.

#### Scenario: gpt-5.4-nano uses max_completion_tokens

- GIVEN a deployment with `modelName: "gpt-5.4-nano"`
- WHEN the route is resolved
- THEN `tokenLimit` is `"max_completion_tokens"`

#### Scenario: gpt-4o uses max_tokens

- GIVEN a deployment with `modelName: "gpt-4o"`
- WHEN the route is resolved
- THEN `tokenLimit` is `"max_tokens"`

#### Scenario: o1-series model uses max_completion_tokens

- GIVEN a deployment with `modelName: "o1-mini"` (not in MODEL_DEFAULTS)
- WHEN the route is resolved
- THEN `tokenLimit` is `"max_completion_tokens"` (matched by the regex rule)

### Requirement: Discovery Summary Logging

The system SHALL log a single summary line after discovery listing the number of
chat-capable deployments found and each deployment's name, publisher, and API
route description.

#### Scenario: Summary line emitted after successful discovery

- GIVEN three chat-capable deployments are found
- WHEN the extension completes discovery
- THEN a log line is printed matching
  `"[Azure Foundry] Found 3 deployment(s): <name> (<publisher>, <route>), ..."`
