---
name: agentcore-consumer
description: Guidance for building a consumer of an AWS AgentCore Gateway, meaning a service that exposes tools to agents by registering an AWS::BedrockAgentCore::GatewayTarget on a shared gateway through that gateway stack's CloudFormation exports. Covers Lambda tool targets (the default), MCP server, OpenAPI, API Gateway and AgentCore Runtime targets, the Fn::ImportValue contract, granting the gateway role invoke access, tool schemas, and the Lambda tool-handler event shape. Use this skill whenever the user wants to add a tool or target to an AgentCore gateway, mentions GatewayTarget, AgentCore Gateway, MCP tools for agents, or gateway exports, or is creating a new project meant to be called by agents through a gateway, even if they don't say "consumer".
---

# AgentCore Gateway Consumers

A consumer is any project that adds tools to a shared AgentCore Gateway. The gateway stack owns the
gateway, its IAM role, auth, policy engine and logging. A consumer owns only its tool implementation, its
`GatewayTarget`, and the permission that lets the gateway call it.

This skill is generic. It works against any gateway stack that publishes the export contract in
`gateway-contract.md`. Facts about one specific gateway live in its own reference file.

## Reference files

| Task | Reference file |
|---|---|
| Export contract, finding the prefix, importing, nested-stack hand-off, limits | `references/gateway-contract.md` |
| Lambda tool target (the default choice) | `references/lambda-target.md` |
| MCP server, OpenAPI, API Gateway, AgentCore Runtime targets | `references/other-targets.md` |
| The `ironbear-agentcore` gateway specifically (prefix, auth, known gaps) | `references/ironbear-gateway.md` |

Load `gateway-contract.md` for every consumer task, plus the one target-type file that applies. Load a
gateway-specific file only when the user is targeting that gateway. Also load the `aws-sam` skill's references
when the consumer is a SAM project: layout and Lambda code conventions come from there.

## Workflow

1. **Identify the gateway.** Find its export prefix (ask, or use the discovery command in
   `gateway-contract.md`). Confirm account and region match.
2. **Pick the target type.** Default to a Lambda target unless the user already has an HTTP service.
   - New tool logic that doesn't exist yet: Lambda target.
   - Existing remote MCP server (HTTPS): MCP server target.
   - Existing REST API with an OpenAPI spec: OpenAPI target.
   - Existing API Gateway REST API in the same account: API Gateway target.
   - An AgentCore Runtime agent: HTTP target (`AgentcoreRuntime`).
3. **Import via the prefix parameter** (`GatewayExportPrefix`). Never hardcode a gateway ID, ARN, role or
   account, and never hardcode the prefix in the template.
4. **Read the matching target reference** and write the resources.
5. **Place resources per house convention.** `AWS::BedrockAgentCore::GatewayTarget` and
   `AWS::Lambda::Permission` are plain CloudFormation, not SAM-native, so they go in an
   `infrastructure/agentcore-target.yaml` nested stack, wired from the root `template.yaml` with the function
   ARN/name forwarded as parameters. Only the `AWS::Serverless::Function` stays in the root template.
6. **Validate** with `sam validate --lint`, and tell the user the deploy order: the gateway stack must
   already be deployed in the same account and region.

## Rules that are easy to get wrong

- The consumer grants access, not the gateway. Assume the gateway role has no blanket
  `lambda:InvokeFunction`. Every Lambda target needs its own `AWS::Lambda::Permission` with the imported
  `GatewayRoleArn` as principal, and the `GatewayTarget` should `DependsOn` it.
- The prefix selects the gateway. Set `GatewayExportPrefix` per environment in `samconfig.toml` so a
  `dev` consumer can't land on the production gateway by accident.
- Exports are regional and per account. A consumer must deploy to the same account and region as the gateway.
- Tool names are namespaced by the target. The agent sees `<TargetName>___<ToolName>` (three underscores),
  so keep target names short, stable and kebab-case.
- A target `Name` matches `^([0-9a-zA-Z][-]?){1,100}$`: alphanumerics and single hyphens, no underscores.
- Changing a target's `GatewayIdentifier` replaces the target. Don't make the prefix something that can drift.
- Tool descriptions are the prompt. Agents choose tools from `Description` and per-field descriptions, so
  write them for a model: what the tool does, when to use it, what each field means.
- Per house conventions: explicit names on every resource, `BACKUP: <stack name>` as the only tag (applied
  through `samconfig.toml`), and `[dev]` / `[prd]` config sections.

## Limits of the contract

The contract guarantees `GatewayId` and `GatewayRoleArn` (and optionally ARN, URL and policy engine values).
It does not give a consumer the role's name or any way to add permissions to the role. That is enough for
Lambda, API Gateway and IAM-authed targets. Outbound `OAUTH` and `API_KEY` credentials need extra permissions
on the gateway role; see `other-targets.md` and the gateway's own reference before promising those work. If a
gateway lacks something a consumer needs, the change belongs in the gateway repo, not in the consumer.

Cedar policies for a policy engine are `AWS::BedrockAgentCore::Policy` resources that reference
`PolicyEngineId`. Look up that resource's current schema before writing one; this skill does not document it.
Check whether the engine is in `LOG_ONLY` or `ENFORCE`: in `LOG_ONLY` new policies only record decisions.
