# Session journal — 2026-11-03: deploy eShop to AWS ECS Fargate

Narrative record of a working session. What we set out to do, what broke, how
we diagnosed it, and the decisions made. The distilled, reusable version lives
in `docs/runbooks/eshop-aws-fargate.md`.

## Goal

Deploy the full eShop application (the .NET Aspire reference app in the `eShop`
child repo) to AWS ECS Fargate — all services, "spare no expense."

## Outcome

Full stack running in AWS account `211095244575` (AWS-InnovationLabs-DEN),
`us-east-1`, under a dedicated VPC. 13 ECS services healthy; storefront served
behind an internet-facing ALB with a wildcard TLS cert.

- Storefront: https://eshop.denver-innovation-lab.slalomdev.io
- Identity:   https://eshop-id.denver-innovation-lab.slalomdev.io
- Mobile BFF: https://eshop-bff.denver-innovation-lab.slalomdev.io

Public ALB later locked down to a single workstation IP (`72.10.71.58/32`).

## Context that shaped the approach

- eShop is a .NET Aspire app. The Aspire AppHost (`src/eShop.AppHost/AppHost.cs`)
  is the single source of truth for the topology and all injected env vars.
- It is modeled for **Azure Container Apps** (`AddAzureContainerAppEnvironment`).
  Reproducing it on AWS Fargate means hand-translating the AppHost's references
  and env wiring into task definitions + Cloud Map service discovery.
- An earlier `infra/aws-fargate-stack` already existed and was ~80% complete
  (Cloud Map model, a reusable service module, ALB, EFS, the OIDC cyclic-URL
  solved via pre-assigned DNS hostnames). We finished and corrected it rather
  than rebuild.

## Problems hit, in order, and the fixes

1. **x86 vs ARM64 image mismatch** — `exec format error`.
   Images built on an arm64 Mac won't run on x86_64 Fargate. Fix: set
   `runtime_platform { cpu_architecture = "ARM64" }` on every task def and build
   images `linux-arm64`.

2. **Health endpoints 404 in Production.** eShop's `MapDefaultEndpoints` only
   maps `/health` and `/alive` when `IsDevelopment()`. Fix: set
   `ASPNETCORE_ENVIRONMENT=Development` on all services (also enables data seeding).

3. **Only one database created.** `POSTGRES_DB` makes a single DB, but eShop needs
   four (catalogdb, identitydb, orderingdb, webhooksdb). EF migrations create
   *tables*, not databases. Fix: postgres container entrypoint drops an idempotent
   init SQL + a background psql loop to create all four.

4. **Postgres data dir not portable across CPU arch.** Postgres first initialized
   on x86_64; after switching the task to ARM64 it PANIC'd —
   `could not locate a valid checkpoint record`. Fix: point `PGDATA` at a fresh
   subdirectory so ARM64 initializes a clean cluster (reference app, no real data).

5. **Task-to-task connectivity silently broken (the big one).** The task security
   group mixed **inline** ingress/egress blocks with a **standalone**
   `aws_security_group_rule` for the self-referential rule. The AWS provider
   treats that as unsupported and deletes the out-of-band rule on every apply —
   so every `terraform apply` broke postgres/redis/rabbitmq connectivity, and
   services timed out connecting to the DB (surfaced as a Polly 30s
   `TimeoutRejectedException` on the catalog page). Fix: move the self-rule
   **inline** (`self = true`) and remove the standalone rule from config + state.

6. **identity-api killed mid-migration.** It runs EF migration + user seeding
   synchronously before the web server listens; the ALB health check killed it
   before it finished. Fix: `health_check_grace_period_seconds = 600`. Also keep
   it single-task (developer signing credential, no key persistence).

7. **mobile-bff is not a repo project.** It's an Aspire YARP resource whose config
   is generated at build time. The prebuilt YARP image is chiseled (no shell, so
   the entrypoint-writes-config trick fails) and hardcodes `ASPNETCORE_URLS=
   http://+:5000`. Fixes: bake the route config into a tiny custom image
   (`mobile-bff/Dockerfile`, `COPY yarp.config /etc/yarp.config`) and override
   `ASPNETCORE_URLS=http://+:8080` to match the port mapping + health check.

## Key decisions

- **Project prefix `eshopf`** (not `eshop`) to avoid colliding with the earlier
  single-service `aws-oidc` stack that already owns the `eshop` cluster, the
  `eshop/webapp` ECR repo, and `eshop-task*` IAM roles. Public hostnames are
  driven by the `subdomain_*`/`domain` vars, so the storefront URL stays clean.
- **Dedicated VPC** (not the account default VPC, whose internet gateway had been
  deleted — a separate earlier rabbit hole).
- **Two env-var naming styles** matter: APIs read `Identity__Url`; Blazor apps
  read `IdentityUrl`. Apps also need `CallBackUrl` = their own public URL.

## Follow-ups / known gaps

- Identity uses an ephemeral developer signing credential — single-task only; not
  safe to scale horizontally without persistent signing keys.
- The stack bills continuously (~13 Fargate tasks + ALB + NAT + EFS). Scale app
  services to 0 or `terraform destroy` when idle.
