# Architecture

## Infrastructure

### Logical architecture

The cluster is shared infrastructure for the services of the `backstage`
system. It provides workload placement, a public entry point, and the identity
and network access that tasks share, and publishes what service stacks need to
attach to each:

- The ECS cluster is the placement boundary. It owns no compute capacity;
  services run their tasks on Fargate or Fargate Spot.
- The shared load balancer is the single public HTTPS entry point. It
  terminates TLS for one hostname; each service attaches a path-based rule.
- The shared task execution role lets ECS pull images and write logs for any
  task in the cluster.
- The shared container security group admits traffic from the load balancer
  and identifies cluster tasks to other stacks, such as the database.
- The SSM integration contract publishes the generated identifiers, so
  consuming stacks resolve them at deploy time and never hard-code names or
  ARNs.

Services, their tasks, images, task roles, target groups, and logs belong to
the consuming service components, not to this cluster.

#### Logical architecture diagram

```mermaid
flowchart LR
    Client["Internet client"]
    Entry["Shared HTTPS entry point"]
    Default["Default fixed response"]
    Services["Consuming ECS services"]
    Cluster["ECS cluster"]
    Shared["Shared execution role and container security group"]
    Database["backstage-db"]
    Contract["SSM integration contract"]

    Client --> Entry
    Entry -->|"Path matches a service rule"| Services
    Entry -->|"No rule matches"| Default
    Services -->|"Run in"| Cluster
    Services -->|"Start and network with"| Shared
    Services -->|"Reach through the shared security group"| Database
    Contract -.->|"Read at deploy time"| Services
    Contract -.->|"Read at deploy time"| Database
```

#### Service attachment call sequence

A service stack attaches to the cluster when it deploys. Nothing in the
cluster stack changes; the service only reads the published values.

```mermaid
sequenceDiagram
    participant Pipeline as Service deployment
    participant CFN as CloudFormation
    participant SSM as SSM Parameter Store
    participant ECS as ECS cluster
    participant ALB as Shared listener

    Pipeline->>CFN: Deploy service stack with SSM parameter names
    CFN->>SSM: Resolve names under the cluster's main path
    alt A parameter is missing
        SSM-->>CFN: Parameter not found
        CFN-->>Pipeline: Deployment fails before any resource changes
    else All parameters resolve
        SSM-->>CFN: Cluster name, execution role, security group, listener ARN
        CFN->>ECS: Create service with the shared role and security group
        CFN->>ALB: Add listener rule and target group
        Note over CFN,ALB: Fails if the rule priority is already used
        CFN-->>Pipeline: Service stack complete
    end
```

Parameters are resolved when a consuming stack deploys, not continuously. A
changed value reaches a consumer only on that consumer's next deployment.

#### Request routing call sequence

The listener evaluates the rules added by services in ascending priority order
and falls back to the cluster's default action.

```mermaid
sequenceDiagram
    participant Client
    participant DNS as Route 53
    participant Listener as Shared HTTPS listener
    participant Rule as Service listener rule
    participant Task as Service task

    Client->>DNS: Resolve platform.serverlessops.io
    DNS-->>Client: Load balancer addresses from the alias record
    Client->>Listener: HTTPS request on 443
    Note over Listener: TLS ends here using the stack's ACM certificate
    alt Path matches a service rule
        Listener->>Rule: First match in priority order
        Rule->>Task: Forward over HTTP to a healthy task
        Task-->>Listener: Service response
        Listener-->>Client: HTTP response
    else Matching rule has no registered task
        Listener-->>Client: 503 Service Unavailable
    else No rule matches
        Listener-->>Client: 200 fixed response from the default action
    end
```

The fixed response is the listener's default action. It does not indicate that
any service is running or healthy. Traffic between the load balancer and tasks
is unencrypted HTTP inside the VPC.

While `backstage-app` holds a `*` rule, every request matches it and the
default action is never reached.

### Physical architecture

```mermaid
flowchart TD
    Zone["Route 53 hosted zone (existing)"]
    Record["Route 53 alias A record"]
    Certificate["ACM certificate"]
    Vpc["VPC and public subnets (existing)"]
    AlbSG["Load balancer security group: TCP 443 from anywhere"]
    ALB["Internet-facing Application Load Balancer"]
    Listener["HTTPS listener on 443"]
    Cluster["ECS cluster: FARGATE_SPOT and FARGATE"]
    Role["Task execution IAM role"]
    TaskSG["Container security group"]
    SSM["SSM parameters"]
    Services["Service stacks (separate components)"]
    Db["backstage-db stack"]

    Zone --- Record
    Record -->|"Alias target"| ALB
    Zone -.->|"DNS validation records"| Certificate
    Certificate -->|"TLS certificate"| Listener
    Vpc --- ALB
    AlbSG --- ALB
    ALB --> Listener
    AlbSG -->|"TCP 80-65535"| TaskSG
    Services -->|"Listener rules and target groups"| Listener
    Services -->|"Fargate services and tasks"| Cluster
    Services -->|"Execution role"| Role
    Services -->|"Task network interfaces"| TaskSG
    TaskSG -->|"Database port"| Db
    SSM -.->|"Resolved at deploy time"| Services
    SSM -.->|"Resolved at deploy time"| Db
```

`template.yaml` owns the cluster, execution role, both security groups, load
balancer, listener, certificate, DNS record, and SSM parameters. The VPC,
subnets, hosted zone, `/org/networking/...` SSM parameters, and CI roles are
external dependencies. Listener rules, target groups, services, tasks, and task
roles are owned by the service stacks. The database's own security group and
its rule admitting the container security group are owned by `backstage-db`.

The SSM parameter names are listed in
[Features](01-features.md#ssm-integration-contract).

### CloudFormation template

`template.yaml` is the infrastructure-as-code entry point. Its
`Transform: AWS::Serverless-2016-10-31` lets the SAM CLI validate, build, and
deploy it, but it declares no SAM resource types: every resource is a standard
CloudFormation type. The template has no `Conditions` or `Outputs`; other
stacks consume it only through SSM parameters.

| Section | Purpose |
| --- | --- |
| `Parameters` | Supplies domain/system/component metadata, the deployed ref, the hostname, the hosted zone ID, and SSM parameter names for the VPC and subnets. |
| `Resources` | Declares the cluster, execution role, two security groups and the rule between them, load balancer, listener, certificate, DNS record, and six SSM parameters. |

`Ref` and `GetAtt` connect resources without hard-coding generated names or
ARNs, and the SSM parameter names are built from the metadata parameters with
`Sub`.

#### Template resource relationships

Arrows below run from the referencing resource/configuration to its target;
they show infrastructure references, not request flow.

```mermaid
flowchart TD
    VpcId["VpcId parameter: resolves existing SSM value"]
    Subnets["VpcSubnets parameter: resolves existing SSM value"]
    ZoneId["DnsZoneId parameter"]
    Hostname["Hostname parameter"]
    Path["Domain, System, Component, CodeBranch parameters"]
    Cluster["EcsCluster: ECS cluster"]
    Role["EcsTaskExecutionIamRole: IAM role"]
    AlbSG["EcsAlbSecurityGroup: security group"]
    TaskSG["ContainerSecurityGroup: security group"]
    Ingress["ContainerAlbSecurityGroupIngress: ingress rule"]
    ALB["EcsAlb: load balancer"]
    Listener["EcsAlbListener: HTTPS listener"]
    Certificate["SiteCertificate: ACM certificate"]
    Record["SiteDnsRecord: Route 53 record"]
    ClusterSsm["EcsClusterNameSsmParam"]
    RoleSsm["EcsTaskExecutionIamRoleArn"]
    TaskSgSsm["ContainerSecurityGroupIdSsmParam"]
    ListenerSsm["EcsAlbListenerArnSsmParam"]
    AlbNameSsm["EcsAlbNameSsmParam"]
    AlbFullNameSsm["EcsAlbFullNameSsmParam"]

    AlbSG -->|"VpcId"| VpcId
    TaskSG -->|"VpcId"| VpcId
    Ingress -->|"GroupId"| TaskSG
    Ingress -->|"SourceSecurityGroupId"| AlbSG
    ALB -->|"Subnets"| Subnets
    ALB -->|"SecurityGroups"| AlbSG
    Listener -->|"LoadBalancerArn"| ALB
    Listener -->|"CertificateArn"| Certificate
    Certificate -->|"DomainName"| Hostname
    Certificate -->|"DNS validation HostedZoneId"| ZoneId
    Record -->|"Name"| Hostname
    Record -->|"HostedZoneId"| ZoneId
    Record -->|"AliasTarget"| ALB
    ClusterSsm -->|"Value"| Cluster
    RoleSsm -->|"Value"| Role
    TaskSgSsm -->|"Value"| TaskSG
    ListenerSsm -->|"Value"| Listener
    AlbNameSsm -->|"Value"| ALB
    AlbFullNameSsm -->|"Value"| ALB
    ClusterSsm -.->|"Name"| Path
    RoleSsm -.->|"Name"| Path
    TaskSgSsm -.->|"Name"| Path
    ListenerSsm -.->|"Name"| Path
    AlbNameSsm -.->|"Name"| Path
    AlbFullNameSsm -.->|"Name"| Path
```

Replacing the cluster, execution role, container security group, load
balancer, or listener changes the value of the matching SSM parameter.
Consumers keep using the old value until they redeploy, and a replaced listener
loses every service's rule. CloudFormation cannot delete the old container
security group while tasks or the database's ingress rule still use it.

Two details of the SSM resources are easy to misread:

- The logical ID `EcsTaskExecutionIamRoleArn` names an SSM parameter, not the
  role, and its value is the role name. See
  [Features](01-features.md#ssm-integration-contract).
- The `Description` properties of `ContainerSecurityGroupIdSsmParam` and
  `EcsAlbNameSsmParam` were copied from neighboring resources and do not
  describe their values. Go by the parameter name.

`cfn-parameters.json` and `cfn-tags.json` provide deployment-action inputs,
while `samconfig.toml` supplies SAM deployment defaults. See
[Configuration](03-configuration.md) for their expression syntax and external
prerequisites. Validate changes with `sam validate --lint` before deployment.

## Software

### Repository file layout

```text
catalog-info.yaml       Backstage Component entity
docs/handbook/          Handbook pages
README.md               Project entry point
template.yaml           SAM/CloudFormation cluster infrastructure
cfn-parameters.json     Deployment-action parameter expressions
cfn-tags.json           Deployment-action tag expressions
samconfig.toml          SAM validate and deploy defaults
.github/workflows/      main.yml, techdocs-docs.yml, techdocs-specs.yml
```

The repository contains no application code, container image, or tests. The
software that runs in the cluster lives in the service components that depend
on it. `backstage-app` declares the dependency in its `catalog-info.yaml`, so
it appears under this component's dependents in the catalog. `backstage-db`
consumes the container security group through SSM.
