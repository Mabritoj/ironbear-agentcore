# ironbear-agentcore
AWS AgentCore setup and configuration.

Stands up an Amazon Bedrock AgentCore Gateway (MCP) with JWT inbound auth against an existing Cognito user pool, using AWS SAM.

## Layout
```
template.yaml                         # root stack: nested-stack wiring only
infrastructure/agentcore-gateway.yaml # gateway IAM role, policy engine (LOG_ONLY), Gateway with response streaming
infrastructure/gateway-logs.yaml      # CloudWatch log group + vended log delivery (APPLICATION_LOGS)
samconfig.toml                        # [dev] / [prd] deploy config
```

## Prerequisites
AWS SAM CLI, Node 24, AWS credentials for a region where AgentCore is available.

## Commands
```
sam validate --lint
sam build
sam deploy --config-env dev     # or prd
```

Cognito is not provisioned here. Set `CognitoDiscoveryUrl` and `CognitoClientId` (your existing user pool and app client) in `samconfig.toml`.

Gateway logs go to `/aws/vendedlogs/bedrock-agentcore/gateway/APPLICATION_LOGS/<GatewayName>` (retention set by `LogRetentionInDays`). The policy engine starts empty in `LOG_ONLY` mode; add Cedar policies, review the decisions in the gateway logs, then set `PolicyEngineMode=ENFORCE`.

Stack outputs: `GatewayUrl`, `GatewayId`, `PolicyEngineId`, `LogGroupArn`.

## Calling the gateway
Get a token from your existing Cognito app client, then send MCP requests to `GatewayUrl` with `Authorization: Bearer <token>`.

## Adding a tool target from another repo
This stack exports, per environment (`<env>` is `dev` or `prd`):

| Export | Use |
|---|---|
| `ironbear-agentcore-<env>-GatewayId` | `GatewayIdentifier` on the target |
| `ironbear-agentcore-<env>-GatewayArn` | scoping conditions |
| `ironbear-agentcore-<env>-GatewayRoleArn` | principal in the Lambda permission |
| `ironbear-agentcore-<env>-GatewayUrl` | clients / smoke tests |
| `ironbear-agentcore-<env>-PolicyEngineId` / `-PolicyEngineArn` | attaching Cedar policies |

The gateway role has no blanket invoke rights. Each consumer grants it access on its own function:

```yaml
# consumer template.yaml
Parameters:
  Environment:
    Type: String

Resources:
  GatewayInvokePermission:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref MyToolFunction
      Action: lambda:InvokeFunction
      Principal:
        Fn::ImportValue: !Sub ironbear-agentcore-${Environment}-GatewayRoleArn

  MyToolTarget:
    Type: AWS::BedrockAgentCore::GatewayTarget
    DependsOn: GatewayInvokePermission
    Properties:
      Name: my-tool
      GatewayIdentifier:
        Fn::ImportValue: !Sub ironbear-agentcore-${Environment}-GatewayId
      # TargetConfiguration (Lambda ARN + tool schema) and
      # CredentialProviderConfigurations (gateway IAM role) go here.
```

Export names are a contract: they can't be renamed or removed while any consumer stack imports them.
