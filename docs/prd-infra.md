# Infrastructure & Deployment PRD

**Tag**: `infra`
**Folder**: `/`, `deployments/`, `.github/`
**Dependencies**: None (first wave)

## Overview

Establish the foundational infrastructure for Meridian including build tooling, local development environment with Tilt, CI/CD pipelines, and Kubernetes deployment manifests.

## Scope

### In Scope
- Go 1.23+ module initialization with proper structure
- Build tooling (Makefile, task runner)
- Tilt configuration for local Kubernetes development
- Kubernetes manifests (deployments, services, configmaps)
- GitHub Actions CI/CD pipeline
- Docker multi-stage builds for production images
- Development environment documentation

### Out of Scope
- Application code (handled by other tags)
- Database schema (handled by platform tag)
- Proto definitions (handled by api-contracts tag)

## Technical Requirements

### TR-1: Go Module Structure
- Initialize `go.mod` with `github.com/bjcoombs/meridian`
- Go 1.23+ with workspace support
- Dependency management with go.mod/go.sum
- Vendor directory optional (decide based on build speed)

### TR-2: Build Tooling
- Makefile with targets:
  - `make build` - Build all services
  - `make test` - Run all tests
  - `make lint` - Run golangci-lint
  - `make proto` - Generate proto code
  - `make docker` - Build Docker images
  - `make deploy-local` - Deploy to local Tilt
- golangci-lint configuration with strict rules
- Pre-commit hooks for formatting and linting

### TR-3: Tilt Configuration
- Tiltfile that orchestrates:
  - CockroachDB or YugabyteDB (single node for dev)
  - Redis for idempotency
  - Kafka + Zookeeper
  - All Go services with live reload
- Fast rebuild on code changes (< 5 seconds)
- Unified logs and resource dashboard
- Port-forwarding for local testing

### TR-4: Kubernetes Manifests
- Base manifests in `deployments/k8s/base/`:
  - Deployment for each service
  - Service (ClusterIP) for each
  - ConfigMap for configuration
  - Secret references (not actual secrets)
- Kustomize overlays for environments:
  - `dev/` - Development settings
  - `staging/` - Staging configuration
  - `production/` - Production hardening
- Health check probes configured
- Resource limits and requests defined

### TR-5: Docker Images
- Multi-stage Dockerfile:
  - Stage 1: Build (Go compilation)
  - Stage 2: Runtime (minimal distroless or alpine)
- Image size < 50MB for services
- Security scanning with trivy
- Image tagging strategy (git SHA + semver)

### TR-6: CI/CD Pipeline
- GitHub Actions workflows:
  - `test.yml` - Run on every PR
    - Checkout code
    - Run tests
    - Upload coverage
  - `lint.yml` - Run linters and security scans
  - `build.yml` - Build and push Docker images
  - `deploy.yml` - Deploy to staging/production
- Required status checks for PR merge
- Automatic rollback on health check failure

### TR-7: Local Development Setup
- README with setup instructions
- Script for installing dependencies (Tilt, kubectl, etc.)
- Environment variable template (.env.example)
- Quick start guide (< 5 minutes to running system)

## Acceptance Criteria

- [ ] Go module initialized with correct structure
- [ ] Makefile provides all essential build targets
- [ ] Tilt starts all dependencies and services locally
- [ ] Kubernetes manifests deploy successfully to local cluster
- [ ] Docker images build and are security scanned
- [ ] CI pipeline passes on sample PR
- [ ] Developer can clone repo and run `tilt up` successfully
- [ ] Documentation explains setup and common tasks

## Success Metrics

- Time from `git clone` to running system: < 5 minutes
- Rebuild time on code change: < 5 seconds
- CI pipeline duration: < 10 minutes
- Docker image size: < 50MB per service
- Zero security vulnerabilities in base images

## Folder Structure

```
meridian/
├── .github/
│   └── workflows/
│       ├── test.yml
│       ├── lint.yml
│       ├── build.yml
│       └── deploy.yml
├── deployments/
│   ├── docker/
│   │   └── Dockerfile
│   └── k8s/
│       ├── base/
│       │   ├── deployment.yaml
│       │   ├── service.yaml
│       │   └── configmap.yaml
│       └── overlays/
│           ├── dev/
│           ├── staging/
│           └── production/
├── Tiltfile
├── Makefile
├── go.mod
├── go.sum
├── .golangci.yml
└── README.md
```

## Notes

- Tilt is mandatory for developer experience
- Use Kustomize for environment-specific configuration
- Security scanning is non-negotiable for production
- Fast feedback loops are critical for TDD
