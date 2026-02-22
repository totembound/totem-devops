# TotemBound DevOps

GitHub Actions workflows for building and deploying the TotemBound game.

## Workflows

### App (Frontend → S3 + CloudFront)

| Workflow | File | What it does |
|----------|------|--------------|
| **CI app build** | `ci-app-build.yml` | Checkout → lint → test → Vite build → zip → S3 |
| **CI app deploy** | `ci-app-deploy.yml` | Download zip from S3 → sync to app bucket → invalidate CloudFront |
| **PROD app build** | `prod-app-build.yml` | Same as CI, prod env vars + PostHog key |
| **PROD app deploy** | `prod-app-deploy.yml` | Same as CI, prod S3 bucket + CloudFront distribution |

### API (Lambda)

| Workflow | File | What it does |
|----------|------|--------------|
| **CI API build** | `ci-api-build.yml` | Checkout → npm ci → lint → test → build → package zip → S3 |
| **CI API deploy** | `ci-api-deploy.yml` | `lambda update-function-code` from S3 zip → wait → verify |
| **PROD API build** | `prod-api-build.yml` | Same as CI, prod credentials |
| **PROD API deploy** | `prod-api-deploy.yml` | Same as CI, prod Lambda function |

### Infrastructure (CloudFormation)

| Workflow | File | What it does |
|----------|------|--------------|
| **CI infra deploy** | `ci-infra-deploy.yml` | Upload CF template to S3 → create/update stack → print outputs |
| **PROD infra deploy** | `prod-infra-deploy.yml` | Same as CI, prod credentials + region |

**Stacks** (deploy order matters — cross-stack references):

1. **core** — 9 DynamoDB tables + IAM execution role
2. **cognito** — User Pool + App Client (token validity, auth flows)
3. **api** — Lambda function + REST API Gateway + Cognito Authorizer
4. **iot** — Cognito Identity Pool + IoT policies for real-time push via MQTT/WebSocket

When deploying `all`, stacks run sequentially: core → cognito → api → iot.

## Environments

| | Staging | Production |
|---|---------|------------|
| **Region** | us-west-2 | us-east-1 |
| **AWS credentials** | `AWS_ACCESS_KEY_ID2` | `AWS_ACCESS_KEY_ID` |
| **S3 releases bucket** | `totemboundci-releases` | `totembound-releases` |
| **S3 app bucket** | `totemboundci-app` | `totembound-app` |
| **CloudFront (app)** | `E30D3Q6UOWWMO4` | `E3TBCP5P18U4EE` |
| **Lambda function** | `totemboundci-api` | `totembound-api` |
| **Stack prefix** | `totemboundci-*-stack` | `totembound-*-stack` |
| **Frontend URL** | totembound-test.net | totembound.com |

## S3 Release Bucket Layout

```
totemboundci-releases/
  totem-app/
    release-{version}.zip            # Frontend build
  totem-api/
    totembound-api-{version}.zip     # Lambda code package
    totembound-api-latest.zip
    cloudformation/
      core-{version}.yml             # CF templates (versioned + latest)
      core-latest.yml
      cognito-{version}.yml
      cognito-latest.yml
      api-{version}.yml
      api-latest.yml
      iot-{version}.yml
      iot-latest.yml
```

## Deploy Pattern

All deployments follow the same flow: **build → version → S3 → deploy from S3**.

```
App:    source → lint/test → vite build → zip → S3 → unzip → sync to app bucket → CloudFront invalidation
API:    source → lint/test → build → package zip → S3 → lambda update-function-code from S3
Infra:  source → upload template to S3 → cloudformation create/update-stack from local file → wait
```

## Triggers

All workflows use `workflow_dispatch` (manual trigger from GitHub Actions UI).

**Build workflows** accept:
- `build_version` — version string (e.g. `0.2.0`)
- `source_branch` — git branch to build from (default: `main`)

**Deploy workflows** accept:
- `release_version` — must match a version already in S3

**Infra deploy** accepts:
- `stack` — which stack to deploy (`core`, `cognito`, `api`, or `all`)
- `release_version` — version tag for the template in S3
- `source_branch` — branch containing CF templates (default: `main`)

## Local Deploy (CLI)

Infrastructure can also be deployed from a local machine using Node.js scripts in `totem-api/scripts/`:

```bash
cd totem-api
node scripts/deploy-core.js      # Deploy core stack
node scripts/deploy-cognito.js   # Deploy cognito stack
node scripts/deploy-api.js       # Deploy api stack (also packages + uploads Lambda zip)
```

Or directly with AWS CLI:

```bash
aws cloudformation update-stack \
  --stack-name totemboundci-cognito-stack \
  --template-body file://infrastructure/cloudformation/cognito.yml \
  --parameters ParameterKey=Environment,ParameterValue=staging \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-west-2
```

## Parameter Files

CloudFormation parameters live in `totem-api/infrastructure/`:

| File | Used by |
|------|---------|
| `params-core-staging.json` | core stack (staging) |
| `params-cognito-staging.json` | cognito stack (staging) |
| `params-api-staging.json` | api stack (staging) — Stripe price IDs, SSM paths |

Prod parameter files (`params-*-prod.json`) will be added when production infrastructure is deployed.

## Secrets (GitHub)

| Secret | Purpose |
|--------|---------|
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` | Prod AWS credentials |
| `AWS_ACCESS_KEY_ID2` / `AWS_SECRET_ACCESS_KEY2` | Staging AWS credentials |
| `PAT` | GitHub Personal Access Token for cross-repo checkout |
