# Eranin CI

## Overview
Eranin CI provides standardized GitHub Actions workflows used across all Eranin projects. This repository centralizes reusable pipelines such as Docker Build & Push, security scanning, code quality checks, and future enterprise‑grade CI/CD automations.

## Features
- Unified reusable workflows via `workflow_call`
- Deterministic image tagging
- Multi‑platform Docker Buildx support
- Secure (and insecure‑registry compatible) Docker push
- Extensible design for scanning, linting, SBOM generation, and more

## Repository Structure
```
.
├── .github/
│   └── workflows/
│       └── common_ci.yml      # Reusable CI workflow
└── README.md
```

## Usage: Docker Build & Push Workflow
This pipeline builds and pushes Docker images to an internal Eranin container registry.

### Triggering Workflow
In your project repo, create a workflow:

```
name: Build Backend

on:
  push:
    branches: [ main ]

jobs:
  build:
    uses: eranin/eranin-ci/.github/workflows/common_ci.yml@main
    with:
      image_name: "emoney-be"
      dockerfile: "Dockerfile"
      context: "."
      platforms: "linux/amd64"
    secrets:
      REGISTRY_USERNAME: ${{ secrets.REGISTRY_USERNAME }}
      REGISTRY_PASSWORD: ${{ secrets.REGISTRY_PASSWORD }}
      REGISTRY_URL: ${{ secrets.REGISTRY_URL }}
```

## Inputs
| Name        | Required | Description |
|-------------|----------|-------------|
| `image_name` | Yes | Docker image name (without registry prefix) |
| `dockerfile` | No | Dockerfile path (default: `Dockerfile`) |
| `context` | No | Build context (default: `.`) |
| `platforms` | No | Target build platforms (default: `linux/amd64`) |

## Outputs
| Name | Description |
|------|-------------|
| `image_tag` | Auto‑generated tag: `YYYYMMDD_<commit_count>` |

## Architecture Overview
### Image Tagging
- Ensures predictable versioning across all Eranin deployments  
- Tag format: `20251130_52`  

### Insecure Registry Support
Some internal environments use HTTP registries.  
`common_ci.yml` configures Docker daemon & Buildx to allow pushing insecure images safely in CI.

### Buildx Builder
Provides:
- Multi‑platform builds  
- Parallel layer caching  
- Configured BuildKit with registry-specific settings  

### Security (Future Roadmap)
Planned modules:
- `security_scan.yml` – SAST/DAST integration  
- `dependency_audit.yml` – Vulnerability audits  
- `sbom_generate.yml` – SBOM creation (CycloneDX)  
- `license_scan.yml` – License compliance checker  

## Best Practices
- Always use automated image tags instead of manual versions  
- Rotate registry credentials regularly  
- Enforce workflow dispatch for production releases  
- Use environment protection rules for deployment pipelines  

## Troubleshooting
### 1. “Access Denied” when pushing image
Check:
- Registry URL with correct port  
- HTTP/HTTPS mismatch  
- Credentials stored in repo secrets  

### 2. Buildx fails to start
Run locally:
```
docker buildx ls
docker buildx create --use
```

### 3. Tag not generated
Make sure repository is not shallow-cloned.

## Contact
For support, reach out to **Eranin DevOps Team**.
