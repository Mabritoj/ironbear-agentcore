# ironbear-agentcore CodePipeline - setup runbook

CodePipeline (V2) that validates and deploys the SAM stack in `agentcore/` (the AgentCore Gateway and
its log delivery).

```
Source (GitHub) -> Verify -> Approve -> Deploy
                                          Deploy_Gateway  sam build + sam deploy of agentcore/template.yaml
```

- **Verify** runs `sam validate --lint` against `agentcore/template.yaml`.
- **Approve** is automatic by default; set `EnableManualApproval=true` for a manual gate.
- **Deploy** runs `sam deploy` with `--role-arn <CfnDeployRoleArn>`, so CloudFormation (not the CodeBuild
  role) creates the gateway, policy engine, gateway role and log delivery.

`agentcore/samconfig.toml` is gitignored, so the pipeline passes every `sam deploy` setting explicitly from
the pipeline stack's parameters (`StackName`, `GatewayName`, `CognitoDiscoveryUrl`, `CognitoClientId`,
`PolicyEngineMode`, `LogRetentionInDays`, `SamArtifactsBucketName`, `CfnDeployRoleArn`). Keep them in line
with your local `agentcore/samconfig.toml`.

## Prerequisites

- AWS CLI v2 and SAM CLI, credentials for the target account.
- A CodeConnections connection to this GitHub repo in the target account/region, status `Available`
  (the GitHub handshake can only be completed in the console).
- An existing SAM package bucket (`SamArtifactsBucketName`).
- An existing Cognito user pool and app client for the gateway's JWT auth.

## Step 1 - Create the CloudFormation execution role

Create the out-of-band role the pipeline passes to `sam deploy --role-arn`. See
[`prereq-iam/README.md`](../prereq-iam/README.md) for the placeholder table and why this role is made by hand.

## Step 2 - Fill in `samconfig.toml`

```bash
cp samconfig.toml.example samconfig.toml
```

Replace every `<PLACEHOLDER>`.

## Step 3 - Deploy the pipeline stack

```bash
cd codepipeline
sam validate --lint --template-file template.yaml --region us-east-1
sam deploy
```

`confirm_changeset = true`, so review the changeset before accepting.

## Step 4 - First run

A push to the source branch starts the pipeline (`SourceDetectChanges=true`). To run it now:

```bash
aws codepipeline start-pipeline-execution --name ironbear-agentcore --region us-east-1
aws codepipeline get-pipeline-state --name ironbear-agentcore --region us-east-1 --query "stageStates[].{Stage:stageName,Status:latestExecution.status}" --output table
aws logs tail /aws/codebuild/ironbear-agentcore --follow --region us-east-1
```

The first run creates the app stack and exports the `ironbear-agentcore-*` values that consumer stacks
import.

## Day-2 operations

- Change pipeline stages, compute or IAM by editing `template.yaml` and re-running `sam deploy`.
- Buildspecs under `infrastructure/buildspec/` need no redeploy: CodeBuild reads them from the source
  artifact at run time.
- Pause automatic deploys by redeploying with `SourceDetectChanges=false`; the pipeline then runs only
  on `start-pipeline-execution` or "Release change".
- Flip `PolicyEngineMode` to `ENFORCE` by changing the pipeline stack parameter and redeploying it, then
  re-running the pipeline.
- If a deploy fails on a permission, the stack event names the action. Add it to the execution role
  (`prereq-iam/permissions-policy.json`) by hand.

## Teardown

```bash
aws cloudformation delete-stack --stack-name ironbear-agentcore-pipeline --region us-east-1
```

The artifact bucket and the CodeBuild log group are `DeletionPolicy: Retain` and survive on purpose.
The app stack (`ironbear-agentcore`) is separate; remove it with `sam delete`, after any consumer stacks
that import its exports. The execution role is out-of-band; delete it by hand with
`aws iam delete-role-policy` and `aws iam delete-role`.
