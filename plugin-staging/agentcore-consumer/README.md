# agentcore-consumer

Skill for building a consumer of an AgentCore Gateway: a project (usually a SAM app) that registers its
own `AWS::BedrockAgentCore::GatewayTarget` on a shared gateway, using the gateway stack's CloudFormation
exports. Generic: it works with any gateway that publishes the export contract, selected by a
`GatewayExportPrefix` parameter. Gateway-specific facts are in `references/ironbear-gateway.md`.

Pairs with the `aws-sam` plugin: this skill covers the AgentCore wiring, `aws-sam` covers project layout
and handler code.

## Contents
- `skills/agentcore-consumer/SKILL.md` - workflow and rules
- `references/gateway-contract.md` - the generic export contract and how to import it
- `references/ironbear-gateway.md` - the ironbear-agentcore gateway: prefix, auth, known gaps
- `references/lambda-target.md` - Lambda tool target (permission, target, tool schema, handler)
- `references/other-targets.md` - MCP server, OpenAPI, API Gateway, AgentCore Runtime targets

## Install
Copy this folder to `plugins/agentcore-consumer/` in the plugins repo and add it to
`.claude-plugin/marketplace.json`:

```json
{
  "name": "agentcore-consumer",
  "description": "Build services (Lambda, MCP server, OpenAPI, API Gateway targets) that plug into an AgentCore Gateway.",
  "source": "./plugins/agentcore-consumer"
}
```

Then `claude plugin install agentcore-consumer@ironbear-plugins`.
