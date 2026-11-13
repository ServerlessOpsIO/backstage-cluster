# Documentation

platform / backstage / backstage-cluster

Backstage ECS Cluster

This component is the shared ECS cluster for the services of the **backstage**
system. It provides a shared HTTPS load balancer at
`platform.serverlessops.io`, a shared task execution role, a shared container
security group, and the SSM parameters that service components read to attach
to it. It contains no application code.

## Where to start

- **Running a service in this cluster:** read
  [Connect an ECS service](handbook/07-common-tasks.md#connect-an-ecs-service).
- **Deploying the cluster for the first time:** complete the
  [deployment prerequisites](handbook/03-configuration.md#deployment-prerequisites).
- **Investigating a problem:** follow the
  [investigation checklist](handbook/06-observability.md#investigation-checklist).
