# Secure, low-cost Cloud Run delivery baseline

**Research question:** What is the smallest secure path for building and delivering separate web and API containers from GitHub Actions to Cloud Run, while retaining a credible route to production operations?

**Evidence checked:** 2026-08-29. All links below are first-party Google Cloud, GitHub, OpenTofu, or official provider/action documentation. Prices and limits are a snapshot, not architectural constants.

This note does **not** choose the final architecture. It establishes constraints, a feasible delivery path, and the decisions that remain open.

## Reading the findings

- **Documented fact** means the statement is directly supported by the linked first-party documentation.
- **Current price or limit** means it was verified on 2026-08-29 and should be rechecked before implementation.
- **Inference** means it is a design conclusion drawn from those facts, not a Google or GitHub requirement.

## Executive finding

A secure low-cost baseline is feasible without storing a Google service-account key in GitHub:

1. GitHub Actions obtains a short-lived OIDC identity.
2. A tightly restricted Google Workload Identity Provider accepts identities only from this repository and an allowed release context.
3. The workflow receives narrowly scoped Google permissions, either directly or by impersonating a keyless release service account.
4. The workflow builds two images, pushes them to a regional Artifact Registry repository, and deploys immutable digests as separate Cloud Run revisions.
5. A Cloud Run Job runs database migrations and the release stops unless that job succeeds.
6. Secret Manager supplies runtime secrets; the release identity does not read secret payloads unless a concrete deployment mechanism requires it.
7. Built-in Cloud Run logs and metrics, a very small alert set, revision traffic controls, and data-store recovery provide the first operational safety net.

With request-based billing and zero minimum instances, the web and API services have no idle compute charge. At learning-scale traffic, the persistent data store is much more likely than Cloud Run itself to determine the monthly bill. Future long-lived realtime connections change that economics.

## 1. Identity and pipeline boundary

### Documented facts

GitHub Actions can request an OIDC token when a job has `id-token: write`; this permission only permits fetching an identity token and does not itself grant write access to cloud resources. Workload Identity Federation exchanges that token for short-lived Google credentials and avoids a long-lived Google credential stored as a GitHub secret. ([GitHub OIDC for GCP](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-google-cloud-platform), [Google deployment-pipeline federation](https://cloud.google.com/iam/docs/workload-identity-federation-with-deployment-pipelines))

GitHub uses one OIDC issuer across organizations. Google therefore requires an attribute condition that restricts which GitHub organization is admitted. Google also advises using immutable numeric claims such as `repository_id` and `repository_owner_id`, rather than reusable names, when mapping identities. The condition may additionally restrict a branch or workflow context. ([Google attribute mapping and conditions](https://cloud.google.com/iam/docs/workload-identity-federation-with-deployment-pipelines#define_an_attribute_mapping))

Google documents both direct federation to resource IAM policies and federation through service-account impersonation. The maintained `google-github-actions/auth` action supports both and warns that the GitHub OIDC token currently expires after five minutes. Service-account JSON keys are long-lived credentials and are explicitly the less secure option. ([Google auth action](https://github.com/google-github-actions/auth), [Google deployment-pipeline federation](https://cloud.google.com/iam/docs/workload-identity-federation-with-deployment-pipelines#authenticate_a_deployment_pipeline))

For a container deployment, the deployer needs access to three distinct resources: Cloud Run, the image repository, and the runtime service account. Google's predefined-role mapping is `roles/run.developer` on the Cloud Run resource, `roles/artifactregistry.reader` on the image repository, and `roles/iam.serviceAccountUser` on the runtime service identity. Pushing an image additionally needs `roles/artifactregistry.writer` on the repository. Google recommends a user-managed runtime service account with minimal permissions. ([Cloud Run deployment permissions](https://cloud.google.com/run/docs/reference/iam/roles#deployment_permissions), [Artifact Registry roles](https://cloud.google.com/iam/docs/roles-permissions/artifactregistry), [Cloud Run service identity](https://cloud.google.com/run/docs/configuring/services/service-identity))

GitHub recommends pinning third-party actions to a full commit SHA; this is the only action reference GitHub describes as immutable. GitHub environments can restrict deployment branches and serialize deployment intent, although required reviewers on GitHub Free, Pro, or Team are only available for public repositories. GitHub Actions concurrency can ensure only one deployment for an environment runs at a time. ([GitHub secure-use reference](https://docs.github.com/en/actions/reference/security/secure-use), [deployments and environments](https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments), [concurrency](https://docs.github.com/en/actions/concepts/workflows-and-actions/concurrency))

### Inferences for a first baseline

- Create a dedicated Workload Identity Pool/Provider for GitHub Actions. Map `sub`, `repository_id`, `repository_owner_id`, `ref`, and any claim used in conditions. Admit only this repository's numeric ID and the selected release context (for example `main` or a protected GitHub environment).
- Prefer a dedicated **release identity**, separate from all runtime identities. Service-account impersonation is the conservative compatibility path for the first iteration because its effective permissions are easy to inspect as one Google principal. Direct federation remains a valid simplification to test before the architecture is locked.
- Give each Cloud Run resource a user-managed runtime identity: at least `web-runtime`, `api-runtime`, and `migration-runtime` if their permissions differ. The release identity may act as these accounts for deployment, but must not inherit their runtime access.
- Scope Artifact Registry reader/writer grants to the repository and Secret Manager access to individual secrets. Avoid project-level Editor, Owner, Cloud Run Admin, and Secret Manager Admin in the release workflow.
- Put `id-token: write` only on jobs that need Google access. Use `contents: read`, pin external actions to full SHAs, use a GitHub `production` environment, and serialize releases with a production concurrency group.
- Treat GitHub branch protection as part of the trust boundary. On a private repository/plan without environment reviewers, a manually triggered release after protected-branch CI is a credible first substitute; it is not equivalent to independent approval.

### Permission shape to validate

| Principal | Narrow purpose | Candidate roles |
| --- | --- | --- |
| GitHub WIF principal | Obtain release identity | `roles/iam.workloadIdentityUser` on the release service account, restricted to the repository identity |
| Release identity | Push and deploy known artifacts | Artifact Registry Writer on the Docker repository; Cloud Run Developer on the two services and migration job; Service Account User on runtime identities; Cloud Run Invoker on the migration job |
| Web runtime | Serve the Next.js application | Only APIs it actually calls; no deployment permissions |
| API runtime | Serve GraphQL | Secret Accessor on API-specific secrets plus data-store permissions selected later |
| Migration runtime | Apply schema changes | Secret/data-store permissions required for migrations, and nothing else |

The predefined roles above are a starting point. Resource-level grants and, if justified by the final deployment mechanism, a small custom role can reduce them further.

## 2. Images and the authoritative deployment writer

### Documented facts

Google recommends Artifact Registry for Cloud Run images. Co-locating the repository and Cloud Run in one region reduces latency and avoids cross-region Artifact Registry transfer charges. Artifact Registry supports cleanup policies with a dry-run mode and keep/delete rules. ([Cloud Run image deployment](https://cloud.google.com/run/docs/deploying), [Artifact Registry locations and cost](https://cloud.google.com/artifact-registry/docs/repositories#repository_location), [cleanup policies](https://cloud.google.com/artifact-registry/docs/repositories/cleanup-policy-overview))

Cloud Run accepts an image tag or digest. When given a tag, Cloud Run resolves it to a digest; the resulting revision continues serving that exact digest. Revisions are immutable, and configuration changes also create new revisions. ([Cloud Run image deployment](https://cloud.google.com/run/docs/deploying))

OpenTofu expects provider version constraints and records selected provider versions and checksums in `.terraform.lock.hcl`, which should be reviewed and committed. The OpenTofu GCS backend supports state locking and recommends Object Versioning for recovery. OpenTofu state can contain secrets and must itself be treated as sensitive. ([OpenTofu provider requirements](https://opentofu.org/docs/language/providers/requirements/), [dependency lock file](https://opentofu.org/docs/language/files/dependency-lock/), [GCS backend](https://opentofu.org/docs/language/settings/backends/gcs/), [sensitive state](https://opentofu.org/docs/language/state/sensitive-data/))

The Google provider, maintained collaboratively by Google and HashiCorp, exposes v2 resources for both a Cloud Run service and a run-to-completion Cloud Run job. The same `hashicorp/google` provider source is installable by OpenTofu. ([Google provider source](https://github.com/hashicorp/terraform-provider-google), [`google_cloud_run_v2_service`](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/cloud_run_v2_service), [`google_cloud_run_v2_job`](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/cloud_run_v2_job))

### Inferences for a first baseline

- Use one regional Docker repository with separate image names, for example `web` and `api`. Tag every build with the full Git commit SHA and deploy the digest emitted by the build, not a mutable convenience tag.
- Add a cleanup policy only after a dry run. Keep the currently deployed images and a small rollback window; expire old unreferenced development images. Cloud Run retains its imported copy while a serving revision uses it, but the registry still needs a human-comprehensible retention rule.
- Provision long-lived resources with OpenTofu: APIs, identities and IAM, Workload Identity Federation, Artifact Registry, Secret Manager secret containers, state storage, monitoring policies, and whichever Cloud Run resources the final ownership model assigns to it.
- Use a remote GCS state bucket with uniform access, minimal IAM, locking, recoverable historical state, and a lifecycle policy. Google-managed encryption at rest is sufficient for the first baseline; CMEK and OpenTofu's additional state encryption can remain later hardening unless policy requires them. Google encrypts customer content at rest by default. ([Google default encryption](https://cloud.google.com/docs/security/encryption/default-encryption))

### Decision that must remain open

There must be **one authoritative writer for revision-bound Cloud Run configuration**, including the image digest, command, runtime identity, environment, secrets, scaling, and traffic policy.

Two coherent paths exist:

1. GitHub Actions supplies image digests to `tofu plan/apply`, and OpenTofu owns the Cloud Run services and job in full.
2. OpenTofu owns the supporting platform, while a release tool such as `gcloud` or `google-github-actions/deploy-cloudrun` owns the complete service/job revision specification.

Letting OpenTofu declare those fields while a second tool mutates them creates drift and can cause a later `tofu apply` to restore an old image or configuration. Choosing the writer requires a separate architecture decision; this research does not select between the two paths.

## 3. Minimal release flow

### Documented facts

The maintained `google-github-actions/deploy-cloudrun` action can deploy a service or job, deploy a new service revision with no traffic, assign revision traffic, and execute a job while waiting for completion. Cloud Run also exposes equivalent `gcloud` and provider resources. ([deploy-cloudrun action](https://github.com/google-github-actions/deploy-cloudrun), [Cloud Run rollout controls](https://cloud.google.com/run/docs/rollouts-rollbacks-traffic-migration))

A Cloud Run Job does not listen for requests; it runs tasks to completion. An execution succeeds only when all tasks succeed, writes logs to Cloud Logging, and sends monitoring data to Cloud Monitoring. Jobs default to one task, a ten-minute task timeout, and three retries; all values are configurable. ([create jobs](https://cloud.google.com/run/docs/create-jobs), [execute jobs](https://cloud.google.com/run/docs/execute/jobs))

### Inferred release sequence

For ordinary releases after bootstrap:

1. Run linting, type checks, unit/integration tests, container builds, and local container smoke tests.
2. Authenticate keylessly, log in to the regional Artifact Registry with a short-lived token, and push `web` and `api` images tagged by commit SHA.
3. Resolve and record both image digests in the GitHub job summary/deployment record.
4. Create the candidate API revision with no production traffic, using the exact API digest.
5. Update the migration job to that same API digest (or a separate migration image if later justified), execute exactly one migration task, and wait for success. A failed or timed-out migration stops the release.
6. Smoke-test the candidate API revision through an authenticated revision URL where the chosen access model permits it.
7. Move API traffic to the candidate revision, then deploy and smoke-test the web revision. The precise order may change if the web/API compatibility contract demands it.
8. Record the serving revision names, digests, migration execution, and workflow URL as release evidence.

For migrations, begin with `tasks = 1` and `parallelism = 1`. Either make every migration idempotent and retry-safe, or set retries to zero; the documented default of three is unsafe for a migration that assumes one execution. This is an application-level decision, not a Cloud Run guarantee.

Every production schema change should be backward-compatible with the previous API revision (expand/contract discipline). Cloud Run can immediately route traffic back to old application code, but it does not undo a database migration. Destructive contract steps must therefore be delayed until no rollback candidate depends on the old schema.

The first-ever deployment is a bootstrap exception because no previous API revision exists. The migration job can run before the first service begins serving traffic.

## 4. Secrets

### Documented facts

Google recommends Secret Manager for Cloud Run secrets. The runtime service identity needs `roles/secretmanager.secretAccessor`. A volume-mounted secret reads the current Secret Manager value and works well with rotation. An environment-variable secret is resolved at instance startup, and Google recommends pinning it to a numeric version rather than `latest`. ([Cloud Run secrets](https://cloud.google.com/run/docs/configuring/services/secrets), [Secret Manager best practices](https://cloud.google.com/secret-manager/docs/best-practices))

OpenTofu's `sensitive` flag suppresses display but does not keep a value out of state. State contains resource attributes and can include initial passwords. ([OpenTofu input variables](https://opentofu.org/docs/language/values/variables/), [sensitive state](https://opentofu.org/docs/language/state/sensitive-data/))

### Inferences for a first baseline

- OpenTofu should create secret resources, IAM, and Cloud Run references. Decide deliberately whether secret **payloads** are managed through OpenTofu; if they are, access to state is equivalent to access to those secrets.
- Prefer secret volume mounts for values that need rotation without a new revision. Prefer pinned numeric versions for environment variables so a revision is reproducible and rollback restores the expected secret version.
- Grant each runtime identity access only to its own secrets. The release identity should be able to reference secret names/versions without reading payloads wherever the deployment API permits that separation.
- Keep non-secret configuration in ordinary environment variables. Never pass secret values as workflow command-line flags or plain GitHub variables.

## 5. Access model for web and API

### Documented facts

Cloud Run services are private by default. Public access is an explicit choice. A private receiving service can grant `roles/run.invoker` to the calling service's identity; the caller supplies a Google-signed ID token with the receiving service as audience. Ingress and IAM are separate controls. ([Cloud Run authentication overview](https://cloud.google.com/run/docs/authenticating/overview), [service-to-service authentication](https://cloud.google.com/run/docs/authenticating/service-to-service), [ingress controls](https://cloud.google.com/run/docs/securing/ingress))

### Decision that must remain open

The web service will need a public path, but the API has two materially different viable shapes:

- A private API invoked server-to-server by Next.js, with Cloud Run IAM between `web-runtime` and `api-runtime`.
- An internet-reachable GraphQL API invoked by the browser, protected by application-level OIDC authorization and its own origin/rate-limit controls.

Cloud Run IAM does not replace end-user authentication by the future OIDC provider. The choice depends on the intended Next.js rendering/data-fetching boundary and GraphQL client behavior, so this delivery research must not decide it implicitly.

The default `run.app` endpoints are enough to prove the baseline. An external Application Load Balancer, Cloud Armor, CDN, and custom domain can wait until their product value is clear.

## 6. Logs, metrics, and the first alerts

### Documented facts

Cloud Run automatically writes request, container, and system logs to Cloud Logging. Application output on `stdout` and `stderr` is collected without a logging agent. Cloud Run is automatically integrated with Cloud Monitoring; built-in metrics include request count, latency, instance count, startup latency, CPU, memory, and billable instance time. Google currently charges nothing for fully managed Cloud Run system metrics. ([Cloud Run logging](https://cloud.google.com/run/docs/logging), [Cloud Run monitoring](https://cloud.google.com/run/docs/monitoring))

### Inferences for a first baseline

- Emit one-line structured JSON to stdout/stderr with severity, service, revision/commit, request or trace correlation ID, event name, and safe diagnostic context. Never log access tokens, secret values, full authorization headers, or sensitive user payloads.
- Start with built-in dashboards. Custom OpenTelemetry metrics, tracing sidecars, and an external observability platform are not prerequisites for the first release.
- Define the smallest actionable alert set as code: public web uptime failure, elevated 5xx ratio, and failed migration-job execution. Add latency and API availability alerts once there is enough traffic to choose meaningful thresholds.
- Send alerts to at least one channel that will actually be observed, initially email. Add a short runbook link to each alert.
- Use log exclusions or sampling only after measuring volume; request logs are automatic and cannot be disabled inside Cloud Run itself.

## 7. Rollback and recovery

### Documented facts

Cloud Run revisions are immutable. Traffic can be assigned to a no-traffic candidate, split gradually, or moved back to a previous revision. Rolling back application traffic does not drop in-flight requests. ([Cloud Run rollouts and rollback](https://cloud.google.com/run/docs/rollouts-rollbacks-traffic-migration))

For a PostgreSQL Cloud SQL example, automated backups and point-in-time recovery can restore persistent data, but when an instance is provisioned with Terraform/OpenTofu or the API those protections must be enabled explicitly rather than assumed from console defaults. This is evidence about one possible datastore, not a datastore selection. ([Cloud SQL PITR configuration](https://cloud.google.com/sql/docs/postgres/backup-recovery/configure-pitr), [Cloud SQL backups](https://cloud.google.com/sql/docs/postgres/backup-recovery/backups))

### Inferences for a first baseline

- Application rollback: keep the previous healthy revision addressable and record a tested command/workflow that sends it 100% of traffic. Do not make “rebuild an old commit” the rollback mechanism.
- Database recovery: whichever persistent store is selected must have automated backup/PITR configured as code, a stated retention period, and a documented restore drill. “Backups enabled” is incomplete until a restore has been tested.
- Migration recovery: prefer a forward fix or database restore based on an explicit incident decision. Never assume shifting Cloud Run traffic reverses schema/data changes.
- Infrastructure recovery: back OpenTofu with locked remote GCS state and recoverable state history. Retain a lifecycle-bounded set of versions so recovery protection does not grow without limit.
- Configuration recovery: release records must connect Git commit, web/API image digests, Cloud Run revisions, migration execution, and OpenTofu revision/plan.

## 8. Scale-to-zero and future realtime collaboration

### Documented facts

With zero minimum instances, Cloud Run removes the final idle instance. A later request starts a new instance and can experience cold-start latency. Scaling from zero can only be triggered by a request; a process cannot remain dormant inside a zero-instance service and wake itself. With request-based billing, CPU after a response is disabled or severely limited, so background work should finish within the request or move to a job/queue-driven design. ([Cloud Run overview](https://cloud.google.com/run/docs/overview/what-is-cloud-run#scale_to_zero_and_minimum_instances), [autoscaling](https://cloud.google.com/run/docs/about-instance-autoscaling), [background activity](https://cloud.google.com/run/docs/tips/general#background_activity))

Cloud Run supports WebSockets, but each connection is a long-running request subject to a configured timeout of at most 60 minutes. Clients must reconnect. Session affinity is best effort, and multiple instances need an external synchronization mechanism. An instance with an open WebSocket is active and billable. ([Cloud Run WebSockets](https://cloud.google.com/run/docs/triggering/websockets), [request timeout](https://cloud.google.com/run/docs/configuring/request-timeout))

### Inferences

- Scale-to-zero is a strong fit for the initial management/list/CRUD application, provided both containers start quickly and the first request may be slower.
- Set an explicit low maximum instance count as a budget and database-connection guard, then tune it from evidence. Google's current cost-safety guidance suggests starting at three, while warning that the limit can briefly be exceeded. ([maximum-instance guidance](https://cloud.google.com/run/docs/configuring/max-instances-limits))
- A live collaborative canvas remains feasible on Cloud Run, but it is not “free while idle” when users hold sockets open. Reconnect logic, cross-instance state synchronization, connection-aware concurrency, and the cost of Redis/Firestore or another broker/store become a separate architecture decision.
- Do not provision Memorystore, a VPC connector, or instance-based billing before the realtime slice needs them. Google's WebSocket reference architecture uses external synchronization and recommends Direct VPC egress for private Redis access; those are later costs and operational commitments, not part of this baseline.

## 9. Current price and limit snapshot

The following values were verified on 2026-08-29 and must be rechecked before implementation.

| Component | Current documented price or free allocation | Baseline consequence |
| --- | --- | --- |
| Cloud Run services | Request-based free tier: 180,000 vCPU-seconds, 360,000 GiB-seconds, and 2 million requests per billing account/month, expressed as a Tier 1 spending discount. With no minimum instances, idle services are not charged. Pricing varies by region; Warsaw (`europe-central2`) is currently Tier 2. ([pricing](https://cloud.google.com/run/pricing)) | Two zero-min services can cost $0 at tiny usage, but do not assume Warsaw receives the same effective unit allowance as a Tier 1 region. |
| Cloud Run jobs | Jobs use instance-based rates with a one-minute minimum per started instance; the pricing page's example of an hourly one-minute, 1 vCPU/512 MiB job in `europe-west1` is $0 after free tier, or $0.45 without it. ([pricing](https://cloud.google.com/run/pricing#jobs)) | Occasional migrations are negligible relative to a database. |
| Artifact Registry | First 0.5 GiB-month free per billing account; above that, `$0.000136986/GiB-hour`, approximately `$0.10/GiB-month`. Same-region transfer to Cloud Run is free. ([pricing](https://cloud.google.com/artifact-registry/pricing)) | Image storage becomes a small recurring cost unless cleanup is enforced. |
| Secret Manager | First 6 active versions and 10,000 access operations per billing account/month are free; then `$0.06` per active version/month and `$0.03` per 10,000 accesses. ([pricing](https://cloud.google.com/secret-manager/pricing)) | A small application remains near zero; destroy obsolete versions deliberately because disabled versions are still billable. |
| Cloud Logging | First 50 GiB/project/month of standard log storage is free; then `$0.50/GiB`, including 30-day storage. ([pricing](https://cloud.google.com/logging#pricing)) | Structured, non-verbose logs should fit easily; uncontrolled debug or payload logging is the cost risk. |
| Cloud Monitoring | Fully managed Cloud Run system metrics are currently free. Public uptime checks have a 1 million execution/month free allotment across the billing account. ([Cloud Run monitoring](https://cloud.google.com/run/docs/monitoring), [uptime pricing examples](https://cloud.google.com/stackdriver/observability-pricing-examples#uptime-checks)) | Built-in metrics and a few uptime checks should add no fixed cost. |
| GitHub Actions | Public repositories on standard hosted runners are free. GitHub Free private repositories currently include 2,000 minutes and 500 MB artifact storage/month; other plans differ. ([billing](https://docs.github.com/en/billing/concepts/product-billing/github-actions)) | Builds are free within the user's existing allowance; keep artifacts short-lived and use registry digests as release artifacts. |
| OpenTofu state | GCS is usage-priced and the state is tiny, but historical object versions also consume storage. ([GCS backend](https://opentofu.org/docs/language/settings/backends/gcs/), [Cloud Storage Object Versioning](https://cloud.google.com/storage/docs/object-versioning)) | Usually pennies or less; lifecycle old versions rather than allowing unbounded history. |

### Likely fixed-cost drivers not resolved here

- A managed relational database such as Cloud SQL does not scale to zero and adds continuous compute, storage, backup, and possibly HA charges.
- Realtime synchronization such as Memorystore adds continuously provisioned infrastructure; Firestore has a different usage-based model.
- Minimum Cloud Run instances, Serverless VPC Access connectors, an external load balancer/Cloud Armor, custom DNS/domain registration, and some external OIDC provider plans can add fixed or recurring charges.

**Cost inference:** excluding the database, external identity provider, domain, and future realtime infrastructure, a careful learning-scale baseline should remain between $0 and a few USD per month. The datastore decision needs its own cost comparison before the target budget can be trusted.

## 10. What can remain outside the first baseline

- Kubernetes/GKE, multi-region service deployment, active-active data, and automated cross-region disaster recovery.
- Cloud Deploy and Cloud Build; GitHub Actions can build and release the first two images.
- External Application Load Balancer, Cloud Armor, CDN, custom domain, and per-PR cloud environments.
- Memorystore/Redis, WebSockets, realtime presence, and cross-instance collaboration state.
- Custom OpenTelemetry pipelines, third-party observability, SLO automation, and extensive custom metrics.
- Binary Authorization, CMEK, VPC Service Controls, a full organization landing zone, and dedicated security projects, unless an external policy makes them mandatory.
- Automatic canary analysis; a no-traffic smoke test plus explicit traffic switch and tested rollback is enough to learn the first operational loop.
- Fully automated destructive schema changes. The first baseline should support only backward-compatible migrations.

## Decisions this research leaves for later

1. Whether OpenTofu or a dedicated release command owns Cloud Run revision configuration.
2. Direct Workload Identity Federation versus keyless service-account impersonation after a small compatibility test.
3. Whether GraphQL is browser-to-API or server-to-server through Next.js, which determines the API's Cloud Run access model.
4. Region selection: Warsaw proximity versus Tier 1 price and service availability in another EU region.
5. Persistent database and its connection path, backup/PITR policy, restore objective, and fixed cost.
6. OIDC provider and whether app-layer authentication is shared by web and API.
7. The later realtime synchronization design and whether Cloud Run remains the right host for that slice.

## Primary-source index

- [Google: Workload Identity Federation with deployment pipelines](https://cloud.google.com/iam/docs/workload-identity-federation-with-deployment-pipelines)
- [GitHub: OIDC in Google Cloud](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-google-cloud-platform)
- [Google GitHub auth action](https://github.com/google-github-actions/auth)
- [Google GitHub Cloud Run deploy action](https://github.com/google-github-actions/deploy-cloudrun)
- [Cloud Run: deploy images](https://cloud.google.com/run/docs/deploying)
- [Cloud Run: IAM roles and deployment permissions](https://cloud.google.com/run/docs/reference/iam/roles)
- [Cloud Run: jobs](https://cloud.google.com/run/docs/create-jobs)
- [Cloud Run: secrets](https://cloud.google.com/run/docs/configuring/services/secrets)
- [Cloud Run: logs and metrics](https://cloud.google.com/run/docs/logging)
- [Cloud Run: rollout and rollback](https://cloud.google.com/run/docs/rollouts-rollbacks-traffic-migration)
- [Cloud Run: WebSockets](https://cloud.google.com/run/docs/triggering/websockets)
- [Cloud Run pricing](https://cloud.google.com/run/pricing)
- [Artifact Registry pricing](https://cloud.google.com/artifact-registry/pricing)
- [Secret Manager pricing](https://cloud.google.com/secret-manager/pricing)
- [OpenTofu GCS backend](https://opentofu.org/docs/language/settings/backends/gcs/)
- [OpenTofu sensitive state](https://opentofu.org/docs/language/state/sensitive-data/)
- [Google provider source and documentation](https://github.com/hashicorp/terraform-provider-google)
