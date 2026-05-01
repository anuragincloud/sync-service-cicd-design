# sync-service-cicd-design
| Branch      | Environment | Purpose                   |
| ----------- | ----------- | ------------------------- |
| `feature/*` | —           | Feature development       |
| `develop`   | `qa`        | Integration testing       |
| `staging`   | `staging`   | Pre-production validation |
| `main`      | `prod`      | Production release        |

Flow : feature/* → develop → staging → main

Preventing Accidental Prod Deployments:
Only main branch can deploy to production

## Jenkins Pipeline Design (Stages):
1. Checkout
2. Build (Maven/Gradle)
3. Unit Tests
4. Static Code Analysis (SonarQube optional)
5. Build Docker Image
6. Push to Artifact Registry (GCP)
7. Deploy (based on branch)
8. Smoke Tests
