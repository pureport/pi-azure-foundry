# Extension Specification

## Purpose

The extension entry point is the exported default async function that pi invokes
at startup. It orchestrates the full initialization sequence: loading config,
acquiring an initial auth token, fetching the Azure Foundry deployments list,
mapping deployments to pi models, populating the module-level auth map, and
registering the provider with `pi.registerProvider`. It is the sole integration
seam between the rest of the extension's logic and the pi runtime.

## Requirements

### Requirement: Endpoint URL Construction

The system SHALL construct the Azure AI Foundry base endpoint as
`https://<resourceId>.services.ai.azure.com/api/projects/<projectId>` using
values from the loaded config, and SHALL use this endpoint as both the
deployment discovery URL base and the `baseUrl` passed to `pi.registerProvider`.

#### Scenario: Endpoint reflects config values

- GIVEN a config with `resourceId: "my-resource"` and `projectId: "my-project"`
- WHEN the extension initializes
- THEN the deployments API is called at
  `https://my-resource.services.ai.azure.com/api/projects/my-project/deployments?api-version=v1`
- AND `baseUrl` registered with pi equals
  `https://my-resource.services.ai.azure.com/api/projects/my-project`

### Requirement: Startup Log Sequence

The system SHALL emit exactly these log lines during initialization, in order,
before any network request is made:

1. `[Azure Foundry] Loading config from: <path>` (emitted by `loadConfig`)
2. `[Azure Foundry] Fetching deployments from: <url>`
3. `[Azure Foundry] Auth: <auth-type>`

Both lines 2 and 3 appear before `getToken()` is called and before the
deployments fetch is initiated.

#### Scenario: Fetch URL logged before auth type

- GIVEN a valid config
- WHEN the extension initializes
- THEN `"[Azure Foundry] Fetching deployments from: <url>"` is printed first
- AND `"[Azure Foundry] Auth: <type>"` is printed immediately after
- AND both appear before any network call

### Requirement: Deployment Discovery Request

The system SHALL fetch deployments from the Foundry API using a
`Bearer <token>` Authorization header and the `v1` API version, regardless of
which auth type is configured.

#### Scenario: Discovery request uses Bearer authorization

- GIVEN a valid config (either auth type)
- WHEN the extension initializes
- THEN a GET request is sent to the deployments endpoint with
  `Authorization: Bearer <token>` where `<token>` is the value returned by the
  configured token getter

### Requirement: HTTP Error Handling During Discovery

The system SHALL throw a descriptive error and halt initialization if the
deployments API returns a non-2xx HTTP status.

#### Scenario: API returns an error status

- GIVEN the deployments API responds with status 401
- WHEN the extension initializes
- THEN an error is thrown containing the status code and up to 200 characters
  of the response body
- AND `pi.registerProvider` is never called

### Requirement: Chat-Capable Deployment Filtering

The system SHALL register only deployments that have
`capabilities.chat_completion` equal to `"true"`, silently ignoring all other
deployments returned by the API.

#### Scenario: Mixed deployment list is filtered

- GIVEN the API returns three deployments: two with `chat_completion: "true"`
  and one without
- WHEN the extension initializes
- THEN only the two chat-capable deployments are converted to models and
  registered

### Requirement: Empty Deployment List Error

The system SHALL throw an error and halt initialization if no chat-capable
deployments are found after filtering.

#### Scenario: No chat-capable deployments

- GIVEN the API returns a list where no deployment has
  `capabilities.chat_completion` equal to `"true"`
- WHEN the extension initializes
- THEN an error with the message `"No chat-capable deployments found"` is thrown
- AND `pi.registerProvider` is never called

### Requirement: Deployment Summary Logging

The system SHALL log a summary line listing the count of discovered deployments
along with each deployment's name, publisher, and API route description.

#### Scenario: Summary line emitted after successful discovery

- GIVEN three chat-capable deployments are found
- WHEN the extension completes discovery
- THEN a log line is printed matching
  `"[Azure Foundry] Found 3 deployment(s): <name> (<publisher>, <route>), ..."`

### Requirement: Unknown Model Defaults Warning

The system SHALL log a warning for each registered deployment whose model name
is not present in `MODEL_DEFAULTS`, indicating the deployment name, the unknown
model name, and the inferred API route.

#### Scenario: Unknown model name triggers warning

- GIVEN a deployment with `modelName: "some-new-model"` not in `MODEL_DEFAULTS`
- WHEN the extension completes registration
- THEN a log line is printed containing the deployment name,
  `"no explicit defaults for"`, the model name, and the route description

### Requirement: ProviderAuth Map Population

The system SHALL store a `ProviderAuth` entry in `providerAuthMap` under the
`"azure-foundry"` key before calling `pi.registerProvider`, so that the
streaming layer can look it up per-request.

#### Scenario: Auth map populated before registration

- GIVEN a valid config with any auth type
- WHEN the extension initializes successfully
- THEN `providerAuthMap.get("azure-foundry")` returns a record with the correct
  `type` and `getToken` function

### Requirement: Provider Registration

The system SHALL call `pi.registerProvider` exactly once per initialization with
a fixed provider ID of `"azure-foundry"` and the following fields:

- `name`: `"Azure Foundry"`
- `baseUrl`: the constructed endpoint URL
- `apiKey`: the literal API key for `api-key` auth, or the sentinel string
  `"azure-identity"` for identity auth
- `api`: the string `"azure-foundry"`
- `streamSimple`: the `streamAzureFoundry` function
- `models`: the array of models built from chat-capable deployments

#### Scenario: api-key auth passes real key to registerProvider

- GIVEN a config with `auth.type: "api-key"` and `apiKey: "real-key"`
- WHEN the extension registers the provider
- THEN `pi.registerProvider` is called with `apiKey: "real-key"`

#### Scenario: azure-identity auth passes sentinel to registerProvider

- GIVEN a config with `auth.type: "azure-identity"`
- WHEN the extension registers the provider
- THEN `pi.registerProvider` is called with `apiKey: "azure-identity"`

### Requirement: Registration Success Logging

The system SHALL log a confirmation line after successfully calling
`pi.registerProvider`, including the count of registered models.

#### Scenario: Success log after registration

- GIVEN three chat-capable deployments were found
- WHEN `pi.registerProvider` is called successfully
- THEN a log line is printed matching `"✓ Registered 3 model(s)"`
