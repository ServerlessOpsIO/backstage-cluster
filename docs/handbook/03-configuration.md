# Configuration

## Stack parameters: `cfn-parameters.json`

This file supplies the deployment action with values for the `Parameters` in
`template.yaml`.

| Parameter | Value/source | Meaning |
| --- | --- | --- |
| `Domain` | `platform` | Catalog domain name. First segment of the SSM integration path. |
| `System` | `backstage` | Catalog system name. Second segment of the SSM integration path. |
| `Component` | `$env.GITHUB_REPOSITORY_NAME_PART_SLUG_CS` | Repository-derived component identifier provided through the workflow environment; resolves to `backstage-cluster`. Third segment of the SSM integration path. |
| `CodeBranch` | `$env.GITHUB_REF_SLUG_CS` | Slug of the deployed ref. Fourth segment of the SSM integration path; consumers expect `main`. |
| `VpcId` | `/org/networking/VpcId` | **SSM parameter name**, not the literal VPC ID; CloudFormation resolves it using `AWS::SSM::Parameter::Value<String>`. Used by both security groups. |
| `VpcSubnets` | `/org/networking/VpcPublicSubnets` | **SSM parameter name** resolving to a comma-delimited list of public subnet IDs for the load balancer. |
| `Hostname` | `platform.serverlessops.io` | Certificate domain name and DNS record name. |
| `DnsZoneId` | `$secrets.ROUTE53_ZONE_ID` | Literal Route 53 hosted zone ID, taken from a GitHub secret. Unlike the VPC values, it is not an SSM parameter name. |

The `$env` and `$secrets` expressions are intentionally unquoted. The file is
deployment-action input, **not strict JSON** for direct consumption by
`json.load`, `jq`, or the SAM CLI. The same applies to the expression in
`cfn-tags.json`. The workflow passes `env_json`, `vars_json`, and
`secrets_json` to `ServerlessOpsIO/gha-deploy-aws-sam@v1` for resolution.
Consult that action's configuration when changing expression syntax or adding
values.

For a manual deployment, supply resolved CloudFormation parameter values through
SAM's parameter overrides; do not pass this file as though it were the AWS CLI's
parameter-list format. Use `Component=backstage-cluster` and `CodeBranch=main`
so the SSM paths match what consumers read.

`Component` comes from the repository name. Consumers hard-code
`backstage-cluster` in their SSM paths, so renaming the repository changes the
paths they read.

## Stack tags: `cfn-tags.json`

| Tag | Value |
| --- | --- |
| `org:domain` | `infra` |
| `org:system` | `backstage` |
| `org:component` | `$env.GITHUB_REPOSITORY_NAME_PART_SLUG_CS` |

`org:domain` is `infra`, while the `Domain` stack parameter and the catalog
domain are `platform`. Confirm which is intended before relying on the
`org:domain` tag for inventory or cost allocation.

CloudFormation propagates stack tags to resources that support them. These
metadata tags are not runtime configuration or access controls.

## Deployment prerequisites

- A target AWS account and deployment region with IAM permissions to create
  the ECS cluster, IAM role, SSM parameters, security groups, load balancer and
  listener, ACM certificate, and Route 53 record.
- A VPC with its ID stored in the deployment account/region's SSM parameter
  `/org/networking/VpcId`, and public subnets in at least two Availability
  Zones with their IDs stored as a comma-delimited list in
  `/org/networking/VpcPublicSubnets`.
- A Route 53 hosted zone authoritative for `platform.serverlessops.io`, with
  its ID in the GitHub secret `ROUTE53_ZONE_ID`. DNS validation must be able to
  create certificate records; the stack waits until the certificate is issued.
- GitHub Actions secrets `AWS_CICD_ACCOUNT_ID`, `DEPLOYMENT_ACCOUNT_ID`, and
  `ROUTE53_ZONE_ID`, available to the jobs that use them. The documentation
  workflow's deploy job does not run in a GitHub environment, so it cannot read
  secrets that exist only on the `production` environment.
- GitHub Actions variable `TECHDOCS_BUCKET`, for documentation publishing only.
- AWS OIDC trust and build/deployment roles expected by the shared
  `gha-assume-aws-credentials` action. Account IDs alone do not grant deployment
  access. Verify role names, trust restrictions, region, and artifact storage
  against the shared action's requirements.
- GitHub environment `production` settings appropriate for the deploy job,
  including any required approvals and branch restrictions.

## Network exposure

The load balancer is internet-facing and accepts TCP 443 from any IPv4 address.
The template does not configure AWS WAF, load balancer access logs, deletion
protection, or a specific TLS security policy. Review this exposure against
your organization's network policy.

Tasks are not directly reachable through the shared container security group
from the internet: its only ingress is from the load balancer's security group.
That rule covers TCP 80 to 65535, so the load balancer can reach any such port
on any task in the cluster, not only the port a service registers.

Where a service runs its tasks is the service's decision. `backstage-app`
places tasks in the public subnets with public IP addresses, and relies on this
security group to keep them unreachable from the internet.

## SAM settings

`samconfig.toml` sets the default stack name to `backstage-cluster`, enables
linting for `sam validate`, enables cached and parallel builds, disables
changeset confirmation, and requests `CAPABILITY_NAMED_IAM` and
`CAPABILITY_AUTO_EXPAND`. It does not define separate branch environments.
Verify the final stack name used by the shared deployment action before
deploying.

The stack name is not branch-qualified, although `CodeBranch` is part of every
SSM parameter name. See [CI/CD pipeline](05-ci-cd.md#deploying-from-other-refs)
before deploying from any ref other than `main`.
