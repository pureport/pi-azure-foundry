# Auth Specification

## Purpose

The auth domain is responsible for producing bearer tokens used in all Azure AI
Foundry HTTP requests. It supports two authentication modes — a static API key
and Azure identity-based tokens via `DefaultAzureCredential` — and caches Entra
ID tokens to avoid redundant network calls, refreshing proactively before expiry.

## Requirements

### Requirement: Auth Mode Selection

The system SHALL select the token acquisition strategy at extension load time
based on the `auth.type` field in the loaded config, and SHALL NOT mix
strategies within a single provider registration.

#### Scenario: API key mode selected

- GIVEN a config with `auth.type` equal to `"api-key"` and a non-empty `apiKey`
- WHEN `makeTokenGetter` is called with that config
- THEN it returns a function that always resolves to the configured `apiKey` string
- AND the returned function never makes a network call

#### Scenario: Azure identity mode selected

- GIVEN a config with `auth.type` equal to `"azure-identity"`
- WHEN `makeTokenGetter` is called with that config
- THEN it returns the `getIdentityToken` function backed by `DefaultAzureCredential`

### Requirement: Entra ID Token Caching

The system SHALL cache the most recently acquired Entra ID token in the
module-level `cachedToken` variable and SHALL reuse it across requests as long
as it remains valid with at least a 5-minute buffer before its expiry timestamp.

#### Scenario: Valid cached token is reused

- GIVEN a previously acquired token whose `expiresOnTimestamp` is more than
  5 minutes in the future
- WHEN `getIdentityToken` is called
- THEN the cached token string is returned immediately
- AND no call to `DefaultAzureCredential.getToken` is made

#### Scenario: Expired or near-expiry token triggers refresh

- GIVEN a cached token whose `expiresOnTimestamp` is within 5 minutes of the
  current time (or has already passed)
- WHEN `getIdentityToken` is called
- THEN a new token is acquired by calling `DefaultAzureCredential.getToken`
  with the scope `"https://ai.azure.com/.default"`
- AND the new token is stored in `cachedToken`
- AND the new token string is returned

#### Scenario: No cached token on first call

- GIVEN `cachedToken` is `null` (extension just loaded, no prior acquisition)
- WHEN `getIdentityToken` is called
- THEN `DefaultAzureCredential.getToken` is called with scope
  `"https://ai.azure.com/.default"`
- AND the result is stored in `cachedToken` and returned

### Requirement: Identity Token Acquisition Failure

The system SHALL throw a descriptive error if `DefaultAzureCredential.getToken`
returns a falsy result, preventing silent failures with empty tokens.

#### Scenario: Credential returns null

- GIVEN `DefaultAzureCredential.getToken` resolves to `null` or `undefined`
- WHEN `getIdentityToken` is called
- THEN an error with the message `"[Azure Foundry] Failed to acquire identity token"`
  is thrown
- AND no token string is returned

### Requirement: ProviderAuth Registration

The system SHALL store a `ProviderAuth` record (containing the auth type and
token getter) in `providerAuthMap` keyed by provider ID at extension
initialization, so that the streaming layer can retrieve it per-request without
re-reading config.

#### Scenario: Auth context stored at startup

- GIVEN a successfully loaded config with any valid auth type
- WHEN the extension entry point calls `pi.registerProvider`
- THEN `providerAuthMap` already contains an entry for `"azure-foundry"` with
  the correct `type` and `getToken` function matching the configured auth mode
- AND the map is populated *before* `pi.registerProvider` is called

### Requirement: DefaultAzureCredential Instantiation

The system SHALL construct a new `DefaultAzureCredential` instance on each
cache-miss call to `getIdentityToken`. The credential object is not cached
between calls; only the acquired token is cached.

#### Scenario: New credential on every cache miss

- GIVEN the cached token is absent or near expiry
- WHEN `getIdentityToken` is called
- THEN a new `DefaultAzureCredential()` is instantiated for that call
- AND `getToken` is invoked on it with scope `"https://ai.azure.com/.default"`

### Requirement: Streaming Layer Auth Resolution

The system SHALL resolve auth for each streaming request by looking up the
provider ID in `providerAuthMap`, and SHALL fall back to constructing an
api-key `ProviderAuth` from `options.apiKey` when no entry is found.

#### Scenario: Auth resolved from map

- GIVEN `providerAuthMap` contains an entry for `"azure-foundry"`
- WHEN `streamAzureFoundry` is invoked
- THEN the `ProviderAuth` from the map is used to build request headers

#### Scenario: Fallback to options.apiKey

- GIVEN `providerAuthMap` has no entry for the model's provider
- WHEN `streamAzureFoundry` is invoked
- THEN a transient `ProviderAuth` of type `"api-key"` is constructed using
  `options.apiKey` (which may be an empty string)
