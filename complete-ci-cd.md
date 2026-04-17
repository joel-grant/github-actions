# CI/CD Pipeline Architecture — Rails on Docker (DigitalOcean)
### DevSecOps-Integrated | Production-Grade Maturity

---

## Step 1: Developer Commits Code
- Developer commits to a feature branch following **Conventional Commits** format
- **Pre-commit hooks** (Overcommit/Lefthook) run RuboCop, Brakeman, and GitLeaks locally
- Signed commits (GPG/SSH) are required for author identity verification

## Step 2: Branch Strategy & Pull Request
- PR is opened targeting `main` or `develop` with an auto-populated template
- **CODEOWNERS** assigns reviewers; branch protection requires 2 approvals, passing CI, and signed commits
- **GitHub Rulesets** enforce branch and tag naming conventions

## Step 3: CI Pipeline is Triggered
- GitHub Actions triggers on `pull_request` and `push` events
- **OIDC** authenticates to DigitalOcean — no long-lived secrets
- Workflow permissions follow least-privilege; run metadata is logged for audit

## Step 4: Dependency Installation & Supply Chain Security
- `bundle install` and `yarn install --frozen-lockfile` with CI caching
- `bundler-audit`, `yarn audit`, and **Dependabot** scan for vulnerable dependencies
- An **SBOM** is generated and attached as a build artifact

## Step 5: Static Analysis & Security Linting
- **RuboCop** enforces code quality; **Brakeman** scans for Rails security vulnerabilities
- **Semgrep** runs custom and OWASP security rules
- Critical or high-severity findings block the PR

## Step 6: Build Docker Image
- **Multi-stage Dockerfile**: builder stage compiles assets, runtime stage uses a minimal base image
- Image runs as non-root, excludes build tools, and is tagged with the commit SHA
- Secrets use BuildKit secret mounts — never baked into the image

## Step 7: Unit Testing
- RSpec/Minitest runs against a PostgreSQL service container
- **SimpleCov** enforces a minimum coverage threshold; tests are parallelized for speed
- Flaky tests are auto-retried and quarantined

## Step 8: Integration Testing
- Request specs and Capybara validate API endpoints and page interactions
- Sidekiq, ActionMailer, and ActiveStorage are tested in-process
- **Contract testing** (Pact) ensures API compatibility between services

## Step 9: End-to-End (E2E) Testing
- Full stack spun up via `docker compose`; Capybara + Playwright simulates user workflows
- Critical paths covered: auth, CRUD, payments, admin access
- Accessibility validated with `axe-core`; screenshots captured on failure

## Step 10: Security Scanning (DevSecOps Gate)
- **SAST**: Brakeman + Semgrep; **SCA**: bundler-audit + yarn audit + license compliance
- **Container scanning**: Trivy/Grype for CVEs; **Secret detection**: GitLeaks + GitHub Push Protection
- **DAST**: OWASP ZAP baseline scan against the test environment
- Any critical finding blocks promotion to staging

## Step 11: Push Image to DigitalOcean Container Registry (DOCR)
- Image is pushed to DOCR, tagged with commit SHA and semantic version on merge
- **Cosign** signs the image; **SLSA provenance attestation** is attached
- Registry access is scoped: CI pushes, production only pulls

## Step 12: Deploy to Staging on DigitalOcean
- Deployed via **App Platform**, **DOKS**, or **Droplets** depending on infrastructure choice
- Database migrations run as a pre-deploy job
- Secrets injected via environment-scoped config; staging is network-isolated (VPN/IP allowlist)

## Step 13: Staging Validation & Penetration Testing
- Smoke tests and Rails health check (`/up`) verify core functionality
- **k6** runs performance benchmarks; `bullet` gem flags N+1 queries
- OWASP ZAP active scan and optional chaos testing validate security and resilience

## Step 14: Database Migration Safety Check
- **strong_migrations** flags dangerous operations (column removal, non-concurrent indexes)
- Migrations must be backward-compatible for zero-downtime deploys
- Database backup confirmed via DigitalOcean Managed DB snapshots before destructive changes

## Step 15: Approval Gate & Change Management
- GitHub Actions `environment: production` requires manual approval from designated reviewers
- Approvers verify staging results, security scans, and rollback plan
- Approval metadata is logged immutably for **SOC 2 / audit compliance**

## Step 16: Deploy to Production on DigitalOcean
- The **same immutable, signed image** from staging is deployed — no rebuild
- Zero-downtime strategy: rolling updates (App Platform/DOKS) or load-balanced rotation (Droplets)
- **Feature flags** (Flipper) enable dark launches; **canary deploys** route incremental traffic

## Step 17: Post-Deployment Verification
- Health checks and smoke tests validate critical flows; error rates compared to pre-deploy baseline
- Automatic rollback triggers if error rate spikes >2x
- Results posted to Slack and GitHub Deployment status

## Step 18: Monitoring & Observability
- **DigitalOcean Monitoring** tracks infrastructure metrics (CPU, memory, DB, Redis, Load Balancer)
- **APM** (AppSignal/New Relic/Datadog) provides Rails-specific performance insights and N+1 detection
- **Structured logging** via `lograge` with centralized aggregation; **OpenTelemetry** for distributed tracing

## Step 19: Alerting & Incident Response
- Alerts configured for error rate spikes, latency breaches, queue depth, and security anomalies
- Routed to on-call via **PagerDuty/Opsgenie** with escalation policies and Slack integration
- **Runbooks** document remediation; **SEV1-SEV4 classification** sets response SLAs

## Step 20: Rollback Procedures
- **App rollback**: Redeploy previous image tag through the same pipeline
- **DB rollback**: `rails db:rollback` or DigitalOcean point-in-time recovery
- **Feature flag rollback**: Flipper toggles disable features instantly — no deploy needed

## Step 21: Scaling on DigitalOcean
- Horizontal scaling via App Platform auto-scaling, DOKS HPA, or Load Balancer + Terraform
- Sidekiq, database (read replicas + PgBouncer), and Redis scaled independently
- **rack-attack** provides application-layer rate limiting and DDoS protection

## Step 22: Compliance, Governance & Audit
- Immutable audit trail for all deployments, approvals, and rollbacks
- Encryption at rest (AES-256) and in transit (TLS 1.2+); `ActiveRecord::Encryption` for PII
- Least-privilege access, secret rotation, Dependabot SLA triage, and quarterly pen testing

## Step 23: Disaster Recovery & Business Continuity
- **RPO < 1 hour / RTO < 30 minutes** backed by automated DB backups and multi-region DNS failover
- DR drills run quarterly (DB restore), semi-annually (full stack), and annually (tabletop)
- Entire stack reproducible from **Terraform** — no manual configuration

## Step 24: Feedback & Continuous Improvement
- **DORA metrics** tracked: deploy frequency, lead time, change failure rate, MTTR
- Blameless post-incident retrospectives with action items tracked in GitHub Issues
- Pipeline optimized continuously: parallel tests, Docker caching, flaky test quarantine, cost right-sizing

---

> **Maturity Model Summary**
>
> | Capability | Maturity Level |
> |---|---|
> | Source Control & Branching | ✅ Advanced — signed commits, rulesets, CODEOWNERS |
> | CI/CD Automation | ✅ Advanced — fully automated, parallelized, cached |
> | Security (SAST/SCA/DAST) | ✅ Advanced — multi-layer scanning, policy gates |
> | Container Security | ✅ Advanced — image signing, provenance, distroless |
> | Supply Chain Security | ✅ Advanced — SBOM, Dependabot, SLSA attestation |
> | Testing | ✅ Advanced — unit, integration, E2E, mutation, accessibility |
> | Deployment Strategy | ✅ Advanced — zero-downtime, canary, feature flags |
> | Monitoring & Observability | ✅ Advanced — APM, structured logs, tracing, synthetic |
> | Incident Response | ✅ Advanced — runbooks, escalation, status page |
> | Compliance & Governance | ✅ Advanced — audit trail, encryption, policy-as-code |
> | Disaster Recovery | ✅ Advanced — tested DR, multi-region, IaC |
> | Continuous Improvement | ✅ Advanced — DORA metrics, retrospectives, cost optimization |