# Config Specification

## Purpose

The config domain is responsible for locating, loading, and parsing the
`azure-foundry.config.json` file that supplies the Azure AI Foundry resource
coordinates and authentication method to the extension. It enforces a two-location
search order (project root first, global `~/.pi/` fallback) and surfaces a clear
error when no config file is found.

## Requirements

### Requirement: Search Order

The system SHALL search for `azure-foundry.config.json` in the following order,
loading the first file found:

1. `<cwd>/azure-foundry.config.json` (project-specific)
2. `~/.pi/azure-foundry.config.json` (global fallback)

#### Scenario: Project-root config takes precedence

- GIVEN a file exists at `<cwd>/azure-foundry.config.json`
- AND a file also exists at `~/.pi/azure-foundry.config.json`
- WHEN the extension initializes
- THEN the project-root file is loaded
- AND the global file is ignored

#### Scenario: Global fallback used when no project config

- GIVEN no file exists at `<cwd>/azure-foundry.config.json`
- AND a file exists at `~/.pi/azure-foundry.config.json`
- WHEN the extension initializes
- THEN the global file is loaded

#### Scenario: Project-root config used when global is absent

- GIVEN a file exists at `<cwd>/azure-foundry.config.json`
- AND no file exists at `~/.pi/azure-foundry.config.json`
- WHEN the extension initializes
- THEN the project-root file is loaded

### Requirement: Missing Config Error

The system SHALL throw a descriptive error when no config file is found at either
candidate location.

#### Scenario: No config file found

- GIVEN no file exists at `<cwd>/azure-foundry.config.json`
- AND no file exists at `~/.pi/azure-foundry.config.json`
- WHEN the extension initializes
- THEN an error is thrown containing the message `azure-foundry.config.json not found`
- AND the error lists both paths that were checked
- AND the error instructs the user where to create the file

### Requirement: Config Load Logging

The system SHALL log the resolved path of the config file to stdout when a file is
successfully located.

#### Scenario: Config found at project root

- GIVEN a file exists at `<cwd>/azure-foundry.config.json`
- WHEN the extension initializes
- THEN a message is printed to stdout containing `Loading config from:` followed by
  the absolute path of the loaded file

### Requirement: Required Config Fields

The system SHALL expect the following fields in the config file:

- `resourceId` — the Azure AI Services resource name
- `projectId` — the Azure AI Foundry project name
- `auth` — an authentication configuration object

Note: `loadConfig` performs no explicit field validation. Missing or malformed
fields are not caught at load time; they surface as downstream errors (e.g., a
broken endpoint URL or a crash inside `makeTokenGetter`).

#### Scenario: Valid api-key config accepted

- GIVEN a config file containing `resourceId`, `projectId`, and
  `auth: { type: "api-key", apiKey: "<value>" }`
- WHEN the extension loads the config
- THEN the config is returned with all fields populated

#### Scenario: Valid azure-identity config accepted

- GIVEN a config file containing `resourceId`, `projectId`, and
  `auth: { type: "azure-identity" }`
- WHEN the extension loads the config
- THEN the config is returned with all fields populated

### Requirement: Malformed JSON Error

The system SHALL propagate a `SyntaxError` from `JSON.parse` if the config file
contains invalid JSON. No `[Azure Foundry]` prefix or user-friendly wrapping is
applied; the raw parse error is thrown.

#### Scenario: Invalid JSON in config file

- GIVEN the config file at the resolved path contains malformed JSON
- WHEN the extension initializes
- THEN a `SyntaxError` is thrown by `JSON.parse`
- AND no `[Azure Foundry]`-prefixed error message is included

### Requirement: Auth Config Variants

The system SHALL support exactly two authentication types in the `auth` field:

- `api-key`: requires an `apiKey` string field containing the static API key
- `azure-identity`: requires no additional fields; uses `DefaultAzureCredential`
  at request time

#### Scenario: api-key auth type

- GIVEN a config with `auth.type` set to `"api-key"` and a non-empty `apiKey` value
- WHEN `makeTokenGetter` is called with this config
- THEN it returns a function that always resolves to the static `apiKey` string

#### Scenario: azure-identity auth type

- GIVEN a config with `auth.type` set to `"azure-identity"`
- WHEN `makeTokenGetter` is called with this config
- THEN it returns the `getIdentityToken` function backed by `DefaultAzureCredential`
