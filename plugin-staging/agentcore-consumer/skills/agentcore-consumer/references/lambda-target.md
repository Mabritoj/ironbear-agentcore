# Lambda Tool Target

The default consumer: a Lambda function whose tools the gateway exposes over MCP. Combine with
`gateway-contract.md` (imports, nested-stack hand-off) and the `aws-sam` skill (project layout, Node 24 ESM,
logger, config, tests).

## Resources

In `infrastructure/agentcore-target.yaml`:

```yaml
Parameters:
  GatewayExportPrefix:
    Type: String
  FunctionName:
    Type: String
  FunctionArn:
    Type: String

Resources:
  GatewayInvokePermission:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref FunctionName
      Action: lambda:InvokeFunction
      Principal:
        Fn::ImportValue: !Sub ${GatewayExportPrefix}-GatewayRoleArn

  OrdersTarget:
    Type: AWS::BedrockAgentCore::GatewayTarget
    DependsOn: GatewayInvokePermission
    Properties:
      Name: orders
      Description: Look up and manage customer orders
      GatewayIdentifier:
        Fn::ImportValue: !Sub ${GatewayExportPrefix}-GatewayId
      CredentialProviderConfigurations:
        - CredentialProviderType: GATEWAY_IAM_ROLE
      TargetConfiguration:
        Mcp:
          Lambda:
            LambdaArn: !Ref FunctionArn
            ToolSchema:
              InlinePayload:
                - Name: get_order
                  Description: >-
                    Fetch a single order by its ID. Use when the user asks about the status,
                    contents or total of a specific order.
                  InputSchema:
                    Type: object
                    Required: [orderId]
                    Properties:
                      orderId:
                        Type: string
                        Description: The order ID, for example ORD-10042.
                  OutputSchema:
                    Type: object
                    Properties:
                      status:
                        Type: string
                        Description: Order status.
                      total:
                        Type: number
                        Description: Order total in the account currency.
```

Notes on the schema, taken from the CloudFormation reference:
- `ToolSchema` takes either `InlinePayload` (a list of `ToolDefinition`) or `S3`. Use inline for a handful
  of tools; use S3 when the schema is large or generated.
- `ToolDefinition` requires `Name`, `Description` and `InputSchema`; `OutputSchema` is optional.
- `SchemaDefinition` requires `Type` (`string | number | object | array | boolean | integer`) and supports
  `Description`, `Properties`, `Required` and `Items`. It nests, so an `array` needs `Items`.
- Property keys inside `Properties` and `Required` are your tool's argument names and are case-sensitive.
  Those keys are the one place where CloudFormation does not PascalCase your data.
- `CredentialProviderType: GATEWAY_IAM_ROLE` means the gateway calls the function with its own role, which
  is why the `Lambda::Permission` above is required.

The `LambdaArn` pattern accepts an optional qualifier (`:$LATEST`, an alias). Prefer pointing at an alias
if the function has one, and grant the permission on that alias too (`FunctionName` of the alias ARN),
otherwise the invoke is denied.

## Handler contract

Verify these two details against the current AgentCore Gateway Lambda target documentation when writing
the first handler; they come from the service's documented behavior and are the most likely thing to change.

- `event` is the tool's input arguments object directly (the fields from `InputSchema`), not an API Gateway
  or MCP envelope.
- The tool being called is in `context.clientContext.custom.bedrockAgentCoreToolName`, formatted
  `<TargetName>___<ToolName>`. One Lambda can serve several tools; route on the part after `___`.

This replaces the path/method routing in the `aws-sam` skill's API Gateway handler examples. The same ideas
carry over: thin router in `index.ts`, one file per leaf operation, a correlation ID on the logger, module-scope
SDK clients, and a try/catch at the leaf.

```typescript
// handlers/orders/index.ts - router: one Lambda, several tools
import type { Context } from 'aws-lambda';
import { Logger } from 'logger';

const TOOL_DELIMITER = '___';

export const handler = async (
  event: Record<string, unknown>,
  context: Context
): Promise<unknown> => {
  const logger = new Logger(context.awsRequestId);
  const fullName: string = context.clientContext?.custom?.bedrockAgentCoreToolName ?? '';
  const toolName = fullName.split(TOOL_DELIMITER).pop();

  switch (toolName) {
    case 'get_order':
      return (await import('./get-order.js')).handler(event, logger);
    default:
      logger.warning('Unknown tool', { fullName });
      return { error: { code: 'UNKNOWN_TOOL', message: `Unknown tool: ${toolName}` } };
  }
};
```

```typescript
// handlers/orders/get-order.ts - leaf
import { Logger } from 'logger';

export const handler = async (
  event: Record<string, unknown>,
  logger: Logger
): Promise<unknown> => {
  const orderId = event.orderId;
  if (typeof orderId !== 'string') {
    return { error: { code: 'VALIDATION_ERROR', message: 'orderId is required' } };
  }
  logger.info('Fetching order', { orderId });

  try {
    // business logic here
    return { status: 'SHIPPED', total: 42.5 };
  } catch (err) {
    logger.error('Failed to fetch order', { orderId, error: (err as Error).message });
    return { error: { code: 'INTERNAL_ERROR', message: 'Failed to fetch order' } };
  }
};
```

Keep the `{ error: { code, message } }` shape from the `aws-sam` skill. There is no HTTP status code here;
the return value is the tool result the agent reads, so make error messages useful to a model.

## Testing

- Unit tests follow the `aws-sam` skill (Jest, `aws-sdk-client-mock`). Build the `Context` with a
  `clientContext.custom.bedrockAgentCoreToolName` so the router path is covered.
- `sam local invoke` takes the colocated `<name>.event.json`, which for this target is just the arguments
  object, for example `{ "orderId": "ORD-10042" }`. It can't supply `clientContext` by default, so test
  routing in Jest and use `sam local` for leaf logic.
- A real end-to-end check needs a deployed stack: obtain a token from the Cognito app client, call the
  gateway's `tools/list`, then `tools/call` with `<TargetName>___<ToolName>`.

## Checklist

- [ ] `GatewayExportPrefix` parameter on root and nested stack, set per environment in `samconfig.toml`
- [ ] `Lambda::Permission` principal is the imported `GatewayRoleArn`; `GatewayTarget` has `DependsOn`
- [ ] Target `Name` is kebab-case with no underscores; tool names are `snake_case` or `camelCase` consistently
- [ ] Every tool and every input field has a description written for a model
- [ ] Function `Timeout` is deliberate (house default is 30s) and the handler returns errors, not throws
- [ ] Qualifier/alias in `LambdaArn` matches where the permission is granted
