# 🚀 Some Reusable GitHub Actions CI/CD Pipelines

Welcome to my personal collection of reusable GitHub Actions workflows! This repository contains a comprehensive set of CI/CD pipelines that I've developed to automate the entire software development lifecycle of my own projects - from testing and linting to building, deploying, and maintaining applications in production.

## 🎯 What's Inside

This collection includes battle-tested workflows for:

- **🧪 Testing & Quality Assurance** - Automated testing and code quality checks
- **🔍 Linting & Security** - Code style enforcement and security auditing  
- **🏗️ Container Building** - Docker image building and registry publishing
- **☸️ Kubernetes Deployments** - Automated deployments to Kubernetes clusters
- **📦 Release Management** - Automated versioning and release creation
- **⚙️ Infrastructure Maintenance** - Cluster maintenance and monitoring

All workflows are designed to be **reusable**, **configurable**, and **production-ready**.

## 🏗️ Architecture

### Reusable Workflows
Located in `.github/workflows/`, these workflows can be called from any repository:

- **`test-lint-rails.yaml`** - Comprehensive Rails testing with PostgreSQL, security audits, and code quality reports
- **`build-push.yaml`** - Multi-platform Docker image building and registry publishing  
- **`deploy.yaml`** - Kubernetes deployments with Helm charts and environment management
- **`release-please.yaml`** - Automated version bumping and release creation

### Templates
The `templates/` directory contains ready-to-use examples showing how to consume these workflows in your own projects.

## 🚀 Quick Start

### 1. Using a Reusable Workflow

Copy any template from the `templates/` directory and customize it for your project:

```yaml
# Example: .github/workflows/ci.yaml
name: CI Pipeline
on: [push, pull_request]

jobs:
  test:
    uses: joel-grant/github-actions/.github/workflows/test-lint-rails.yaml@main
    with:
      repo_name: "my-awesome-app"
    secrets:
      dockerhub_username: ${{ secrets.DOCKERHUB_USERNAME }}
      dockerhub_token: ${{ secrets.DOCKERHUB_TOKEN }}
```

### 2. Available Templates

- **`rails-test-lint-template.yaml`** - Complete Rails CI pipeline
- **`build-push-template.yaml`** - Docker build and publish
- **`deploy-template.yaml`** - Kubernetes deployment
- **`release-template.yaml`** - Automated releases

## 📋 Features

### 🔧 Rails Testing Pipeline
- **Multi-service testing** with PostgreSQL and external APIs
- **Security auditing** with Bundler Audit and Brakeman
- **Code quality** analysis with RubyCritic
- **Configurable linting** with RuboCop (optional)
- **GitHub Pages** integration for quality reports

### 🐳 Container Operations  
- **Multi-platform builds** (AMD64, ARM64)
- **Semantic tagging** with commit hashes and latest
- **Registry flexibility** (DockerHub, private registries)
- **Build optimization** with Docker Buildx

### ☸️ Kubernetes Deployments
- **Environment-specific** deployments (prod, staging, dev)  
- **Helm chart** integration with value customization
- **Secret management** for sensitive configuration
- **Rolling deployments** with health checks

### 📈 Release Management
- **Conventional commits** for automated versioning
- **Changelog generation** from commit messages  
- **Multi-package** support with release-please
- **GitHub Releases** with automated assets

## 🛠️ Configuration

Each workflow accepts inputs and secrets for maximum flexibility. Check the templates for complete configuration examples, including:

- Database connections and service dependencies
- Container registry credentials  
- Kubernetes cluster access
- Environment-specific variables
- Security and monitoring integrations

## 🤝 Contributing & Usage

**This is a demo/personal project**, but you're absolutely welcome to:

- ✨ **Use these workflows** in your own projects
- 🎨 **Draw inspiration** for your CI/CD pipelines  
- 🔧 **Adapt and modify** them for your specific needs
- 💡 **Share feedback** or suggestions for improvements
- 🐛 **Report issues** if you find any problems

No attribution required, but always appreciated! 

## 📚 Documentation

Detailed documentation for each workflow can be found in the template files, including:

- Required secrets and environment variables
- Configuration options and examples  
- Prerequisites and setup instructions
- Common usage patterns and best practices

## 🏷️ Workflow Status

All workflows are actively maintained and tested in production environments. They follow GitHub Actions best practices and are designed for reliability and scalability.

---

**Happy Automating!** 🎉

*Feel free to explore, use, and adapt these workflows for your own projects. If you find them helpful, consider starring the repository or sharing it with others who might benefit from these automation patterns.*
