# CI/CD Pipeline Architecture — Rails on Docker (DigitalOcean)
### DevSecOps-Integrated | Production-Grade Maturity

---

## Step 1: Developer Commits Code
- Developer writes Rails code locally and commits changes to a feature branch
- Commit messages follow **Conventional Commits** format (e.g., `feat:`, `fix:`, `chore:`) enforced by `commitlint`
- **Pre-commit hooks** (via Overcommit or Lefthook) run automatically:
  - RuboCop for style enforcement
  - Brakeman for static security scanning
  - GitLeaks for secret detection in staged files
  - ERB Lint for template validation
- Signed commits are required using GPG or SSH keys to guarantee author identity and code integrity
- Code is pushed to the remote repository on GitHub

## Step 2: Branch Strategy & Pull Request
- The team follows **GitHub Flow** or **Gitflow** depending on release cadence
- Developer opens a pull request (PR) targeting the `main` or `develop` branch
- PR template is auto-populated with:
  - Description and motivation
  - Migration notes and rollback instructions
  - Security impact checklist
  - Linked issues and acceptance criteria
- **CODEOWNERS** file enforces automatic reviewer assignment by directory and domain
- Branch protection rules require:
  - Minimum 2 approvals (including 1 from a security champion for sensitive paths)
  - All CI status checks passing
  - No dismissed reviews
  - Up-to-date branch with base
  - Signed commits
- **GitHub Rulesets** enforce naming conventions on branches and tags

## Step 3: CI Pipeline is Triggered
- A GitHub Actions workflow is triggered on `pull_request` and `push` events
- The workflow checks out the source code at the exact commit SHA
- **OpenID Connect (OIDC)** is used for keyless authentication to DigitalOcean — no long-lived API tokens stored in secrets
- Rails secrets (`RAILS_MASTER_KEY`, database credentials) are injected from GitHub Actions Secrets with **environment-scoped access controls**
- A clean, ephemeral runner is provisioned; self-hosted runners are hardened and rotated regularly
- **Workflow permissions** follow least-privilege: `permissions: contents: read` by default
- Pipeline run metadata (trigger, actor, SHA, timestamp) is logged for audit trail

## Step 4: Dependency Installation & Supply Chain Security
- `bundle install` resolves and installs all Ruby gems from `Gemfile.lock`
- `yarn install --frozen-lockfile` installs JavaScript dependencies deterministically
- GitHub Actions caching is configured for `vendor/bundle` and `node_modules`
- **Supply chain security checks**:
  - `bundler-audit` scans gems against the Ruby Advisory Database
  - `yarn audit` checks JavaScript packages for known vulnerabilities
  - **Dependabot** is enabled for automated dependency update PRs with auto-merge for patch versions
  - **GitHub Dependency Graph** and **Dependabot Alerts** provide continuous visibility into vulnerable transitive dependencies
- A **Software Bill of Materials (SBOM)** is generated using `cyclonedx-ruby` and attached as a build artifact
- **Gem source verification**: `Gemfile` locks to `https://rubygems.org` only; private gems use a scoped authenticated source

## Step 5: Static Analysis & Security Linting
- **RuboCop** with `rubocop-rails`, `rubocop-rspec`, and `rubocop-performance` extensions enforces code quality
- **Brakeman** performs deep static security analysis:
  - SQL injection, XSS, CSRF, mass assignment, command injection, file access
  - Confidence-level thresholds are configured — high-confidence warnings fail the build
- **ESLint / Prettier** lint and format JavaScript and frontend assets
- **Rails Best Practices** gem flags anti-patterns and performance issues
- **Reek** detects code smells in Ruby (feature envy, long methods, data clumps)
- **Flay** identifies structural duplication across the codebase
- **Semgrep** runs custom and community security rules tailored to Rails patterns
- All findings are posted as inline PR annotations for developer-friendly feedback
- Any critical or high-severity finding blocks the PR from merging

## Step 6: Build Docker Image
- A **multi-stage `Dockerfile`** minimizes the final image size and attack surface:
  - **Stage 1 (Builder)**: Full build toolchain installs gems, compiles assets with `rails assets:precompile`, and builds native extensions
  - **Stage 2 (Runtime)**: Copies only compiled app into a minimal base image (`ruby:3.3-slim` or distroless)
- **Security hardening in the Dockerfile**:
  - Runs as a non-root user (`USER rails`)
  - No shell or package manager in the final image (if using distroless)
  - `apt-get` layers are cleaned up; no build tools in production image
  - `HEALTHCHECK` instruction defines a container-level health probe
- The image is tagged with commit SHA and branch: `registry.digitalocean.com/<registry>/myapp:<commit-sha>`
- `.dockerignore` excludes `.git`, `test/`, `spec/`, `log/`, `tmp/`, `.env*`, and credentials files
- Build arguments inject `RAILS_ENV=production`; secrets are passed via BuildKit secret mounts — **never as `ARG` or `ENV` in the image**
- **Docker layer caching** in CI speeds up rebuilds
- **Reproducible builds**: Pinned base image digests ensure builds are deterministic

## Step 7: Unit Testing (RSpec / Minitest)
- A PostgreSQL service container is spun up alongside the test runner
- `rails db:create db:schema:load` prepares the test database from `db/schema.rb`
- `bundle exec rspec --format progress --format json --out rspec_results.json` runs the full suite
- **SimpleCov** measures code coverage with a minimum threshold (e.g., 90%) — builds fail if coverage drops
- **Mutation testing** with `mutant` gem validates test suite effectiveness beyond line coverage
- Factory Bot generates test data; DatabaseCleaner resets state between tests
- Tests are parallelized across CI jobs using `parallel_tests` or `knapsack_pro` for faster feedback
- Flaky test detection: failed tests are automatically retried once; persistent flakes are quarantined and tracked

## Step 8: Integration Testing
- Request specs validate API endpoints, response formats, status codes, and pagination
- Capybara with a headless Chrome driver tests full page interactions, form submissions, and JavaScript behavior
- **Sidekiq** jobs are tested inline using `Sidekiq::Testing.inline!` to verify background processing pipelines
- **ActionMailer** deliveries are asserted in test mode for transactional emails
- **ActiveStorage** uploads are tested against a local disk service
- Service objects and interactors are tested with real database calls
- External API calls are stubbed with **WebMock** or recorded/replayed with **VCR** cassettes
- **Contract testing** (via Pact) ensures API schemas between Rails services and consumers stay in sync

## Step 9: End-to-End (E2E) Testing
- A full Rails stack is spun up using `docker compose` with PostgreSQL, Redis, Sidekiq, and Nginx
- System specs driven by Capybara + Playwright simulate real user workflows
- Critical paths covered:
  - User registration, login, MFA enrollment
  - Core CRUD operations
  - Payment/billing flows
  - Admin and role-based access paths
  - File upload and processing
- Accessibility testing is integrated using `axe-core` via `axe-core-rspec`
- Screenshots and trace files are captured on failure and uploaded as CI artifacts
- Test environment is fully torn down after the suite completes — no state leakage

## Step 10: Security Scanning (DevSecOps Gate)
- **Static Application Security Testing (SAST)**:
  - Brakeman final report with zero high-confidence warnings required
  - Semgrep with Rails-specific and OWASP rule packs
- **Software Composition Analysis (SCA)**:
  - `bundler-audit` and `yarn audit` flag insecure dependencies
  - License compliance check ensures no GPL or incompatible licenses in production dependencies
- **Container Image Scanning**:
  - **Trivy** or **Grype** scans the Docker image for OS-level and library CVEs
  - Critical/high CVEs with available fixes block the pipeline
  - Base image age is checked — images older than 30 days trigger a rebuild
- **Secret Detection**:
  - **GitLeaks** scans the full commit history and diff for leaked credentials
  - GitHub **Push Protection** blocks pushes containing detected secrets in real time
- **Infrastructure as Code (IaC) Scanning**:
  - **Checkov** or **tfsec** scans Terraform files for DigitalOcean misconfigurations
  - Docker Compose and Kubernetes manifests are validated against CIS benchmarks
- **Dynamic Application Security Testing (DAST)**:
  - **OWASP ZAP** baseline scan runs against the E2E test environment
  - Targets OWASP Top 10: injection, broken auth, misconfig, XSS, SSRF
- All security scan results are aggregated into a unified report
- **Security quality gate**: Any critical finding blocks promotion to staging

## Step 11: Push Docker Image to DigitalOcean Container Registry (DOCR)
- CI authenticates to DOCR using **OIDC federation** or a scoped DigitalOcean API token

      doctl registry login

- Image is tagged and pushed:

      docker tag myapp:latest registry.digitalocean.com/<registry>/myapp:<commit-sha>
      docker push registry.digitalocean.com/<registry>/myapp:<commit-sha>

- On merge to `main`, the image is additionally tagged with semantic version and `latest`
- **Image signing** with Cosign (Sigstore) ensures image integrity and provenance
- DOCR garbage collection policies automatically remove untagged and expired images
- **Image provenance attestation** is generated and attached using SLSA framework standards
- Registry access is scoped: CI can push; production can only pull

## Step 12: Deploy to Staging on DigitalOcean
- **Option A — DigitalOcean App Platform**:
  - App Spec YAML is updated to reference the new image tag
  - DigitalOcean handles zero-downtime rolling deploys
- **Option B — DigitalOcean Kubernetes (DOKS)**:
  - Helm chart or Kustomize overlay is applied:

        kubectl set image deployment/myapp-staging myapp=registry.digitalocean.com/<registry>/myapp:<commit-sha>

  - **Pod Security Standards** enforce `restricted` profile — no privileged containers, no root
  - **Network Policies** isolate staging from production namespaces
- **Option C — Droplets**:
  - Deploy script SSHs (via SSH key, no passwords) into the staging Droplet
  - Pulls the new image and restarts via `docker compose up -d`
  - Firewall rules (DigitalOcean Cloud Firewall) restrict SSH to CI runner IPs only
- **Database migrations** run as a pre-deploy job:

      docker exec myapp-staging bin/rails db:migrate

- `RAILS_MASTER_KEY`, `DATABASE_URL`, `REDIS_URL` are injected via:
  - App Platform: encrypted environment variables
  - DOKS: Kubernetes Secrets (encrypted at rest with DigitalOcean-managed encryption)
  - Droplets: environment file with `600` permissions, owned by deploy user
- **Staging is network-isolated**: accessible only via VPN or IP allowlist

## Step 13: Staging Validation & Penetration Testing
- Automated smoke tests hit critical endpoints and verify HTTP 200 responses
- Rails health check endpoint (`/up`) confirms the app is booted and the database is connected
- Sidekiq dashboard is verified — workers are processing jobs, no dead queues growing
- **Performance testing**:
  - **k6** or **Apache Bench** simulates expected and peak traffic
  - Response time baselines are compared — regressions >20% fail the gate
  - Database query performance is profiled; N+1 queries are flagged by `bullet` gem
- **DAST in staging**: OWASP ZAP active scan runs a deeper crawl against the staging environment
- **Chaos testing** (optional): Randomly kill pods/containers to validate resilience and restart behavior
- QA team performs manual exploratory testing against `staging.myapp.com`
- Test results, performance reports, and security findings are posted to the PR and deployment dashboard

## Step 14: Database Migration Safety Check
- **strong_migrations** gem flags dangerous operations:
  - Removing columns without `ignored_columns` in the model first
  - Adding indexes without `CONCURRENTLY`
  - Changing column types on large tables
  - Adding `NOT NULL` constraints without defaults
- All migrations must be **backward-compatible** with the currently running version to support zero-downtime deploys
- A **database backup** is taken via DigitalOcean Managed Database automated snapshots before any destructive migration
- Point-in-time recovery window is confirmed (default: 7 days on DigitalOcean Managed DB)
- Rollback instructions (`rails db:rollback STEP=N`) are documented in the PR
- **Schema change review** is required from a DBA or database-knowledgeable reviewer for migrations touching high-traffic tables

## Step 15: Approval Gate & Change Management
- A required manual approval step is configured in GitHub Actions using `environment: production`
- **Environment protection rules**:
  - Only designated approvers (tech leads, SRE, security champion) can approve
  - Approval expires after a configurable window (e.g., 1 hour)
  - Wait timer can enforce a cooldown period between staging validation and production deploy
- Approvers review:
  - Staging test results and performance benchmarks
  - Security scan summary (SAST, SCA, DAST, container scan)
  - Migration safety report
  - Rollback plan and blast radius assessment
- Approval metadata (who, when, what) is logged immutably for **SOC 2 / audit compliance**
- **Deployment freeze windows** are enforced — no production deploys during peak traffic or maintenance periods

## Step 16: Deploy to Production on DigitalOcean
- The **same immutable, signed Docker image** from staging is deployed — no rebuild
- **Image signature is verified** before deployment using Cosign
- Deployment strategy:
  - **App Platform**: Update the App Spec; DigitalOcean handles zero-downtime rolling deploys with automatic rollback on health check failure
  - **DOKS**: Rolling update with `maxUnavailable: 0`, `maxSurge: 1`; readiness and liveness probes gate traffic
  - **Droplets behind DigitalOcean Load Balancer**: Instances updated one at a time; health checks remove unhealthy nodes from rotation
- Pre-deploy job runs `bin/rails db:migrate` against the production Managed Database
- Production environment variables injected securely (same pattern as staging)
- **Feature flags** (Flipper gem) allow new functionality to be deployed dark and enabled incrementally
- **Canary deploys** (DOKS): Route 5% of traffic to the new version; promote to 100% after validation
- Deployment event recorded:
  - Artifact SHA and version
  - Deployer identity
  - Timestamp
  - Linked PR and approval
  - SBOM and security scan summary

## Step 17: Post-Deployment Verification
- Health check endpoint (`/up`) is polled until HTTP 200 is returned
- Automated smoke tests exercise critical user flows:
  - Authentication (login, session, MFA)
  - Core business operations
  - API endpoint responses
  - Background job processing (Sidekiq heartbeat)
  - ActionCable / WebSocket connectivity (if applicable)
- **Synthetic monitoring** runs continuously from multiple DigitalOcean regions
- **Error rate comparison**: New error rates are compared against the pre-deploy baseline
  - If error rate spikes >2x, automatic rollback is triggered
- If any check fails:
  - **App Platform**: Reverts to previous deployment automatically
  - **DOKS**: `kubectl rollout undo deployment/myapp-production`
  - **Droplets**: Script re-pulls the previous image tag and restarts
- Results are posted to Slack, deployment dashboard, and the GitHub Deployment status

## Step 18: Monitoring & Observability
- **Infrastructure monitoring** (DigitalOcean Monitoring):
  - Droplet / App Platform CPU, memory, disk, bandwidth
  - Managed Database connections, replication lag, query latency
  - Managed Redis memory usage, evictions, connection count
  - Load Balancer request rates, error rates, latency distribution
- **Application Performance Monitoring (APM)**:
  - **AppSignal**, **New Relic**, or **Datadog APM** provides Rails-specific insights:
    - Controller action response times broken down by middleware, view, and database
    - N+1 query detection and slow query analysis
    - Sidekiq job duration, retry rates, and dead job tracking
    - ActionMailer delivery success/failure rates
    - ActiveStorage upload/download performance
- **Structured logging**:
  - Rails configured with `lograge` to output single-line structured JSON logs to `STDOUT`
  - Request ID propagation across logs for end-to-end traceability
  - Logs aggregated via **Papertrail**, **Datadog Logs**, or **DigitalOcean App Platform logs**
  - Sensitive data (PII, tokens) is filtered from logs using `Rails.application.config.filter_parameters`
- **Distributed tracing** (optional):
  - OpenTelemetry SDK traces requests across Rails app, Sidekiq, Redis, and PostgreSQL
  - Traces exported to Datadog, Honeycomb, or Jaeger
- **Custom dashboards** visualize:
  - Request throughput and error rates
  - Apdex score
  - Deploy markers overlaid on performance graphs
  - Business metrics (signups, conversions, key feature usage)
- **DigitalOcean Uptime Checks** monitor public endpoints from multiple global regions

## Step 19: Alerting & Incident Response
- **Alert rules** configured at multiple layers:
  - **Application**: 5xx error rate spike, p95 latency breach, Sidekiq queue depth growing
  - **Database**: Connection pool exhaustion, replication lag, high CPU/memory, slow queries
  - **Infrastructure**: Droplet/container OOM, disk >85%, load balancer 5xx spike
  - **Security**: Failed login spike (potential brute force), CSP violation reports, WAF triggers
- Alerts route to on-call engineers via **PagerDuty** or **Opsgenie** with escalation policies
- **Slack integration** posts alerts to `#incidents` channel with severity and context
- **Runbooks** document step-by-step remediation for common failures:
  - `PG::ConnectionBad` — connection pool tuning, PgBouncer restart
  - `Redis::CannotConnectError` — Redis cluster failover procedure
  - `ActionView::Template::Error` — asset pipeline debugging
  - Sidekiq memory bloat — worker restart and memory killer configuration
  - High memory / OOM — identify leaky endpoints with memory profiler
- **Incident severity classification** (SEV1-SEV4) determines response SLA and communication cadence
- **Status page** (e.g., Statuspage.io) is updated automatically or manually during incidents
- DigitalOcean Managed Database **automatic failover** handles primary database failures transparently

## Step 20: Rollback Procedures
- **Application rollback**:
  - Redeploy the previous stable Docker image using the same pipeline:

        docker pull registry.digitalocean.com/<registry>/myapp:<previous-sha>

  - Rollback is a first-class deployment — it goes through the same process, minus the approval gate (pre-authorized)
- **Database rollback**:
  - Only executed if migration is backward-incompatible:

        docker exec myapp-production bin/rails db:rollback STEP=1

  - DigitalOcean Managed Database point-in-time recovery for data corruption scenarios
  - Backup restoration tested quarterly as part of disaster recovery drills
- **Feature flag rollback** (fastest):
  - Flipper toggles disable the problematic feature instantly — no deploy needed
  - Percentage-based rollout can be dialed down to 0%
- **DNS-level rollback**: DigitalOcean Load Balancer or DNS can redirect traffic to a standby environment
- All rollback events are logged with reason, actor, timestamp, and linked incident

## Step 21: Scaling on DigitalOcean
- **Horizontal scaling**:
  - **App Platform**: `instance_count` and auto-scaling rules in App Spec
  - **DOKS**: Horizontal Pod Autoscaler (HPA) scales Rails pods on CPU/memory; Cluster Autoscaler adds nodes
  - **Droplets**: DigitalOcean Load Balancer distributes traffic; Terraform or `doctl` provisions new Droplets
- **Sidekiq workers** scaled independently from web processes:
  - Separate deployment/container for Sidekiq
  - Queue-specific workers with priority configuration
  - Autoscaling based on queue depth metrics
- **Database scaling**:
  - Vertical scaling via DigitalOcean Managed Database resize
  - Read replicas for read-heavy workloads (configured in Rails via `connects_to`)
  - **PgBouncer** for connection pooling at scale
- **Redis scaling**:
  - DigitalOcean Managed Redis with cluster mode for high availability
  - Separate Redis instances for cache, sessions, and Sidekiq to isolate failure domains
- **CDN**: DigitalOcean Spaces + CDN for static assets, user uploads, and Rails ActiveStorage blobs
- **Rate limiting**: `rack-attack` gem throttles abusive clients and protects against DDoS at the application layer

## Step 22: Compliance, Governance & Audit
- **Audit trail**: Every deployment, approval, rollback, and configuration change is logged immutably
- **SOC 2 / ISO 27001 readiness**:
  - Access to production is restricted to break-glass procedures only — no direct SSH
  - All secrets are rotated on a scheduled basis (quarterly minimum)
  - MFA is enforced for all GitHub organization members and DigitalOcean team accounts
- **Data protection**:
  - DigitalOcean Managed Database encryption at rest (AES-256) and in transit (TLS 1.2+)
  - Rails `ActiveRecord::Encryption` for application-level encryption of sensitive fields (PII, tokens)
  - Backup encryption enabled on all DigitalOcean snapshots and database backups
- **Access control**:
  - GitHub team-based permissions with least-privilege roles
  - DigitalOcean team access scoped by project
  - Kubernetes RBAC restricts namespace and resource access
  - Production database credentials are rotated and never shared — injected via secrets management only
- **Vulnerability management**:
  - Dependabot alerts triaged within SLA (critical: 24h, high: 7d, medium: 30d)
  - Container base images rebuilt weekly to pick up OS-level patches
  - Quarterly penetration testing by an external security firm
- **Policy as Code**: Open Policy Agent (OPA) or Kyverno enforces cluster-wide policies in DOKS (no privileged pods, required labels, resource limits)

## Step 23: Disaster Recovery & Business Continuity
- **Recovery Point Objective (RPO)**: < 1 hour (DigitalOcean Managed Database automated backups every hour)
- **Recovery Time Objective (RTO)**: < 30 minutes for application; < 1 hour for full stack
- **Multi-region strategy**:
  - DigitalOcean Spaces replicates static assets across regions
  - Database read replicas in a secondary region for read failover
  - DNS failover via DigitalOcean or Cloudflare for regional outages
- **Disaster recovery drills**:
  - Quarterly: Restore database from backup and verify data integrity
  - Semi-annually: Full stack recovery in a secondary region
  - Annually: Tabletop exercise simulating major outage scenario
- **Infrastructure as Code**: Entire stack is reproducible from Terraform — no manual configuration
- **Runbook**: Step-by-step disaster recovery procedure is documented, versioned, and tested

## Step 24: Feedback & Continuous Improvement
- **DORA metrics** tracked via deployment pipeline data:
  - **Deployment frequency**: Target multiple deploys per day
  - **Lead time for changes**: Commit to production < 1 hour
  - **Change failure rate**: Target < 5%
  - **Mean time to recovery (MTTR)**: Target < 30 minutes
- **Post-incident reviews**: Blameless retrospectives within 48 hours of any SEV1/SEV2 incident
  - Root cause analysis with 5 Whys
  - Action items tracked in GitHub Issues with owners and due dates
- **Pipeline optimization**:
  - Parallel RSpec with `knapsack_pro` for optimal test splitting
  - Docker layer caching and BuildKit for faster image builds
  - Dependency caching tuned for cache hit rate > 95%
  - Flaky tests quarantined, tracked in a dashboard, and fixed within SLA
- **Security posture reviews**: Monthly review of scan findings, dependency alerts, and access logs
- **Cost optimization**: Monthly review of DigitalOcean billing; right-size Droplets, database, and Redis based on actual utilization
- **Pipeline-as-code**: All workflows, Dockerfiles, Helm charts, Terraform, and App Spec files are versioned, reviewed via PR, and subject to the same rigor as application code
- **Team enablement**: DevSecOps guild meets bi-weekly to share learnings, review new tools, and update standards

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