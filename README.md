# My GitHub Actions Modules for Rails Projects

This is my personal collection of reusable GitHub Actions workflows that I use to automate Rails application deployments.

## What's Included

- **Testing & Linting** - Rails testing with optional PostgreSQL, security audits, and code quality
- **Docker Build & Push** - Container building and registry publishing
- **Kubernetes Deployment** - Automated deployments with Helm  
- **Terraform Infrastructure** - Infrastructure as code with plan/apply workflows
- **Helm Charts** - Chart testing and publishing to multiple registries
- **Release Management** - Automated versioning and releases

## Usage

### Basic Rails CI/CD Pipeline

```yaml
# .github/workflows/ci.yaml
name: CI/CD
on: [push, pull_request]

jobs:
  test:
    uses: joel-grant/github-actions/.github/workflows/test-lint-rails.yaml@main
    with:
      repo_name: "my-rails-app"
    secrets: inherit

  build:
    needs: test
    uses: joel-grant/github-actions/.github/workflows/build-push.yaml@main
    secrets: inherit

  deploy:
    needs: build
    if: github.ref == 'refs/heads/main'
    uses: joel-grant/github-actions/.github/workflows/deploy.yaml@main
    with:
      environment: production
    secrets: inherit
```

### Cache Control

Build without cache when needed:

```yaml
jobs:
  build:
    uses: joel-grant/github-actions/.github/workflows/build-push.yaml@main
    with:
      no_cache: true  # Force fresh build
    secrets: inherit
```

### Database Configuration

The Rails testing workflow supports different database configurations:

```yaml
# No database (API-only apps)
jobs:
  test:
    uses: joel-grant/github-actions/.github/workflows/test-lint-rails.yaml@main
    with:
      repo_name: "my-api-app"
      use_postgresql: false
    secrets: inherit

# Custom database URL
jobs:
  test:
    uses: joel-grant/github-actions/.github/workflows/test-lint-rails.yaml@main
    with:
      repo_name: "my-app"
      use_postgresql: false
      database_url: "mysql2://user:pass@localhost:3306/test_db"
    secrets: inherit
```


## Configuration

Each workflow uses `secrets: inherit` to automatically pass repository secrets. Check the workflow files for available inputs and required secrets.

## Files

- `.github/workflows/` - Reusable workflow definitions
- `templates/` - Example workflow templates
- `examples/` - Usage pattern examples
- `docs/` - Additional documentation
