# Docker Build and Push Reusable Workflow

A production-ready GitHub Actions reusable workflow for building and pushing Docker images with intelligent tagging, multi-platform support, and comprehensive features.

## Features

- ✅ Intelligent date-based tagging (`YYYYMMDD_N`)
- ✅ Automatic daily build version increment
- ✅ Git SHA tracking
- ✅ Branch-aware tags
- ✅ Multi-platform builds (AMD64, ARM64, ARM/v7)
- ✅ Build caching for faster builds
- ✅ Flexible configuration
- ✅ Comprehensive build summaries
- ✅ OCI image labels

## Quick Start

### 1. Setup Secrets

Navigate to: **Repository → Settings → Secrets and variables → Actions → Secrets**

Add these repository secrets:

| Secret | Description | Example |
|--------|-------------|---------|
| `REGISTRY_USERNAME` | Docker registry username | `myuser` |
| `REGISTRY_PASSWORD` | Docker registry password/token | `ghp_xxxxx` or token |
| `REGISTRY_URL` | Docker registry URL | `docker.io`, `ghcr.io`, `registry.example.com` |

### 2. Create Workflow

Create `.github/workflows/build.yml` in your repository:
```yaml
name: Build Docker Image

on:
  push:
    branches:
      - main
      - develop
  pull_request:
    branches:
      - main
  workflow_dispatch:

jobs:
  docker:
    uses: your-org/workflows/.github/workflows/docker_build_push.yml@main
    with:
      image_name: my-application
      dockerfile: Dockerfile
      context: .
      platforms: linux/amd64
      push_latest: true
      include_branch_tag: true
      enable_cache: true
    secrets:
      REGISTRY_USERNAME: ${{ secrets.REGISTRY_USERNAME }}
      REGISTRY_PASSWORD: ${{ secrets.REGISTRY_PASSWORD }}
      REGISTRY_URL: ${{ secrets.REGISTRY_URL }}
```

## Configuration

### Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `image_name` | ✅ | - | Docker image name (without registry URL) |
| `dockerfile` | ❌ | `Dockerfile` | Path to Dockerfile |
| `context` | ❌ | `.` | Build context path |
| `platforms` | ❌ | `linux/amd64` | Target platforms (comma-separated) |
| `push_latest` | ❌ | `true` | Push `latest` tag |
| `include_branch_tag` | ❌ | `true` | Include branch name in tags |
| `enable_cache` | ❌ | `true` | Enable build cache |

### Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `REGISTRY_USERNAME` | ✅ | Docker registry username |
| `REGISTRY_PASSWORD` | ✅ | Docker registry password or token |
| `REGISTRY_URL` | ✅ | Docker registry URL |

### Outputs

| Output | Description | Example |
|--------|-------------|---------|
| `image_tag` | Primary generated tag | `20241125_1` |
| `image_full_name` | Full image name with tag | `registry.example.com/my-app:20241125_1` |
| `image_sha_tag` | Tag with commit SHA | `20241125_1-a1b2c3d` |
| `branch_name` | Sanitized branch name | `main`, `develop` |

## Tag Strategy

Generated tags:

1. **Date Tag**: `20241125_1` (primary)
2. **SHA Tag**: `20241125_1-a1b2c3d`
3. **Branch Tag**: `main-20241125_1`
4. **Branch Only**: `main`
5. **Latest**: `latest` (optional)

## Usage Examples

### Basic Usage
```yaml
jobs:
  docker:
    uses: your-org/workflows/.github/workflows/docker_build_push.yml@main
    with:
      image_name: my-app
    secrets: inherit
```

### Multi-Platform Build
```yaml
jobs:
  docker:
    uses: your-org/workflows/.github/workflows/docker_build_push.yml@main
    with:
      image_name: my-app
      platforms: linux/amd64,linux/arm64
    secrets: inherit
```

### Custom Dockerfile
```yaml
jobs:
  docker:
    uses: your-org/workflows/.github/workflows/docker_build_push.yml@main
    with:
      image_name: my-app
      dockerfile: docker/prod.Dockerfile
      context: .
    secrets: inherit
```

### With Deployment Step
```yaml
jobs:
  build:
    uses: your-org/workflows/.github/workflows/docker_build_push.yml@main
    with:
      image_name: my-app
    secrets: inherit
  
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/my-app \
            my-app=${{ needs.build.outputs.image_full_name }}
      
      - name: Verify deployment
        run: |
          kubectl rollout status deployment/my-app
```

### Branch-Specific Configuration
```yaml
jobs:
  build-dev:
    if: github.ref == 'refs/heads/develop'
    uses: your-org/workflows/.github/workflows/docker_build_push.yml@main
    with:
      image_name: my-app-dev
      push_latest: false
    secrets: inherit
  
  build-prod:
    if: github.ref == 'refs/heads/main'
    uses: your-org/workflows/.github/workflows/docker_build_push.yml@main
    with:
      image_name: my-app
      push_latest: true
    secrets: inherit
```

### With Testing and Scanning
```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: npm test
  
  build:
    needs: test
    uses: your-org/workflows/.github/workflows/docker_build_push.yml@main
    with:
      image_name: my-app
    secrets: inherit
  
  scan:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Run Trivy scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ needs.build.outputs.image_full_name }}
          format: 'sarif'
          output: 'trivy-results.sarif'
      
      - name: Upload results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
```

## Registry Configuration

### Docker Hub
```yaml
secrets:
  REGISTRY_USERNAME: your-dockerhub-username
  REGISTRY_PASSWORD: your-dockerhub-token
  REGISTRY_URL: docker.io
```

Image: `docker.io/your-dockerhub-username/my-app:20241125_1`

### GitHub Container Registry
```yaml
secrets:
  REGISTRY_USERNAME: ${{ github.actor }}
  REGISTRY_PASSWORD: ${{ secrets.GITHUB_TOKEN }}
  REGISTRY_URL: ghcr.io
```

Image: `ghcr.io/your-org/my-app:20241125_1`

### Private Registry
```yaml
secrets:
  REGISTRY_USERNAME: admin
  REGISTRY_PASSWORD: your-password
  REGISTRY_URL: registry.example.com
```

Image: `registry.example.com/my-app:20241125_1`

### Amazon ECR
```yaml
jobs:
  setup-ecr:
    runs-on: ubuntu-latest
    outputs:
      registry: ${{ steps.login-ecr.outputs.registry }}
      password: ${{ steps.login-ecr.outputs.docker_password }}
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2
  
  build:
    needs: setup-ecr
    uses: your-org/workflows/.github/workflows/docker_build_push.yml@main
    with:
      image_name: my-app
    secrets:
      REGISTRY_USERNAME: AWS
      REGISTRY_PASSWORD: ${{ needs.setup-ecr.outputs.password }}
      REGISTRY_URL: ${{ needs.setup-ecr.outputs.registry }}
```

## Troubleshooting

### Build fails with "permission denied"

**Solution**: Verify registry credentials have push permissions.

### Multi-platform build fails

**Solution**: Ensure base image supports target platforms. Check dependencies.

### Cache not working

**Solution**: Verify registry supports caching and credentials have read/write access.

### Tags not appearing

**Solution**: Check workflow logs. Verify `push_latest` and `include_branch_tag` settings.

### Rate limit exceeded

**Solution**: For Docker Hub, use authenticated pulls or upgrade plan.

## Best Practices

1. **Security**
   - Use tokens instead of passwords
   - Rotate credentials regularly
   - Use least privilege access

2. **Performance**
   - Enable caching for faster builds
   - Use multi-stage Dockerfiles
   - Optimize layer ordering

3. **Organization**
   - Use consistent naming conventions
   - Implement tag cleanup policies
   - Document image variants

4. **Monitoring**
   - Set up vulnerability scanning
   - Monitor registry storage quotas
   - Track build times

5. **Version Control**
   - Pin workflow version: `@v1` instead of `@main`
   - Test changes in feature branches
   - Document breaking changes

## Advanced Examples

### Matrix Builds
```yaml
jobs:
  build:
    strategy:
      matrix:
        include:
          - app: frontend
            dockerfile: frontend/Dockerfile
          - app: backend
            dockerfile: backend/Dockerfile
          - app: worker
            dockerfile: worker/Dockerfile
    uses: your-org/workflows/.github/workflows/docker_build_push.yml@main
    with:
      image_name: ${{ matrix.app }}
      dockerfile: ${{ matrix.dockerfile }}
    secrets: inherit
```

### Conditional Builds
```yaml
jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      docker: ${{ steps.filter.outputs.docker }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v2
        id: filter
        with:
          filters: |
            docker:
              - 'Dockerfile'
              - 'src/**'
  
  build:
    needs: changes
    if: needs.changes.outputs.docker == 'true'
    uses: your-org/workflows/.github/workflows/docker_build_push.yml@main
    with:
      image_name: my-app
    secrets: inherit
```

### With Notifications
```yaml
jobs:
  build:
    uses: your-org/workflows/.github/workflows/docker_build_push.yml@main
    with:
      image_name: my-app
    secrets: inherit
  
  notify:
    needs: build
    runs-on: ubuntu-latest
    if: always()
    steps:
      - name: Send Slack notification
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Build ${{ needs.build.result }}: ${{ needs.build.outputs.image_full_name }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

## License

MIT