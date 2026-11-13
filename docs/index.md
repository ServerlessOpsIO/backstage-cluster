# backstage-cluster

platform / backstage / backstage-cluster

Backstage ECS Cluster

This component is the shared ECS cluster for the services of the **backstage**
system. It provides a shared HTTPS load balancer at
`platform.serverlessops.io`, a shared task execution role, a shared container
security group, and the SSM parameters that service components read to attach
to it. It contains no application code.

## Handbook

The [project handbook](handbook/00-index.md) is the reference for this cluster:

- [Features](handbook/01-features.md): what the cluster provides and the SSM integration contract.
- [Relationship to the cluster template](handbook/02-relationship-to-template.md): how this repository differs from a cluster created with the **AWS ECS Cluster** template.
- [Configuration](handbook/03-configuration.md): deployment parameters, tags, and prerequisites.
- [Architecture](handbook/04-architecture.md): AWS resources, template relationships, and repository layout.
- [CI/CD pipeline](handbook/05-ci-cd.md): build, deployment, non-main refs, and documentation publishing.
- [Observability](handbook/06-observability.md): logs, metrics, and automatic Datadog integration.
- [Common tasks](handbook/07-common-tasks.md): connect services and change cluster settings.

## Where to start

- **Running a service in this cluster:** read
  [Connect an ECS service](handbook/07-common-tasks.md#connect-an-ecs-service).
- **Deploying the cluster for the first time:** complete the
  [deployment prerequisites](handbook/03-configuration.md#deployment-prerequisites).
- **Investigating a problem:** follow the
  [investigation checklist](handbook/06-observability.md#investigation-checklist).
