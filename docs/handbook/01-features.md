# Features

## ECS cluster

`template.yaml` creates one `AWS::ECS::Cluster`, the shared placement boundary
for the services of the `backstage` system. The template does not set a cluster
name, so CloudFormation generates one from the stack name and logical ID.
Services find it through the
[SSM integration contract](#ssm-integration-contract), not by a fixed name.

The cluster associates two capacity providers, `FARGATE_SPOT` and `FARGATE`.
It sets no default capacity provider strategy, so each service must name a
capacity provider strategy or a launch type itself. `backstage-app` runs on
`FARGATE_SPOT`.

The stack does **not** create ECS services, task definitions, container images,
container instances, autoscaling, or log groups. Those belong to the service
stacks.

There is no Service Connect namespace. Services in this cluster have no private
discovery name for each other.

## Shared task execution role

`EcsTaskExecutionIamRole` is one IAM role that any task in the cluster can use
as its **execution role**. ECS assumes it to start a task, not the application
code. It allows, on all resources:

- Pulling images from Amazon ECR.
- Creating CloudWatch log groups and log streams and writing log events.

It does not allow reading Secrets Manager secrets or SSM parameters, so a task
definition that injects container `secrets` needs either additions to this role
or its own execution role. Application permissions belong in each service's own
task role, not here.

## Shared container security group

`ContainerSecurityGroup` is one security group meant to be attached to every
task in the cluster. Its only ingress rule allows TCP ports 80 to 65535 from
the load balancer's security group. Outbound traffic is unrestricted.

Because every task shares it, the group also identifies "a task in this
cluster" to other stacks. `backstage-db` allows this group to reach the
database port.

The group has no rule that allows traffic from itself, so tasks cannot call
each other directly without an added rule.

## Shared Application Load Balancer

The stack owns one internet-facing Application Load Balancer shared by every
service in the cluster:

| Resource | Behavior |
| --- | --- |
| `EcsAlbSecurityGroup` | Allows inbound TCP 443 from `0.0.0.0/0`. There is no port 80 ingress. |
| `EcsAlb` | Internet-facing ALB in the subnets resolved from `/org/networking/VpcPublicSubnets`. |
| `EcsAlbListener` | HTTPS listener on 443 using `SiteCertificate` and the default ELB TLS policy. There is no HTTP listener or HTTP-to-HTTPS redirect. |

The listener's default action returns a fixed `200` response with the body
`{ "healthy": true }`. The cluster stack creates no target groups or rules.
Each service that uses the load balancer adds its own target group and a
path-based listener rule with a priority that is unique on this listener.

Any request that matches no service rule receives the fixed response, so a
successful response from `https://platform.serverlessops.io` does **not** show
that a service is running or healthy.

`backstage-app` attaches a rule with path `*` and priority `1`, which matches
every request. See [Common tasks](07-common-tasks.md#choose-listener-rule-paths-and-priorities)
before adding a second service to the listener.

## DNS and TLS

The stack creates a DNS-validated ACM certificate and a Route 53 alias `A`
record for **`platform.serverlessops.io`**, pointing at the load balancer.
Every service on the load balancer is reached through this one hostname and is
distinguished by its listener-rule path.

The hosted zone ID comes from the GitHub secret `ROUTE53_ZONE_ID`. The hosted
zone is a prerequisite, not a resource created by this stack. See
[Configuration](03-configuration.md).

## SSM integration contract

After a deployment from `main`, the stack publishes these String parameters
under `/platform/backstage/backstage-cluster/main/`. Service stacks read them
at deploy time, so they never hard-code generated names or ARNs.

| SSM parameter | Value | Read by |
| --- | --- | --- |
| `.../EcsClusterName` | Generated ECS cluster name | `backstage-app` |
| `.../EcsTaskExecutionIamRoleArn` | Shared task execution role. Holds the role **name**, not its ARN; see the note below. | `backstage-app` |
| `.../ContainerSecurityGroupId` | Shared container security group ID | `backstage-app`, `backstage-db` |
| `.../EcsAlbListenerArn` | HTTPS listener ARN | `backstage-app` |
| `.../EcsAlbName` | Load balancer name | Not read by `backstage-app` or `backstage-db` |
| `.../EcsAlbFullName` | Load balancer full name, the form used in CloudWatch metric dimensions | Not read by `backstage-app` or `backstage-db` |

The path is `/<Domain>/<System>/<Component>/<CodeBranch>/<name>`, built from
the stack parameters described in [Configuration](03-configuration.md).
Consumers write the full path, including the `main` segment, into their own
`cfn-parameters.json`. A service can therefore only consume the cluster as
deployed from `main`.

`EcsTaskExecutionIamRoleArn` is set with `Ref` on the IAM role, which returns
the role name. ECS accepts a role name for a task definition's execution role,
so `backstage-app` works. Anything that needs a real ARN, such as an IAM policy
statement, cannot use this value as it is.

Treat the paths and parameter names as an interface. Changing them is a
coordinated change with every consuming stack; see
[Common tasks](07-common-tasks.md#change-ssm-parameter-paths).

## CI/CD

`main.yml` validates and builds the SAM template, then deploys it, on every
push to `main` and on manual dispatch. There is no branch workflow, and a
deployment from any ref other than `main` moves the SSM parameters that
services depend on. See [CI/CD pipeline](05-ci-cd.md) before dispatching the
workflow manually.
