# sync-service-cicd-design
| Branch      | Environment | Purpose                   |
| ----------- | ----------- | ------------------------- |
| `feature/*` | —           | Feature development       |
| `develop`   | `qa`        | Integration testing       |
| `staging`   | `staging`   | Pre-production validation |
| `main`      | `prod`      | Production release        |

Flow : feature/* → develop → staging → main

## Preventing Accidental Prod Deployments:
-- Only the main branch is allowed to deploy to production
-- Branch protection rules are enabled to prevent direct commits
-- All changes must go through pull request (PR) reviews before merging
-- CI/CD pipeline does not deploy on PRs (only build and test run)
-- A manual approval step is required before production deployment
-- Environment mapping is strictly controlled (develop → qa, staging → staging, main → prod)
-- These controls help prevent accidental or unsafe production releases

## Jenkins Pipeline Design (Stages):
1. Checkout
2. Build (Maven/Gradle)
3. Unit Tests
4. Static Code Analysis (SonarQube)
5. Build Docker Image
6. Push to Artifact Registry (GCP)
7. Deploy (based on branch)
8. Smoke Tests

## Behavior: PR vs Merge
#### Pull Request (PR):

Triggers automated pipeline for validation
Runs build and unit tests
Performs code quality checks (optional: linting, SonarQube)
No deployment to any environment
Ensures code is safe before merging

#### Merge to Branch:

Triggers full CI/CD pipeline
Builds and pushes Docker image
Deploys based on branch:
develop → QA
staging → Staging
main → Production (with manual approval)
Runs post-deployment smoke tests

## Rollback Strategy
Keep previous Docker images tagged (v1, v2, etc.)
If deployment fails: Auto rollback to last stable version
On GCP VM: Use systemd / docker-compose version rollback
Maintain: Health checks & Deployment logs

## Configuration Management
Environment-specific configs are managed using Spring Boot profiles
**-- application-qa.yml
-- application-staging.yml
-- application-prod.yml**
Active profile is selected at runtime using:
     **SPRING_PROFILES_ACTIVE environment variable**
Ensures clear separation of configurations across environments
Sensitive data is not stored in the codebase
Secrets (MongoDB credentials, API keys) are managed using:
**-- GCP Secret Manager
-- Jenkins Credentials Store**
Secrets are injected securely at runtime as environment variables
At runtime, Inject via environment variables: MONGO_URI=____
Improves security, flexibility, and maintainability across environments

## Deployment Strategy
Blue/Green deployment strategy is used to ensure zero downtime
Two identical environments are maintained:
**Blue → current live version
Green → new version**
New releases are deployed to the inactive environment (Green)

Traffic is switched only after successful validation
Previous version (Blue) is kept as a backup for quick rollback

In case of failure:
Instantly switch traffic back to the previous stable version

Ensures:
Minimal to zero downtime
Safer deployments
Easy and fast rollback

**Suitable for VM-based deployments using:
Load balancer or NGINX for traffic switching**

## Zero Downtime Flow
1. Deploy to green
2. Run health check
3. Switch traffic
4. Keep blue as backup
