# Unity Gateway public source map

Use these public sources to verify status, examples, limits, and API fields. Do not rely on roadmap dates that are not in public release notes.

## Product and release status

- [Unity Gateway overview](https://docs.databricks.com/aws/en/ai-gateway/) — concepts and feature index.
- [Unity Gateway release notes](https://docs.databricks.com/aws/en/release-notes/unity-gateway/) — GA/Beta status and minimum developer-tool versions.
- [Supported regions](https://docs.databricks.com/aws/en/machine-learning/model-serving/model-serving-limits) — model-serving feature availability and limits.

## Create and query services

- [Create and manage model services](https://docs.databricks.com/aws/en/ai-gateway/create-model-services) — REST, CLI, SDK, and Terraform examples.
- [Query model services](https://docs.databricks.com/aws/en/ai-gateway/query-model-services) — unified APIs, native APIs, request tags, and `ai_query`.
- [Create and manage model provider services](https://docs.databricks.com/aws/en/ai-gateway/create-model-provider-services) — provider configuration and credential modes.
- [Query model provider services](https://docs.databricks.com/aws/en/ai-gateway/query-model-provider-services) — provider header, managed paths, passthrough, and forwarding.
- [Register an external MCP server](https://docs.databricks.com/aws/en/ai-gateway/register-mcp-service) — connections, permissions, tool selection, invocation, and management examples.

## Runtime behavior

- [Open Responses API](https://docs.databricks.com/aws/en/machine-learning/model-serving/query-open-responses-models) — stateless conversations, provider behavior, tool results, structured output, and multimodal input. This page's example uses the legacy serving path; use the model-service query page for the Unity Gateway URL.
- [Reasoning models](https://docs.databricks.com/aws/en/machine-learning/model-serving/query-reason-models) — reasoning controls and `encrypted_content`.
- [Rate limits](https://docs.databricks.com/aws/en/ai-gateway/rate-limits) — supported scopes, precedence, limits, 429 handling, and distributed enforcement.
- [Routing and fallbacks](https://docs.databricks.com/aws/en/ai-gateway/configure-traffic-splitting) — traffic splits, session affinity, fallback order, and limits.
- [Inference tables](https://docs.databricks.com/aws/en/ai-gateway/inference-tables) — setup, schema, delay, payload limits, storage requirements, and missing error rows.
- [Usage tracking](https://docs.databricks.com/aws/en/ai-gateway/usage-tracking) — `system.ai_gateway.usage`, request tags, routing information, dashboard, and pricing.
- [Budgets](https://docs.databricks.com/aws/en/ai-gateway/budgets) — spend thresholds, blocking, external-model spend Beta, and limitations.

## Management APIs and infrastructure as code

- [Unity Gateway API reference](https://docs.databricks.com/api/workspace/aigateway) — GA REST contract, resource names, scopes, update masks, pagination, and etags.
- [Terraform model service](https://registry.terraform.io/providers/databricks/databricks/latest/docs/resources/ai_gateway_model_service) — model destinations, routing, rate limits, and inference-table fields.
- [Terraform model provider service](https://registry.terraform.io/providers/databricks/databricks/latest/docs/resources/ai_gateway_model_provider_service) — providers, credentials, targets, passthrough, and forwarding.
- [Terraform MCP service](https://registry.terraform.io/providers/databricks/databricks/latest/docs/resources/ai_gateway_mcp_service) — source connections, tool selectors, and rate limits.
- [Terraform provider source examples](https://github.com/databricks/terraform-provider-databricks/tree/main/docs/resources) — versioned Markdown for all three resources.
- [Terraform issue #5976](https://github.com/databricks/terraform-provider-databricks/issues/5976) — open issue for updates to imported built-in `system.ai.*` model services.

## Migration and clients

- [Public migration guide](https://kb.databricks.com/unity-catalog/migration-guide-moving-to-unity-ai-gateway) — workload inventory, client changes, permissions, and configuration re-creation. Its `ucode` and external-budget sections lag the release notes.
- [Coding-agent integration](https://docs.databricks.com/aws/en/ai-gateway/coding-agent-integration-model-services) — `ug`, manual client setup, supported agents, and OpenTelemetry.
- [Unity Gateway CLI](https://github.com/databricks/unity-gateway) — installation and command source.
- [Databricks CLI](https://docs.databricks.com/aws/en/dev-tools/cli/install) — CLI installation.
- [Python SDK](https://docs.databricks.com/aws/en/dev-tools/sdk-python) — Python SDK setup.
- [Go SDK](https://docs.databricks.com/aws/en/dev-tools/sdk-go) — Go SDK setup.
- [Java SDK](https://docs.databricks.com/aws/en/dev-tools/sdk-java) — Java SDK setup.
- [JavaScript SDK](https://www.npmjs.com/package/@databricks/sdk-aigateway) — package and version history.
- [Declarative Automation Bundles](https://docs.databricks.com/aws/en/dev-tools/bundles/) — bundle concepts and current release stage.
