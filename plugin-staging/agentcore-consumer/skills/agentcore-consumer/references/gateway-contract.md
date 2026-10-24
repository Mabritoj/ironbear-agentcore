# Gateway Contract

What a consumer needs from an AgentCore Gateway stack, and how to import it. This is generic: it works
with any gateway stack that publishes the exports below under a common prefix. Gateway-specific facts
(the actual prefix, auth, known gaps) live in a per-gateway reference such as `ironbear-gateway.md`.

## Export contract

A gateway stack publishes CloudFormation exports named `<ExportPrefix>-<Name>`. The prefix is whatever the
gateway stack's authors chose (commonly its stack name).

| Export suffix | Value | Required | Consumer use |
|---|---|---|---|
| `GatewayId` | gateway identifier (`name-xxxxxxxxxx`) | **Yes** | `GatewayIdentifier` on every `GatewayTarget` |
| `GatewayRoleArn` | role the gateway uses to call targets | **Yes** for Lambda, API Gateway and other IAM-authed targets | principal in `AWS::Lambda::Permission` or a resource policy |
| `GatewayArn` | gateway ARN | No | conditions, scoping |
| `GatewayUrl` | MCP endpoint | No | clients and smoke tests |
| `PolicyEngineId` / `PolicyEngineArn` | policy engine | Only if adding Cedar policies | `AWS::BedrockAgentCore::Policy` |

## Finding the prefix

Never guess it. In order:

1. Ask the user, or read it from the consumer's existing `samconfig.toml` / docs.
2. Discover it from the account:

```bash
aws cloudformation list-exports \
  --query "Exports[?ends_with(Name, '-GatewayId')].[Name,Value]" --output table
```

Strip the `-GatewayId` suffix to get the prefix. If several gateways appear, ask which one. Check the
region first: exports are regional.

## Importing

The prefix is a parameter, so one template deploys against any gateway (dev, prd, a sandbox):

```yaml
Parameters:
  GatewayExportPrefix:
    Type: String
    Description: Export prefix of the target AgentCore Gateway stack, for example my-gateway-dev

# in a resource:
GatewayIdentifier:
  Fn::ImportValue: !Sub ${GatewayExportPrefix}-GatewayId
```

Set the value per environment in `samconfig.toml` (`[dev]` / `[prd]` `parameter_overrides`), so selecting
the gateway is the same `--config-env` choice that selects everything else.

Use the long form `Fn::ImportValue: !Sub ...`. It is valid in the root template and in nested stacks.
`Fn::ImportValue` can't contain `!GetAtt` or `!Ref` to resources, only parameters, pseudo-parameters,
mappings and `Fn::Sub`, which is what `${GatewayExportPrefix}` is.

## Nested-stack hand-off

Plain CFN resources live in `infrastructure/agentcore-target.yaml`. The root template forwards what the
nested stack needs (it cannot see root resources), including the prefix:

```yaml
# template.yaml (root)
Resources:
  OrdersFunction:
    Type: AWS::Serverless::Function
    # ...

  AgentCoreTargetStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: infrastructure/agentcore-target.yaml
      Parameters:
        GatewayExportPrefix: !Ref GatewayExportPrefix
        FunctionName: !Ref OrdersFunction
        FunctionArn: !GetAtt OrdersFunction.Arn
```

```yaml
# infrastructure/agentcore-target.yaml
Parameters:
  GatewayExportPrefix:
    Type: String
  FunctionName:
    Type: String
  FunctionArn:
    Type: String
```

The nested stack imports the gateway values itself, so they are not routed through the root.

## Constraints

- Same AWS account and region as the gateway. Exports don't cross either boundary.
- The gateway stack must be deployed first. If an export doesn't exist, the consumer fails at
  create/update with `Export ... not found`. That means "deploy the gateway first" or "wrong prefix".
- Export names are a contract. The gateway can't rename or remove one while any consumer imports it, so
  import only what you use.
- Deleting a consumer stack removes its target cleanly. Deleting the gateway stack fails while any consumer
  still imports its exports, which is the intended guard.
- Callers authenticate to the gateway, not to targets. A consumer never handles the gateway's inbound auth;
  targets authenticate the gateway through the credential provider configured on the `GatewayTarget`.

## If the gateway doesn't publish these exports

A gateway stack that predates this contract may expose only plain outputs or nothing. Don't paste IDs or ARNs
into the consumer. Ask for the gateway stack to add exports (an `Export: Name: !Sub ${AWS::StackName}-GatewayId`
on its outputs is enough). Only as a stopgap, accept `GatewayId` and `GatewayRoleArn` as plain stack
parameters, and say so.
