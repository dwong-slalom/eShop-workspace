# Runbook — eShop on AWS ECS Fargate

Operational reference for the full-stack eShop deployment. The Terraform lives
in the `eShop` child repo at `infra/aws-fargate-stack/`. Narrative background:
`docs/sessions/2026-11-03-aws-fargate-deployment.md`.

## What it is

The complete eShop app (13 services) on ECS Fargate in a dedicated VPC, fronted
by an internet-facing ALB with a wildcard ACM cert. Replicates the Aspire
AppHost topology by hand using Cloud Map (`<service>.eshopf.local`) for internal
service discovery.

| Thing | Value |
| --- | --- |
| AWS account | `211095244575` (AWS-InnovationLabs-DEN) |
| Region | `us-east-1` |
| Project prefix | `eshopf` (cluster, ECR repos, roles) |
| Storefront | https://eshop.denver-innovation-lab.slalomdev.io |
| Identity | https://eshop-id.denver-innovation-lab.slalomdev.io |
| Mobile BFF | https://eshop-bff.denver-innovation-lab.slalomdev.io |

Services: `postgres`, `redis`, `eventbus` (data stores) · `identity-api`,
`catalog-api`, `basket-api`, `ordering-api`, `webhooks-api` · `order-processor`,
`payment-processor` (workers) · `webapp`, `webhooksclient` (front ends) ·
`mobile-bff` (YARP).

## Deploy from scratch

Run from `eShop/infra/aws-fargate-stack`. Requires AWS creds for the account,
Terraform, and Podman (or Docker) for ARM64 image builds.

```bash
# 1. Infra first, app services at 0 (placeholder images would crash-loop)
terraform init
terraform apply   # tfvars default app_desired_count=0, store_desired_count=1

# 2. Build + push all 9 .NET service images as linux-arm64
bash build-and-push.sh                       # tags images with the git short SHA

# 3. Build + push the custom mobile-bff YARP image (ARM64)
#    (ECR repo eshopf/mobile-bff; see mobile-bff/Dockerfile)

# 4. Point services at real images and scale up
#    set image_tag + app_desired_count=1 in terraform.tfvars, then
terraform apply
```

## Verify health

```bash
# Services (expect 13x running=desired, rollout COMPLETED)
for s in postgres redis eventbus identity-api catalog-api basket-api \
  ordering-api order-processor payment-processor webhooks-api \
  webhooksclient webapp mobile-bff; do
  aws ecs describe-services --cluster eshopf --services "$s" --region us-east-1 \
    --query 'services[0].[runningCount,desiredCount,deployments[0].rolloutState]' \
    --output text
done

# Public ALB targets (expect all healthy)
for tg in eshopf-webapp-tg eshopf-identity-tg eshopf-bff-tg; do
  arn=$(aws elbv2 describe-target-groups --names "$tg" --region us-east-1 \
    --query 'TargetGroups[0].TargetGroupArn' --output text)
  aws elbv2 describe-target-health --target-group-arn "$arn" --region us-east-1 \
    --query 'TargetHealthDescriptions[].TargetHealth.State' --output text
done
```

## Gotchas (each cost real time — read before changing anything)

- **ARM64 everywhere.** Images are `linux-arm64`; task defs set
  `runtime_platform.cpu_architecture = ARM64`. Mismatch → `exec format error`.
- **`ASPNETCORE_ENVIRONMENT=Development`** on every service, or `/health` and
  `/alive` 404 (they are only mapped in Development) and seeding is skipped.
- **Four databases** are created by the postgres container entrypoint; EF
  migrations only create tables, not the databases themselves.
- **Postgres data is not portable across CPU architecture.** If you ever switch
  arch, the old data dir PANICs on recovery — use a fresh `PGDATA`.
- **Never mix inline SG rules with a standalone `aws_security_group_rule` on the
  same SG.** It deletes the out-of-band rule every apply. The task-to-task rule
  is inline (`self = true`) on the task SG — keep it that way.
- **Env naming:** APIs use `Identity__Url`; Blazor apps use `IdentityUrl`; apps
  need `CallBackUrl` = their own public URL.
- **identity-api:** single task (ephemeral dev signing key); needs a health-check
  grace period for its synchronous startup migration + seed.
- **mobile-bff:** custom image with `yarp.config` baked in (chiseled base, no
  shell); must set `ASPNETCORE_URLS=http://+:8080` (image defaults to 5000).

## Access control

Public ALB ingress is controlled by `allowed_ingress_cidrs` in
`terraform.tfvars`. Currently locked to one workstation IP. To open or change:

```hcl
allowed_ingress_cidrs = ["0.0.0.0/0"]            # public
# allowed_ingress_cidrs = ["203.0.113.4/32"]     # single host
```
then `terraform apply`.

## Cost / teardown

Runs ~13 Fargate tasks + ALB + NAT gateway + EFS continuously. To stop charges:

```bash
# Pause (keep infra, stop compute)
# set app_desired_count=0 and store_desired_count=0 in terraform.tfvars
terraform apply

# Or tear it all down
terraform destroy
```
