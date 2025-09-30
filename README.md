# Docker Build and Push Action

[![Release](https://github.com/KoalaOps/docker-build-push-action/actions/workflows/release.yml/badge.svg)](https://github.com/KoalaOps/docker-build-push-action/actions/workflows/release.yml)

A thin wrapper around the official `docker/build-push-action` that simplifies common use cases while maintaining full compatibility.

## Why This Wrapper?

1. **Automatic tag generation**: Smart defaults for latest, SHA tags, and multi-registry support
2. **Simplified inputs**: Clean interface without complex configuration
3. **Leverages docker/metadata-action**: For consistent label and tag generation
4. **Input validation**: Prevents invalid configurations and common mistakes
5. **Multi-arch support**: Automatically sets up QEMU when needed
6. **Better summaries**: Clear GitHub Actions summaries with all build details
7. **Security features**: Support for SBOM and provenance attestations

## Prerequisites

**Important**: You must authenticate with your container registry before using this action. This action does not handle authentication.

```yaml
# For GitHub Container Registry
- uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}

# For AWS ECR (using OIDC)
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ vars.AWS_BUILD_ROLE }}
    aws-region: us-east-1

# For Docker Hub
- uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}
```

## Usage

### Automatic Tag Generation (Recommended)

The simplest way to use this action - just provide the image repository and primary tag. The action automatically adds useful tags like `:latest` (on default branch) and `:sha-<commit>` for traceability.

```yaml
# Single registry
- uses: KoalaOps/docker-build-push-action@v1
  with:
    image: myregistry.io/myapp
    base_tag: v1.2.3
    # Automatically generates:
    # - myregistry.io/myapp:v1.2.3
    # - myregistry.io/myapp:sha-abc1234 (first 7 chars of commit SHA)
    # - myregistry.io/myapp:latest (if on default branch, usually 'main')

# Multiple registries (newline-delimited)
# Note: You must authenticate to each registry before this action
- uses: KoalaOps/docker-build-push-action@v1
  with:
    image: |
      ghcr.io/${{ github.repository }}
      123456789.dkr.ecr.us-east-1.amazonaws.com/myapp
      myregistry.azurecr.io/myapp
    base_tag: v1.2.3
    # Generates the same 3 tags for each registry
```

### Manual Tags (Explicit Control)

```yaml
- uses: KoalaOps/docker-build-push-action@v1
  with:
    # Tags are newline-delimited (one per line)
    tags: |
      myregistry.io/myapp:latest
      myregistry.io/myapp:v1.2.3
      ghcr.io/${{ github.repository }}:v1.2.3
    context: .
    dockerfile: Dockerfile
```

## Inputs

### Mode Selection (choose one)

#### Automatic Tag Generation (Recommended)
| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `image` | Base image repository (e.g., `ghcr.io/org/app`). Supports multiple repositories (newline-delimited) | Yes* | - |
| `base_tag` | Primary tag (e.g., `v1.2.3`) | Yes* | - |
| `tag_latest_on_default_branch` | Add `:latest` tag when building from default branch (main/master) | No | `true` |
| `tag_sha` | Add `:sha-<short>` tag for traceability | No | `true` |
| `include_ref_tags` | Generate branch/PR tags (e.g., `pr-123`, `feature-branch`) | No | `false` |
| `include_semver_tags` | Generate major/minor tags from semver (e.g., `1`, `1.2` from `v1.2.3`) | No | `true` |

#### Manual Tags
| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `tags` | Explicit list of image:tag combinations (newline-delimited) | Yes* | - |

*Must provide either (`image` + `base_tag`) OR `tags`

### Build Configuration

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `context` | Build context path | No | `.` |
| `dockerfile` | Path to Dockerfile | No | `Dockerfile` |
| `platforms` | Comma-separated platforms | No | `linux/amd64` |
| `build_args` | Multiline KEY=VALUE build args | No | - |
| `target` | Target build stage | No | - |
| `secrets` | Build secrets (multiline) | No | - |
| `ssh` | SSH agent sockets/keys | No | - |

### Cache Configuration

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `cache_from` | Cache sources | No | `type=gha` |
| `cache_to` | Cache destinations | No | `type=gha,mode=max` |
| `no_cache` | Disable cache | No | `false` |

### Push Configuration

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `push` | Push to registry | No | `true` |
| `load` | Load into Docker daemon (mutually exclusive with push) | No | `false` |
| `pull` | Always pull newer base images | No | `false` |

### Security Configuration

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `provenance` | Enable provenance attestation | No | `false` |
| `sbom` | Generate SBOM attestation | No | `false` |

### Advanced Configuration

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `setup_buildx_with` | Custom configuration for docker/setup-buildx-action (YAML format) | No | - |

## Outputs

| Output | Description |
|--------|-------------|
| `imageid` | Image ID |
| `digest` | Image content-addressable digest |
| `metadata` | Build result metadata |
| `tags_list` | Newline-delimited list of image:tag combinations |

## Examples

### Using Automatic Tag Generation

```yaml
# Simple case - generates v1.2.3, sha-abc1234, and latest (if on default branch)
- uses: KoalaOps/docker-build-push-action@v1
  with:
    image: ghcr.io/${{ github.repository }}
    base_tag: v${{ github.event.release.tag_name }}

# Include branch/PR tags for development builds
- uses: KoalaOps/docker-build-push-action@v1
  with:
    image: myregistry.io/myapp
    base_tag: ${{ github.ref_name }}-${{ github.run_number }}
    include_ref_tags: true  # Adds branch name or pr-123 tags

# Clean release tags only
- uses: KoalaOps/docker-build-push-action@v1
  with:
    image: myregistry.io/myapp
    base_tag: v1.2.3
    tag_sha: false       # No SHA tag
    tag_latest_on_default_branch: true  # Still add latest on default branch
    include_semver_tags: true  # Also generates '1' and '1.2' tags
```

### Multi-Registry with Automatic Tags

```yaml
# Authenticate to each registry first
- uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}

- uses: KoalaOps/cloud-login@v1
  with:
    provider: aws
    region: us-east-1
    aws_role_to_assume: ${{ vars.AWS_BUILD_ROLE }}
    login_to_container_registry: true

# Then push to all registries with the same tags
- uses: KoalaOps/docker-build-push-action@v1
  with:
    image: |
      ghcr.io/${{ github.repository }}
      123456789.dkr.ecr.us-east-1.amazonaws.com/myapp
    base_tag: v1.2.3
```

### Build for Multiple Architectures

```yaml
- uses: KoalaOps/docker-build-push-action@v1
  with:
    tags: myregistry.io/myapp:latest
    platforms: linux/amd64,linux/arm64,linux/arm/v7
    push: true
```

### Build with Build Arguments

```yaml
- uses: KoalaOps/docker-build-push-action@v1
  with:
    tags: myregistry.io/myapp:${{ github.sha }}
    build_args: |
      VERSION=${{ github.ref_name }}
      BUILD_DATE=${{ github.event.head_commit.timestamp }}
      VCS_REF=${{ github.sha }}
```

### Build Specific Target Stage

```yaml
- uses: KoalaOps/docker-build-push-action@v1
  with:
    tags: myregistry.io/myapp:test
    target: test
    load: true  # Load for local testing
    push: false
```

### Using with Matrix Strategy

```yaml
strategy:
  matrix:
    include:
      - registry: ghcr.io
        image: ${{ github.repository }}
      - registry: docker.io
        image: myorg/myapp

steps:
  - uses: KoalaOps/docker-build-push-action@v1
    with:
      tags: ${{ matrix.registry }}/${{ matrix.image }}:${{ github.ref_name }}
```

### Using Output in Downstream Steps

```yaml
- id: build
  uses: KoalaOps/docker-build-push-action@v1
  with:
    tags: myregistry.io/myapp:v1.2.3

- name: Deploy image
  run: |
    echo "Deploying images:"
    echo "${{ steps.build.outputs.tags_list }}"
    # Use the digest for immutable reference
    kubectl set image deployment/myapp app=myregistry.io/myapp@${{ steps.build.outputs.digest }}
```

### With Security Attestations

```yaml
# Generate SBOM and provenance for supply chain security
- uses: KoalaOps/docker-build-push-action@v1
  with:
    image: myregistry.io/myapp
    base_tag: v1.2.3
    provenance: true  # Generate provenance attestation
    sbom: true        # Generate SBOM attestation
```

### Custom Buildx Configuration

```yaml
# Configure buildx with custom driver and settings
- uses: KoalaOps/docker-build-push-action@v1
  with:
    image: myregistry.io/myapp
    base_tag: v1.2.3
    setup_buildx_with: |
      driver: docker-container
      driver-opts: |
        network=host
        image=moby/buildkit:v0.12.0
      install: true
      platforms: linux/amd64,linux/arm64

# Use remote BuildKit instance
- uses: KoalaOps/docker-build-push-action@v1
  with:
    image: myregistry.io/myapp
    base_tag: v1.2.3
    setup_buildx_with: |
      driver: remote
      endpoint: tcp://buildkit.example.com:8125

# Configure buildx with specific builder instance
- uses: KoalaOps/docker-build-push-action@v1
  with:
    image: myregistry.io/myapp
    base_tag: v1.2.3
    setup_buildx_with: |
      use: true
      driver: kubernetes
      driver-opts: |
        namespace=buildkit
        replicas=2
```

## Migration from docker/build-push-action

This action is fully compatible with `docker/build-push-action`. To migrate:

1. Change the action reference
2. No other changes needed - all inputs are compatible
3. Optionally simplify complex tag configurations using our `tags` input

### Before (docker/build-push-action)
```yaml
- uses: docker/build-push-action@v5
  with:
    tags: |
      user/repo:latest
      user/repo:${{ github.sha }}
```

### After (KoalaOps/docker-build-push-action)
```yaml
- uses: KoalaOps/docker-build-push-action@v1
  with:
    tags: |
      user/repo:latest
      user/repo:${{ github.sha }}
```

## Label Generation

This action uses docker/metadata-action to automatically generate rich OCI labels including:
- `org.opencontainers.image.created` - Build timestamp  
- `org.opencontainers.image.revision` - Git SHA
- `org.opencontainers.image.source` - Repository URL
- `org.opencontainers.image.url` - Build run URL
- `org.opencontainers.image.version` - Git ref name
- `org.opencontainers.image.title` - Repository name
- `org.opencontainers.image.description` - Repository description
- `org.opencontainers.image.licenses` - Repository license (if detected)
- `org.opencontainers.image.authors` - Repository authors

## Notes

- Automatically sets up Docker Buildx and QEMU (for multi-arch builds)
- Registry authentication must be configured separately for each registry
- Cannot use `push: true` and `load: true` together (validated automatically)
- When using multi-registry support, the same image layers are pushed to all registries
- Uses docker/metadata-action for consistent label and tag generation
- Order of tags is preserved (using stable de-duplication)