# Project handbook

This handbook describes **backstage-cluster**, the ECS cluster that runs the
Backstage application for the `backstage` system in the `platform` domain.
Keep it current as the cluster's infrastructure, integration contract, and workflows change.

- [Features](01-features.md): ECS cluster, shared task execution role, shared container security group, load balancer, DNS and TLS, and the SSM integration contract.
- [Relationship to the cluster template](02-relationship-to-template.md): how this repository differs from a cluster created with the **AWS ECS Cluster** template.
- [Configuration](03-configuration.md): deployment parameters, tags, and prerequisites.
- [Architecture](04-architecture.md): AWS resources, template relationships, and repository layout.
- [CI/CD pipeline](05-ci-cd.md): build, deployment, non-main refs, and documentation publishing.
- [Observability](06-observability.md): logs, metrics, and automatic Datadog integration.
- [Common tasks](07-common-tasks.md): connect services and change cluster settings.

## Reading in Backstage

The component's `catalog-info.yaml` carries the `backstage.io/techdocs-ref`
annotation, so the catalog shows a **Docs** tab for `backstage-cluster`.
The site is built outside Backstage: `.github/workflows/techdocs-docs.yml`
builds the `docs/` directory and publishes it to the TechDocs S3 bucket. See
[CI/CD pipeline](05-ci-cd.md#documentation-publishing).

The repository has no `mkdocs.yml`. The workflow generates one, so the site
navigation follows the file names in `docs/handbook/` and the landing page is a
generated title page. Add a repository-root `mkdocs.yml` to control the
navigation; the workflow uses it when present.

The pages can also be read directly in GitHub. Paths shown in backticks are
relative to the repository root.

## Content review

This handbook was written from the repository's files and from the
`backstage-app` and `backstage-db` repositories that consume the cluster. It
was not checked against the deployed AWS account. Review its statements about
network exposure, deployment assumptions, and the integration contract before
treating it as the cluster's operational reference.
