# ironbear-agentcore
AWS AgentCore setup and configuration.

Stands up an Amazon Bedrock AgentCore Gateway (MCP) with JWT inbound auth against an existing Cognito user pool, using AWS SAM.

## Layout
```
template.yaml                         # root stack: nested-stack wiring only
infrastructure/agentcore-gateway.yaml # gateway IAM role, policy engine (LOG_ONLY), Gateway with response streaming
infrastructure/gateway-logs.yaml      # CloudWatch log group + vended log delivery (APPLICATION_LOGS)
samconfig.toml                        # deploy config (copy from samconfig.toml.example)
```

## Prerequisites
AWS SAM CLI, Node 24, AWS credentials for a region where AgentCore is available.

## Commands
```
sam validate --lint
sam build
sam deploy
```

Cognito is not provisioned here. Set `CognitoDiscoveryUrl` and `CognitoClientId` (your existing user pool and app client) in `samconfig.toml`.

Gateway logs go to `/aws/vendedlogs/bedrock-agentcore/gateway/APPLICATION_LOGS/<GatewayName>` (retention set by `LogRetentionInDays`). The policy engine starts empty in `LOG_ONLY` mode; add Cedar policies, review the decisions in the gateway logs, then set `PolicyEngineMode=ENFORCE`.

Stack outputs: `GatewayUrl`, `GatewayId`, `PolicyEngineId`, `LogGroupArn`.

## Calling the gateway
Get a token from your existing Cognito app client, then send MCP requests to `GatewayUrl` with `Authorization: Bearer <token>`.

## Adding a tool target from another repo
This stack exports:

| Export | Use |
|---|---|
| `ironbear-agentcore-GatewayId` | `GatewayIdentifier` on the target |
| `ironbear-agentcore-GatewayArn` | scoping conditions |
| `ironbear-agentcore-GatewayRoleArn` | principal in the Lambda permission |
| `ironbear-agentcore-GatewayUrl` | clients / smoke tests |
| `ironbear-agentcore-PolicyEngineId` / `-PolicyEngineArn` | attaching Cedar policies |

The gateway role has no blanket invoke rights. Each consumer grants it access on its own function:

```yaml
# consumer template.yaml
Resources:
  GatewayInvokePermission:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref MyToolFunction
      Action: lambda:InvokeFunction
      Principal:
        Fn::ImportValue: ironbear-agentcore-GatewayRoleArn

  MyToolTarget:
    Type: AWS::BedrockAgentCore::GatewayTarget
    DependsOn: GatewayInvokePermission
    Properties:
      Name: my-tool
      GatewayIdentifier:
        Fn::ImportValue: ironbear-agentcore-GatewayId
      # TargetConfiguration (Lambda ARN + tool schema) and
      # CredentialProviderConfigurations (gateway IAM role) go here.
```

Export names are a contract: they can't be renamed or removed while any consumer stack imports them.
