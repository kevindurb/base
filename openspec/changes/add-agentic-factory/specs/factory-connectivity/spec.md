# Spec Delta

## Purpose

Defines how the factory reaches the private Aperture model and MCP endpoints through an ephemeral Tailscale node, and the constraints that keep that access reproducible and least-privileged.

## ADDED Requirements

### Requirement: Ephemeral tailnet membership

The factory SHALL join the tailnet as an ephemeral node tagged `tag:ci` before any model or MCP call and SHALL leave the tailnet when the job ends.

#### Scenario: Connection

- **WHEN** a phase that needs the model or MCP starts
- **THEN** the workflow connects with an OAuth client and `tag:ci`, and the node is ephemeral

#### Scenario: Model-free phase

- **WHEN** the archive phase runs
- **THEN** it does not join the tailnet

### Requirement: Private model endpoint

The factory SHALL use the Aperture OpenAI-compatible endpoint at `https://ai.beaver-cloud.ts.net/v1` with the `deepseek-v4.1-flash` model and SHALL declare the model's real context and output limits.

#### Scenario: Model selection

- **WHEN** an agent session starts
- **THEN** the default model is `aperture/deepseek-v4.1-flash` and the provider endpoint is the private host

#### Scenario: Limits

- **WHEN** the model is configured
- **THEN** its 1,048,576-token context and 384,000-token output limits are declared

#### Scenario: HTTPS

- **WHEN** the endpoint is configured
- **THEN** it uses `https://`

### Requirement: Private MCP endpoint

The factory SHALL use the MCP endpoint at `https://ai.beaver-cloud.ts.net/v1/mcp` in remote Streamable HTTP mode without OAuth and SHALL expose its tools natively rather than through Code Mode.

#### Scenario: Connection

- **WHEN** an agent allowed MCP tools starts
- **THEN** it connects to the private endpoint and does not attempt OAuth

#### Scenario: Native tools

- **WHEN** the MCP server is configured
- **THEN** Code Mode is disabled so its tools appear directly to the agent

#### Scenario: HTTPS

- **WHEN** the MCP endpoint is configured
- **THEN** it uses `https://`

### Requirement: Preflight before the agent

The factory SHALL verify model reachability before starting an agent and SHALL stop the phase with a clear connectivity error when the endpoint is unreachable.

#### Scenario: Unreachable

- **WHEN** the model endpoint does not respond after Tailscale connects
- **THEN** the phase stops before the agent starts and reports the failure

#### Scenario: Slow propagation

- **WHEN** the tailnet peer is not immediately reachable
- **THEN** the preflight retries for a bounded period

### Requirement: Credentials are isolated from the agent

The factory SHALL read Tailscale credentials from repository secrets and SHALL NOT expose them to the agent step.

#### Scenario: Secret scope

- **WHEN** the agent runs
- **THEN** `TS_OAUTH_CLIENT_ID` and `TS_OAUTH_SECRET` are not present in its environment

#### Scenario: No model key

- **WHEN** the private endpoint is unauthenticated on the tailnet
- **THEN** no model API key is stored or passed

### Requirement: CI access is least-privileged

The factory's CI tag SHALL be allowed to reach only the model and MCP host on the HTTPS port and SHALL NOT reach other tailnet services or SSH.

#### Scenario: Allowed path

- **WHEN** the CI node connects to the tailnet
- **THEN** its ACL permits only `tag:ci` to the AI host on port 443

#### Scenario: Denied paths

- **WHEN** the CI node attempts another tailnet host or port
- **THEN** the connection is denied
