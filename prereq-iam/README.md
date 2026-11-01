# Prerequisite IAM: CloudFormation Execution Role (manual, out-of-band)

This folder holds the IAM policy documents for a **CloudFormation execution role** that
`sam deploy` uses (via `--role-arn`) to deploy both stacks: the app stack (`agentcore/template.yaml`,
statements prefixed `AgentCore`) and the pipeline stack (`codepipeline/template.yaml`, statements
prefixed `Pipeline`).

## Why this is not in `codepipeline/template.yaml`

Everything else in `codepipeline/` is deployed by the pipeline it defines, or by someone running
`sam deploy` against `codepipeline/template.yaml` using the latest checked-in source, so its
permissions are ultimately controlled by whatever is merged into this repo.

This role intentionally breaks that chain. It is created and updated **only** by a human running
AWS CLI commands directly, using their own credentials. It is never referenced by any buildspec or
automation in this repo.

The result: `CodeBuildDeployRole` only holds `iam:PassRole` on this one role plus the handful of
permissions it needs to call CloudFormation and S3 directly. It cannot call `iam:CreateRole`,
`bedrock-agentcore:*`, etc. itself - only CloudFormation, assuming this role, can.
Even a compromised buildspec is limited to "ask CloudFormation to deploy this stack's template".

## Files

- `trust-policy.json` - who can assume the role (CloudFormation only, scoped to this account).
- `permissions-policy.json` - what the role can do. `AgentCore*` statements cover the app stack;
  `Pipeline*` statements cover the pipeline stack.

Substitute every `<PLACEHOLDER>` in both files before use:

| Placeholder | Value | Where to find it |
|---|---|---|
| `<ACCOUNT_ID>` | Target AWS account ID | `aws sts get-caller-identity --query Account --output text` |
| `<REGION>` | Target AWS region | `us-east-1` |
| `<STACK_NAME>` | App stack name | `agentcore/samconfig.toml` `stack_name` / pipeline `StackName` |
| `<PIPELINE_STACK_NAME>` | Pipeline stack name | `codepipeline/samconfig.toml` `stack_name` |
| `<PROJECT_SHORT_NAME>` | Pipeline `ProjectShortName` | `codepipeline/samconfig.toml` (default `ironbear-agentcore`) |
| `<GITHUB_CONNECTION_ARN>` | CodeConnections ARN | pipeline `GitHubConnectionArn` |
| `<SAM_ARTIFACTS_BUCKET>` | SAM package bucket | pipeline `SamArtifactsBucketName` |
| `<CFN_DEPLOY_ROLE_NAME>` | This role's own name | the name you pass to `create-role` below |

## Scope notes

**Nested stacks need their own ARN entries.** The app stack declares `AgentCoreGatewayStack` and
`GatewayLogsStack`, which CloudFormation names `{parent}-{logicalId}-{random}`.
They are listed individually rather than as `<STACK_NAME>-*`, which would also match the pipeline's own
stack if it shares the prefix.

**Nested stacks also require self-PassRole.** The nested stacks inherit this role's identity, and a role
that creates a stack with nested stacks must hold `iam:PassRole` on itself (`PassRoleToNestedStacks`).
Update that statement if you rename the role.

**`CloudFormationSamTransform` is required** because `template.yaml` declares
`Transform: AWS::Serverless-2016-10-31`.

**AgentCore is scoped by account and region, not by resource.** `bedrock-agentcore:*` on
`arn:aws:bedrock-agentcore:<REGION>:<ACCOUNT_ID>:*` covers the gateway and policy engine. Tighten it once the first deploy has shown which actions are used.

**The gateway role name is fixed.** `GatewayRole` and `PassGatewayRoleToAgentCore` name
`ironbear-agentcore-gateway-<REGION>`, matching `RoleName` in `agentcore/infrastructure/agentcore-gateway.yaml`.
Update both if that name changes.

**Log delivery actions use `Resource: "*"`** for `DescribeLogGroups`, `PutResourcePolicy` and
`AllowVendedLogDeliveryForResource`, which do not support resource-level scoping.

**Expect to iterate.** The first deploy may surface a missing permission; the stack event names the
exact action. Add it here and re-run `put-role-policy`.

## One-time setup (per AWS account)

```bash
cd prereq-iam
aws iam create-role --role-name <CFN_DEPLOY_ROLE_NAME> --assume-role-policy-document file://trust-policy.json
aws iam put-role-policy --role-name <CFN_DEPLOY_ROLE_NAME> --policy-name <CFN_DEPLOY_ROLE_NAME>-permissions --policy-document file://permissions-policy.json
aws iam get-role --role-name <CFN_DEPLOY_ROLE_NAME> --query Role.Arn --output text
```

Put the resulting ARN in `codepipeline/samconfig.toml` as `CfnDeployRoleArn=<arn>`.

## Updating permissions later

Update `permissions-policy.json` and re-run `put-role-policy`. Do this deliberately, by hand, after
reviewing the diff - never as part of an automated pipeline step.
