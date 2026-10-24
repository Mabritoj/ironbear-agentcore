# Deployer IAM policies (proposal)

Permissions for the `ironbear` IAM user to deploy this stack and consumer (target) stacks. Three
customer-managed policies rather than one inline policy: a user's inline policies are capped at 2,048
characters in total, and a managed policy at 6,144 each.

| Policy | Covers |
|---|---|
| `ironbear-deploy-core` | CloudFormation (incl. SAM transform and the SAM managed artifact bucket), IAM roles named `ironbear-*`, `iam:PassRole` |
| `ironbear-deploy-agentcore` | AgentCore gateway / policy engine, the CloudWatch vended-log group and log delivery |
| `ironbear-deploy-lambda-targets` | Lambda functions and layers named `ironbear-*` (for consumer stacks) |

The first two are what this repo needs. The third is for consumer repos that add Lambda targets.

## Assumptions to confirm
- Account/region: replace `ACCOUNT_ID` in all three files; region is fixed to `us-east-1`.
- Naming: every stack, role and function is named `ironbear-*`. This holds for this repo (stack
  `ironbear-agentcore-prd`, role `ironbear-agentcore-prd-gateway-us-east-1`) and for nested stacks (their
  names start with the parent's). Consumer stacks, functions and layers must follow it too, or the
  policies won't match them.
- `sam deploy` uses `resolve_s3 = true`, so SAM creates its own `aws-sam-cli-managed-default` stack and
  bucket the first time.

## Apply
Run with admin credentials (the `ironbear` user can't grant itself these):

```bash
for p in core agentcore lambda-targets; do
  aws iam create-policy --policy-name ironbear-deploy-$p \
    --policy-document file://iam/ironbear-deploy-$p.json
done
aws iam attach-user-policy --user-name <ironbear-user-name> \
  --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/ironbear-deploy-core
# ...and the same for -agentcore and -lambda-targets
```

## Known risks and limits
- **Privilege escalation.** Anyone who can create roles and put inline policies on them can grant
  themselves anything. Managed-policy attachment is restricted to two named AWS policies, but
  `iam:PutRolePolicy` on `ironbear-*` roles is not, because CloudFormation needs it for the gateway and
  Lambda roles. To close this, add a permissions boundary and require it with an `iam:PermissionsBoundary`
  condition on `iam:CreateRole`.
- **Better pattern:** give CloudFormation its own service role and deploy with `sam deploy --role-arn`.
  The user then only needs CloudFormation access and `iam:PassRole` on that one role, and the broad
  permissions live on a role that only CloudFormation can assume.
- **AgentCore actions.** `bedrock-agentcore:*` is scoped to this account's gateways, policy engines and
  workload identities. I used a wildcard because I couldn't confirm the exact action list CloudFormation
  calls. Once it deploys, tighten using CloudTrail.
- **Untested.** Expect to iterate on an `AccessDenied` or two on first deploy. CloudTrail shows the exact
  denied action and resource.

## Adding future targets
Consumer stacks need their own resources' permissions on top of `ironbear-deploy-lambda-targets`. Add a
new policy per resource family rather than widening these: API Gateway (`apigateway:*` on the target API),
DynamoDB tables, Secrets Manager for OAuth or API-key credentials, and so on, each scoped to `ironbear-*`
names. Outbound OAuth/API-key targets also need AgentCore Identity actions (workload identity, credential
providers) added to the AgentCore policy.
