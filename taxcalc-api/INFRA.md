# taxcalc-api CloudFormation infrastructure

Week 6 Day 3: four CloudFormation stacks in the GitOps repo, validated against
**Floci** (`profile=floci`, `endpoint=http://localhost:4566`, `region=us-east-1`).
This is not real AWS. Emulator gaps are called out explicitly and are **not**
used as a reason to weaken templates.

## 1. Stacks

| Stack | Template | Role |
| --- | --- | --- |
| `taxcalc-bootstrap-dev` | `cfn/taxcalc-bootstrap-dev.yaml` | Bootstrap S3 bucket + GitHub OIDC deploy role `taxcalc-api-cfn-deploy` |
| `taxcalc-network-dev` | `cfn/taxcalc-network-dev.yaml` | Three-AZ VPC, public/private subnets, NAT, application security group |
| `taxcalc-app-dev` | `cfn/taxcalc-app-dev.yaml` | PostgreSQL RDS, DB subnet group, DB security group |
| `taxcalc-artifacts-dev` | `cfn/taxcalc-artifacts-dev.yaml` | Hardened application artifact bucket |

Observed Floci statuses at documentation time:

- `taxcalc-bootstrap-dev` = `CREATE_COMPLETE`
- `taxcalc-network-dev` = `UPDATE_COMPLETE`
- `taxcalc-app-dev` = `CREATE_COMPLETE`
- `taxcalc-artifacts-dev` = `CREATE_COMPLETE`

## 2. Safe deployment order and dependencies

Deploy with ChangeSets only (`create-change-set` then `execute-change-set`).
Do not use `aws cloudformation deploy`.

```
1. taxcalc-bootstrap-dev
2. taxcalc-network-dev          (no stack imports; exports VpcId, subnets, AppSgId)
3. taxcalc-artifacts-dev       (independent of network/app)
4. taxcalc-app-dev              (imports network exports; secret taxcalc/dev/db-master must exist)
```

`taxcalc-artifacts-dev` does not import the network stack. It can be created
after bootstrap. `taxcalc-app-dev` **must** wait until `taxcalc-network-dev`
exports exist.

Out-of-band prerequisite for the app stack: Secrets Manager secret
`taxcalc/dev/db-master` (JSON keys `username` and `password`). CloudFormation
does not create this secret.

## 3. ChangeSet workflow

Every create or update:

1. `aws cloudformation create-change-set` (`CREATE` or `UPDATE`)
2. wait until the ChangeSet is `CREATE_COMPLETE` / `AVAILABLE`
3. `aws cloudformation describe-change-set` — review resource Action and Replacement
4. execute only if the preview is non-destructive
5. `aws cloudformation execute-change-set`
6. wait for `CREATE_COMPLETE` or `UPDATE_COMPLETE`

All AWS CLI calls for this cohort use:

```
--profile floci \
--endpoint-url http://localhost:4566 \
--region us-east-1
```

CI (`.github/workflows/cfn-validate.yml`) runs **static** `cfn-lint` and
`cfn-nag` on GitHub-hosted `blacksmith-2vcpu-ubuntu-2204` runners. Those
runners cannot reach localhost Floci. The workflow does **not** invent a
fake real-AWS role ARN. `aws cloudformation validate-template` is a local
Floci check.

## 4. Network exports

Stack `taxcalc-network-dev` exports (names resolve as `${AWS::StackName}-…`):

| Export | Meaning |
| --- | --- |
| `taxcalc-network-dev-VpcId` | VPC ID |
| `taxcalc-network-dev-PublicSubnets` | Comma-joined public subnet IDs (A,B,C) |
| `taxcalc-network-dev-PrivateSubnets` | Comma-joined private subnet IDs (A,B,C) |
| `taxcalc-network-dev-AppSgId` | Application security group ID |

Optional helper: `taxcalc-network-dev-VpcCidr`.

Network design (template): DNS support and hostnames on; three public subnets
(`MapPublicIpOnLaunch: true`) and three private subnets via `!GetAZs` / `!Select`
and `!Cidr`; one NAT Gateway in **dev**; NAT per AZ when `EnvName` is staging
or prod (`IsProdLike`). App SG ingress is TCP 8080 from the VPC CIDR only.

## 5. App stack cross-stack imports

`NetworkStackName` defaults to `taxcalc-network-dev`. The app stack imports:

- `${NetworkStackName}-VpcId` — DB security group VPC
- `${NetworkStackName}-AppSgId` — DB SG ingress source (TCP 5432 only)
- `${NetworkStackName}-PrivateSubnets` — DB subnet group

No subnet IDs or security group IDs are hardcoded.

### RDS subnet IDs (final solution)

`PrivateSubnets` is a comma-separated export. AWS-valid CloudFormation allows
`Fn::Split` of `Fn::ImportValue` as a list. Floci did **not** pass that list
into `CreateDBSubnetGroup` (`The request must contain the parameter SubnetIds`).

A literal subnet-ID diagnostic (uncommitted, deleted afterward) proved Floci
RDS accepts `SubnetIds` when no intrinsic expression is involved.

**Final template (AWS-valid, still `!ImportValue`, no hardcoded IDs):**

```yaml
SubnetIds:
  - !Select [0, !Split [",", {Fn::ImportValue: !Sub "${NetworkStackName}-PrivateSubnets"}]]
  - !Select [1, !Split [",", {Fn::ImportValue: !Sub "${NetworkStackName}-PrivateSubnets"}]]
  - !Select [2, !Split [",", {Fn::ImportValue: !Sub "${NetworkStackName}-PrivateSubnets"}]]
```

Bare `Fn::Split` of the imported string is still correct on real AWS. The
`!Select` 0/1/2 form is equivalent and is what Floci required to populate
`SubnetIds`. After that change, `taxcalc-app-dev` reached `CREATE_COMPLETE`.

DB SG: TCP 5432 from imported `AppSgId` only; no `0.0.0.0/0` ingress. RDS is
not publicly accessible; storage is encrypted.

## 6. Secrets

- Secret name: `taxcalc/dev/db-master`
- Created **out-of-band** in Floci (not a CloudFormation-managed secret)
- Password is **not** in Git
- No `NoEcho` password Parameter
- RDS uses Secrets Manager dynamic references:

```
{{resolve:secretsmanager:taxcalc/dev/db-master:SecretString:username}}
{{resolve:secretsmanager:taxcalc/dev/db-master:SecretString:password}}
```

## 7. S3 hardening

Bootstrap (`taxcalc-cfn-bootstrap-dev-${AWS::AccountId}`) and artifacts
(`taxcalc-artifacts-dev-${AWS::AccountId}`) templates both specify:

- all four `PublicAccessBlock` settings `true`
- default SSE-KMS `alias/aws/s3`
- versioning enabled
- lifecycle (bootstrap: expire noncurrent versions; artifacts: STANDARD_IA at
  90 days and GLACIER_IR at 365 days)
- bucket policy Deny when `aws:SecureTransport` is `false`
- `DeletionPolicy: Retain` and `UpdateReplacePolicy: Retain` on the buckets

**Template vs Floci:** the templates keep this posture. Live Floci often does
not persist PAB, SSE-KMS (reports AES256), lifecycle, or the TLS-deny policy.
Versioning **was** observed as `Enabled`. Templates were not weakened for the
emulator.

## 8. Drift detection

Floci returned `UnknownAction` for:

- `DetectStackDrift`
- `DescribeStackDriftDetectionStatus`
- `DescribeStackResourceDrifts`
- `DetectStackResourceDrift`

**DRIFTED → IN_SYNC was not demonstrated.** Do not claim a successful local
drift experiment.

## 9. UPDATE ChangeSet discrepancy

Task 4C added a harmless VPC tag `ManagedBy=cloudformation` and created UPDATE
ChangeSet `w6d3-update-test`.

Preview (`describe-change-set`):

- resource: `Vpc` (`AWS::EC2::VPC`)
- Action: `Modify`
- Replacement: `False`

After `execute-change-set`, the stack reached `UPDATE_COMPLETE`, but Floci
**replaced** the VPC and dependent resources (new VPC, subnet, NAT, and App SG
physical IDs). That live replacement **differs from the ChangeSet preview**.
Treat this as an emulator discrepancy, not as proof of in-place update on
real AWS.

## 10. cfn-author

- Curriculum Skill `/cfn-author taxcalc --region us-east-1` is **unavailable**
- No `SKILL.md` for `cfn-author` in this environment
- **Not executed** (not fabricated)

Manual audit below substitutes for the Skill.

## Manual audit

### Accepted

**Both `DeletionPolicy: Retain` and `UpdateReplacePolicy: Retain` on stateful
resources** (bootstrap bucket, artifact bucket, RDS instance). Deletion and
replacement must both protect data.

Also verified in templates:

- OIDC audience uses `StringEquals` (`token.actions.githubusercontent.com:aud: sts.amazonaws.com`)
- IAM **Allow** actions are explicitly enumerated (CloudFormation, S3 object/bucket, `iam:PassRole`)
- No Allow `Action: "*"`

(`DenyInsecureTransport` uses `s3:*` on a **Deny** with `aws:SecureTransport=false`,
which is the TLS-only control, not a broad Allow.)

### Rejected / avoided

**NoEcho password Parameter.** Use Secrets Manager dynamic references instead.
Passwords are not committed; the master secret is out-of-band.

cfn-nag **warnings** (for example W35 no access logging, W33 public
`MapPublicIpOnLaunch`, W5 HTTPS egress, W60 no VPC flow log, W28 explicit
names) are **not suppressed**. FAIL findings were fixed; warnings remain visible.
