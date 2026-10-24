# The ironbear-agentcore Gateway

Facts specific to the gateway defined in the `ironbear-agentcore` repo. Load this only when the target
gateway is that one. Everything else in this skill is generic.

## Export prefix

`ironbear-agentcore-<env>`, where `<env>` is `dev` or `prd` (the stack name). So
`GatewayExportPrefix` is `ironbear-agentcore-dev` or `ironbear-agentcore-prd`, and the exports are
`ironbear-agentcore-dev-GatewayId`, `...-GatewayArn`, `...-GatewayRoleArn`, `...-GatewayUrl`,
`...-PolicyEngineId`, `...-PolicyEngineArn`.

```toml
# consumer samconfig.toml
[dev.deploy.parameters]
parameter_overrides = [
  "GatewayExportPrefix=ironbear-agentcore-dev",
]

[prd.deploy.parameters]
parameter_overrides = [
  "GatewayExportPrefix=ironbear-agentcore-prd",
]
```

## Behaviour of this gateway

- **Inbound auth:** Cognito JWT (an existing user pool; the gateway only accepts tokens from one configured
  app client). Consumers don't need any of it, but end-to-end tests need a token from that client.
- **Response streaming:** enabled.
- **Policy engine:** present, in `LOG_ONLY` unless the gateway repo has switched `PolicyEngineMode` to
  `ENFORCE`. New Cedar policies only record decisions until then.
- **Logs:** gateway application logs, including request and response bodies, go to a CloudWatch log
  group owned by the gateway repo (`/aws/vendedlogs/bedrock-agentcore/gateway/APPLICATION_LOGS/<gateway-name>`). Tool inputs and outputs end up there, so don't put secrets in tool arguments.
- **Role permissions:** the gateway role has no blanket invoke rights and no AgentCore Identity permissions.

## Known gaps

- The role's name is not exported, only its ARN, and the role has no AgentCore Identity or Secrets Manager
  permissions. Lambda, API Gateway and IAM-authed targets work (the consumer grants access with the ARN).
  `OAUTH` and `API_KEY` outbound credentials do not until the gateway repo adds those permissions. Raise it
  there; don't edit the role from a consumer.
