# CI/CD pipeline

## Workflow triggers

| Workflow | Automatic trigger | Manual trigger |
| --- | --- | --- |
| `.github/workflows/main.yml` (`Main`) | Push to `main` | `workflow_dispatch` |
| `.github/workflows/techdocs-docs.yml` (`Techdocs - Docs`) | Push to any branch that changes `docs/**`, `mkdocs.yml`, or the workflow file | `workflow_dispatch` |
| `.github/workflows/techdocs-specs.yml` (`Techdocs - Specs`) | Push to any branch that changes `specs/**` or the workflow file | `workflow_dispatch` |

There is no `pull_request` trigger and no branch workflow for the
infrastructure. A push to a feature branch does not validate `template.yaml`,
so a change is first validated by the pipeline when it reaches `main`; run the
local checks in
[Common tasks](07-common-tasks.md#validate-and-deploy-changes) before merging.
Manual dispatch of `Main` accepts any ref and **deploys** it; select the ref
carefully.

The jobs run on `ubuntu-latest`. Jobs that reach AWS are granted
`contents: read` and `id-token: write`. The latter allows OIDC-based AWS role
assumption rather than committing long-lived AWS credentials.

## Build job

The build job of `Main` executes these steps in order:

1. `ServerlessOpsIO/gha-setup-workspace@v2` sets up the checked-out workspace.
2. `ServerlessOpsIO/gha-assume-aws-credentials@v1` obtains credentials using
   `AWS_CICD_ACCOUNT_ID`.
3. `aws-actions/setup-sam@v3` installs SAM.
4. `sam validate --lint` validates infrastructure with CloudFormation linting.
5. `sam build --parallel --template template.yaml` prepares the template for
   deployment. There is no code to compile or package.
6. `ServerlessOpsIO/gha-store-artifacts@v2`, with `use_aws_sam: true`, stores
   the build output for the deploy job.

There are no tests. Validation checks the template's structure only. It does
not show that the `/org/networking/...` SSM parameters exist, that the
deployment role has the needed permissions, or that the hosted zone can
validate a certificate.

## Deploy job

`deploy` depends on `build` and only proceeds after it succeeds. It uses the
GitHub environment named **`production`**.

1. The workspace action runs with `checkout_artifact: true` to retrieve the
   build artifact.
2. The credential action receives both `AWS_CICD_ACCOUNT_ID` and
   `DEPLOYMENT_ACCOUNT_ID` for build/deployment account access.
3. `ServerlessOpsIO/gha-deploy-aws-sam@v1` deploys with the target account ID
   and serialized workflow environment, variables, and secrets. It resolves
   `$secrets.ROUTE53_ZONE_ID` in `cfn-parameters.json` from those secrets.

The shared actions implement artifact handling, credential assumptions, and
parameter/tag expression resolution. Their internals are external to this
repository. Confirm the expected IAM roles, region, artifact storage, and
stack-name behavior against the shared actions' configuration.

See [Configuration](03-configuration.md) for parameter files, tags, secrets,
AWS prerequisites, and SAM defaults. GitHub environment protection rules can
require deployment approval or restrict which refs deploy.

## Deploying from other refs

`CodeBranch` is resolved from `$env.GITHUB_REF_SLUG_CS` and is part of every
SSM parameter name the stack publishes. That does not make a second ref a
separate deployment:

- `samconfig.toml` has one stack name, `backstage-cluster`.
- `cfn-parameters.json` has one fixed hostname.
- The workflow always selects the GitHub environment `production`.

Unless the shared deployment action qualifies the stack name by ref, a manual
dispatch from another ref updates **the same stack**. CloudFormation replaces
each SSM parameter with one under the new ref's path and deletes the one under
`.../main/`. Running services are unaffected, but `backstage-app` and
`backstage-db` fail their next deployment until the cluster is deployed from
`main` again.

Restrict the `production` environment to `main`, or establish and verify a
platform policy before deploying other refs:

1. Give each deployed ref a distinct stack name and hostname through the shared
   deployment action's supported configuration.
2. Use an appropriate account/environment and OIDC trust policy, with approval
   rules that prevent unintended production changes.
3. Decide how consumers select a cluster that is not published under `main`;
   their `cfn-parameters.json` files name the `main` paths.
4. Define teardown for the extra stacks. There is no branch-deletion cleanup
   workflow.

## Documentation publishing

`Techdocs - Docs` publishes this handbook. It is separate from `Main` and does
not deploy infrastructure.

1. `check` looks for Markdown under `docs/` and stops the run when there is
   none.
2. `build` copies `docs/` into a temporary MkDocs project. Because the
   repository has no `mkdocs.yml`, it generates one from the name and
   description in `catalog-info.yaml`. `docs/index.md` is the site's landing
   page. It then builds the site
   with `@techdocs/cli generate --no-docker` and `mkdocs-techdocs-core`.
3. `deploy` assumes AWS credentials and runs `@techdocs/cli publish` to the S3
   bucket named by the `TECHDOCS_BUCKET` variable, under the entity path
   `default/Component/backstage-cluster`.

The workflow has no branch filter. A push that changes `docs/` on **any**
branch publishes that branch's content to the same entity, replacing what
readers see in Backstage. Re-run the workflow from `main` to restore it.

The deploy job does not use a GitHub environment. It fails when
`TECHDOCS_BUCKET` is unset.

`Techdocs - Specs` does the same for each directory under `specs/` that has a
`spec.md`, publishing to the entity in that directory's `catalog-info.yaml`.
The repository has no `specs/` directory yet, so its runs find nothing to
build.

## Reviewing a deployment

Inspect the GitHub Actions build and deploy logs, then check the deployed
CloudFormation stack's status, events, and resources. Confirm that the SSM
parameters listed in [Features](01-features.md#ssm-integration-contract) exist
in the target account and region and hold this stack's values.

A first deployment waits for ACM to validate the certificate through DNS. A
`SiteCertificate` that stays in progress usually means `ROUTE53_ZONE_ID` names
a zone that is not authoritative for `platform.serverlessops.io`.

Then request `https://platform.serverlessops.io/`. A response shows that DNS,
the certificate, and the listener work. With `backstage-app` attached it comes
from that service; with no rule attached it is the fixed `{ "healthy": true }`
body, which never establishes that a service is healthy.

Use [Observability](06-observability.md) to investigate the cluster and its
services after deployment.
