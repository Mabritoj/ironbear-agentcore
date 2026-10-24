# Other Target Types

For targets that are not a Lambda function. All sit under `TargetConfiguration` on
`AWS::BedrockAgentCore::GatewayTarget` and are imported with the same `GatewayIdentifier` pattern from
`gateway-contract.md`. Shapes below come from the CloudFormation reference; check it for fields not shown.

`TargetConfiguration` is one of `Mcp`, `Http` or `Inference`. Under `Mcp`, exactly one of `Lambda`,
`McpServer`, `OpenApiSchema`, `SmithyModel`, `ApiGateway` or `Connector`.

## Credentials: read this first

`CredentialProviderConfigurations` takes at most one entry. `CredentialProviderType` is one of
`GATEWAY_IAM_ROLE`, `OAUTH`, `API_KEY`, `CALLER_IAM_CREDENTIALS`, `JWT_PASSTHROUGH`.

| Type | Use for | Works with the gateway as deployed today |
|---|---|---|
| `GATEWAY_IAM_ROLE` | Lambda; AWS services that accept the gateway's role | Yes, once the consumer grants the role access |
| `OAUTH` | Remote APIs/MCP servers behind OAuth | Only if the gateway role has AgentCore Identity permissions (see below) |
| `API_KEY` | Remote APIs keyed by a header/query value | Same as `OAUTH` |
| `CALLER_IAM_CREDENTIALS` | Calling with the end caller's IAM identity | Only if the gateway's inbound auth is `AWS_IAM` |
| `JWT_PASSTHROUGH` | Forward the caller's inbound JWT to the target | Only if the gateway's inbound auth is `CUSTOM_JWT` and the target trusts the same issuer |

`OAUTH` and `API_KEY` use a credential provider created in AgentCore Identity (`ProviderArn`). For the
gateway to fetch those credentials, its role needs AgentCore Identity permissions (workload access token,
resource OAuth2 token / API key retrieval) and `secretsmanager:GetSecretValue` on the provider's secret.
The gateway role belongs to the gateway stack, and the contract exports only its ARN, so a consumer
cannot attach those permissions itself. Check the gateway's reference file (for example
`ironbear-gateway.md`) for whether they are granted. If not, the gateway stack needs to grant them or
export the role name. Do not work around it by editing the role from the consumer.

## MCP server target

An existing remote MCP server reachable over HTTPS.

```yaml
TargetConfiguration:
  Mcp:
    McpServer:
      Endpoint: https://tools.example.com/mcp     # must start with https://
      ListingMode: DEFAULT                        # DEFAULT caches tools at the control plane; DYNAMIC lists live
```

- `McpToolSchema` provides a static tool list and turns off dynamic discovery; supported for all credential
  providers. Use it when the server can't be reached at deploy time.
- `ResourcePriority` (0-1000, lower wins, default 1000) resolves clashes between servers that share resource URIs.
- Authentication is almost always `OAUTH` or `API_KEY`: see the credentials section above.

## OpenAPI target

An existing REST API described by an OpenAPI document.

```yaml
TargetConfiguration:
  Mcp:
    OpenApiSchema:
      S3:
        Uri: s3://my-bucket/openapi.yaml          # or InlinePayload: a JSON/YAML string of the spec
```

`ApiSchemaConfiguration` takes `InlinePayload` (a **string**, unlike the Lambda `ToolSchema` which is
structured) or `S3`. Each operation becomes a tool, so give every operation a good `summary`/`description`
and a stable `operationId`. `SmithyModel` uses the same structure for Smithy models.

OpenAPI targets call an external endpoint, so credentials are `OAUTH` or `API_KEY` (see above).

## API Gateway target

An existing API Gateway REST API in the same account.

```yaml
TargetConfiguration:
  Mcp:
    ApiGateway:
      RestApiId: !Ref OrdersRestApi
      Stage: !Ref StageName
      ApiGatewayToolConfiguration:
        # tool filters/overrides; see the ApiGatewayToolConfiguration reference
```

`RestApiId`, `Stage` and `ApiGatewayToolConfiguration` are all required. The API must permit the gateway role
to invoke it (IAM auth on the API, with `execute-api:Invoke` granted to the imported `GatewayRoleArn` through
the API's resource policy), which is the API Gateway equivalent of the Lambda permission.

## AgentCore Runtime / HTTP target

For routing to an HTTP endpoint such as an AgentCore Runtime agent.

```yaml
TargetConfiguration:
  Http:
    AgentcoreRuntime:
      # RuntimeTargetConfiguration: runtime ARN and qualifier; see the reference for required fields
```

`Http` also supports `Passthrough` (forward to an external HTTP endpoint) and `Connector`. Look up
`RuntimeTargetConfiguration` before writing this; its required fields are not documented here.

## Choosing quickly

| Situation | Target |
|---|---|
| Writing new tool code | Lambda (`lambda-target.md`) |
| Third-party/hosted MCP server | `McpServer` |
| Internal REST API with a spec | `OpenApiSchema` |
| Existing API Gateway REST API | `ApiGateway` |
| AgentCore Runtime agent | `Http.AgentcoreRuntime` |
