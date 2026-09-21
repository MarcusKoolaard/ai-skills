---
name: unity-gateway
description: Create, configure, and query Databricks Unity Gateway model services, model provider services, and MCP services. Use for Unity Catalog-governed LLM endpoints, external providers, routing, rate limits, request tags, inference tables, REST APIs, SDKs, CLI, Terraform, DABs, or coding-agent integration.
---

# Unity Gateway

Unity Gateway is GA. Use Unity Catalog services for new workloads.

Management APIs and developer tools are GA for model, model provider, and MCP services. Declarative Automation Bundles support is Beta. Some runtime features, including the unified Responses surface and service policies, have separate release stages. Check the [release notes](https://docs.databricks.com/aws/en/release-notes/unity-gateway/) before you state a status.

`references/public-sources.md` maps topics to public sources.

## Mental model

- Services are Unity Catalog securables with three-part names: `catalog.schema.service`.
- Main resource types are `MODEL_SERVICE`, `MODEL_PROVIDER_SERVICE`, and `MCP_SERVICE`.
- Model services and model provider services share one name space in a schema.
- The URL selects the request format. For direct provider-service calls, the `Databricks-Model-Provider-Service` header selects the provider service.
- Ready-to-use Databricks model services are in `system.ai`.

## Query paths

| Base path | Target | Model value |
|---|---|---|
| `/ai-gateway/mlflow/v1` | Model service across supported providers | Three-part service name |
| `/ai-gateway/openai/v1` | OpenAI-family model service | Three-part service name |
| `/ai-gateway/openai/v1` plus provider header | OpenAI-compatible provider service | Provider-side model name |
| `/ai-gateway/anthropic/v1/messages` | Claude model or provider service | Service or provider-side model name |
| `/ai-gateway/gemini/v1beta/models/{model}:generateContent` | Gemini model or provider service | Service or provider-side model name |

Use `/ai-gateway/mlflow/v1/chat/completions` as the stable unified chat surface.
Do not send the provider-service header to the MLflow path. Direct provider-service calls require a provider-native path.

`/ai-gateway/mlflow/v1/responses` is the Beta, cross-provider Responses surface. It can require the Supervisor API preview. It is stateless: send the full `input` on each turn. The native `/ai-gateway/openai/v1/responses` path is separate and provides OpenAI-specific features.

## Authentication

- Use the workspace URL and `/ai-gateway/` path for inference.
- Use a token with the least-privilege `ai-gateway` scope.
- A `403` that requires `all-apis` usually means the client uses a deprecated regional `*.ai-gateway.*` host. Move it to the workspace URL.
- Management calls under `/api/2.1/unity-catalog/` use the `unity-catalog` API scope.
- For an external application, prefer OAuth machine-to-machine authentication with a service principal. The Databricks SDK can exchange and refresh tokens from `DATABRICKS_HOST`, `DATABRICKS_CLIENT_ID`, and `DATABRICKS_CLIENT_SECRET`.

## Python query example

```python
import json
import os

from databricks.sdk import WorkspaceClient
from openai import OpenAI

host = os.environ["DATABRICKS_HOST"].rstrip("/")
w = WorkspaceClient()
token = w.config.authenticate()["Authorization"].removeprefix("Bearer ")

client = OpenAI(api_key=token, base_url=f"{host}/ai-gateway/mlflow/v1")
response = client.chat.completions.create(
    model="system.ai.claude-sonnet-4-5",
    messages=[{"role": "user", "content": "Why govern LLM traffic?"}],
    max_tokens=200,
    extra_headers={
        "Databricks-Ai-Gateway-Request-Tags": json.dumps(
            {"application": "support-bot", "environment": "production"}
        )
    },
)
print(response.choices[0].message.content)
```

## Request tags

Set `Databricks-Ai-Gateway-Request-Tags` to a JSON object of string keys and values.

Tags are stored in the `request_tags` column of the service inference table and `system.ai_gateway.usage`. Use them for end-user, project, tenant, or environment attribution when a shared service principal sends requests.

## Permissions

| Action | Required privileges |
|---|---|
| Query a service | `EXECUTE` on the service, plus `USE CATALOG` and `USE SCHEMA` |
| Create a model service | `CREATE SERVICE`, `USE CATALOG`, `USE SCHEMA`, and `EXECUTE` on each destination |
| Create an MCP service | `CREATE SERVICE`, `USE CATALOG`, `USE SCHEMA`, and `USE CONNECTION` on its connection |
| Enable inference logging | `MANAGE` on the service, plus `CREATE TABLE`, `USE CATALOG`, and `USE SCHEMA` on the target |
| Update or delete a service | Owner or `MANAGE`, plus `USE CATALOG` and `USE SCHEMA` |

A workspace admin does not automatically have Unity Catalog privileges.

## Rate limits

- Model services support request and token limits. MCP services use request limits.
- Supported scopes include service, default user, user, user group, and service principal. Check the current API schema for additional scopes.
- A custom limit overrides the default user limit. A user limit takes priority over a group limit.
- If request and token limits both apply, the more restrictive limit wins.
- Maximum: 20 limits per service and 5 group-specific limits.
- An exceeded limit returns HTTP 429. Add exponential backoff.
- Enforcement is distributed and usage is counted after responses. Short bursts above a limit are expected; the long-term average converges to the configured limit.

## Inference tables

- Databricks creates the payload table. Do not create or rename it yourself.
- The default leaf name is `{service}_payload`; `table_name_prefix` changes the prefix.
- The target must be an external-storage catalog. Default-storage catalogs and storage private endpoints are not supported.
- Initial rows can take up to one hour. Delivery is best effort.
- Requests or responses larger than 10 MiB are served but not logged. Check `logging_error_codes`.
- Logs can be absent for 401, 403, 429, and 500 responses. Do not use the table alone to prove that a request was denied.

## Model provider services

A model provider service stores external-provider configuration and credentials. Callers authenticate only to Databricks.

For direct calls:

1. Use the provider-native managed path.
2. Set `Databricks-Model-Provider-Service: catalog.schema.service`.
3. Set `model` to the provider-side model name.

Use `allow_all_targets = false` and explicit `targets` for least privilege. Each target needs `model` and at least one `native_api_types` value. Enable `forward_unmanaged_paths` only when a required provider API has no managed path.

Unmanaged passthrough does not provide token or cost tracking, token rate limits, model access control, or service policies. `forward_headers` and `forward_query_parameters` broaden what reaches the provider; enable only when required.

When a model service routes to a provider service, the model service's limits, policies, inference table, and fallbacks apply. The provider service's gateway settings are skipped.

Estimated external-model spend is available in `system.ai_gateway.external_model_spend`. A Unity Gateway budget can include it through the **External Model Spend in Budgets** Beta preview. Estimates can differ from the provider invoice. Use a provider-service price multiplier when negotiated pricing differs from list price.

Treat provider credentials as sensitive inputs. Do not hardcode them. Terraform plaintext secret fields are input-only, but the supplied value can still exist in Terraform state. Protect the state.

## Routing and fallback

- Destination types cover pay-per-token models, external models, and provisioned-throughput endpoints. For PT, set `provisioned_throughput_config.model_serving_endpoint` to `serving-endpoints/{name}`.
- Traffic percentages must total 100. The product guide supports at most 5 split destinations. The API schema permits up to 10 primary destination entries; stay at 5 until the public documents align.
- Session affinity starts automatically when traffic splitting is configured. Only requests with a session-identifying header are pinned; other requests use the weighted split.
- Up to 5 fallback destinations run in order after eligible primary failures.
- `system.ai_gateway.usage.routing_information` records attempts and routing decisions.

## REST, CLI, SDKs, and Terraform

Minimum GA versions:

- Databricks Terraform provider 1.132.0
- Databricks CLI 1.17.0
- Python SDK 0.136.0
- Go SDK 0.178.0
- Java SDK 0.153.0
- JavaScript SDK `@databricks/sdk-aigateway` 0.19.0

Resource collections:

```text
/api/2.1/unity-catalog/model-services
/api/2.1/unity-catalog/model-provider-services
/api/2.1/unity-catalog/mcp-services
```

Create calls use `parent=schemas/{catalog}.{schema}` and a resource-specific service ID. Update calls require `update_mask`. A mask of `config` replaces the full configuration and clears omitted fields. Use granular paths such as `config.routing.destinations`, `config.routing.fallback.destinations`, `config.rate_limits`, or `config.inference_table` to preserve siblings. Use the returned `etag` for conditional updates and deletes.

Terraform resources:

- `databricks_ai_gateway_model_service`
- `databricks_ai_gateway_model_provider_service`
- `databricks_ai_gateway_mcp_service`

Write `config = { ... }` as an object. A pay-per-token destination uses `models/{catalog}.{schema}.{model}`. Use the public Terraform examples as the field contract.

Open Terraform issue [#5976](https://github.com/databricks/terraform-provider-databricks/issues/5976) reports that updates to imported built-in `system.ai.*` services can resend system-owned routing and fail. Check the issue and provider release notes before managing those services with Terraform.

## MCP services

An MCP service exposes an external MCP server through a Unity Catalog HTTP connection.

- The server must use Streamable HTTP and be reachable from the serverless compute plane.
- Service authors need `USE CONNECTION`; callers need `EXECUTE` on the MCP service.
- Do not grant callers `USE CONNECTION` unless they must bypass the service. It can bypass tool selection, service policies, and service audit controls.
- Use exact names or prefix patterns in `include_tool_selectors`. An empty list exposes all tools.

## Migration

- Legacy settings do not move to Unity Catalog services. Re-create permissions, rate limits, policies, inference logging, and routing.
- Change the base URL from `/serving-endpoints` to `/ai-gateway/mlflow/v1`.
- Change the model value from a workspace endpoint name to a three-part model service name.
- Review `system.ai` grants. Underlying model privileges do not automatically grant access to the corresponding model service.
- Validate that legacy traffic has stopped before you enable any workspace setting that disables legacy paths.
- Do not state retirement dates unless they are in current public release notes.

## Common questions

**Can `ai_query` use a custom model service?** No. It supports Databricks-provided `system.ai` model services. Only usage tracking applies; rate limits, policies, inference tables, and fallbacks do not.

**How do I use OpenAI embeddings or image APIs?** Use an OpenAI model provider service and its native managed path where available. For an unmanaged path, enable passthrough and accept the reduced governance coverage.

**How do I return a tool result with unified Open Responses?** Preserve every model output item. Add the `function_call_output` with the same `call_id`, then send the complete input again. Preserve provider fields such as Gemini `encrypted_content`.

**How do I connect a coding agent?** Install the Unity Gateway CLI with `uv tool install git+https://github.com/databricks/unity-gateway`, then run `ug claude`, `ug codex`, `ug gemini`, or another supported agent. Use `ug configure`, `ug mcp add`, and `ug usage`. The old `ucode` commands remain compatible.

Cursor uses manual setup with `/ai-gateway/cursor/v1` and a Databricks PAT. It does not use an `ug cursor` command.

**Why does a configured service fail at inference?** A model can be visible in Unity Catalog but unavailable from the workspace region. Confirm a matching service in the Unity Gateway UI and check the public model-serving availability matrix.
