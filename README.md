# Docker Build and Push Action

A thin wrapper around the official `docker/build-push-action` that simplifies common use cases while maintaining full compatibility.

## Why This Wrapper?

1. **Simplified inputs**: Use a simple `tags` list instead of complex JSON
2. **Dual mode**: Supports both simple tags and complex JSON configurations
3. **Smart label generation**: Uses docker/metadata-action for rich OCI labels or falls back to built-in
4. **Input validation**: Prevents invalid push+load combinations
5. **Multi-arch support**: Automatically sets up QEMU when needed
6. **Better summaries**: Provides clear GitHub Actions summaries
7. **KoalaOps defaults**: Optimized defaults for our use cases

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

### Simple Mode (90% of use cases)

```yaml
- uses: KoalaOps/docker-build-push-action@v1
  with:
    # Tags are newline-delimited (one per line)
    tags: |
      myregistry.io/myapp:latest
      myregistry.io/myapp:v1.2.3
    context: .
    dockerfile: Dockerfile
```

### Advanced Mode (Multi-registry with JSON)

```yaml
- uses: KoalaOps/docker-build-push-action@v1
  with:
    targets_json: |
      [
        {"image": "ghcr.io/myorg/myapp", "tag": "v1.2.3"},
        {"image": "123456789.dkr.ecr.us-east-1.amazonaws.com/myapp", "tag": "v1.2.3"},
        {"image": "myregistry.azurecr.io/myapp", "tag": "latest"}
      ]
    context: .
    platforms: linux/amd64,linux/arm64
```

## Inputs

### Mode Selection (use one)

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `tags` | Simple list of image tags (newline-delimited) | No* | - |
| `targets_json` | JSON array of `{image, tag}` objects | No* | - |

*Exactly one of `tags` or `targets_json` must be provided (not both)

### Build Configuration

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `context` | Build context path | No | `.` |
| `dockerfile` | Path to Dockerfile | No | `Dockerfile` |
| `platforms` | Target platforms | No | `linux/amd64` |
| `build_args` | Build arguments (multiline) | No | - |
| `target` | Target build stage | No | - |
| `labels` | Additional labels (multiline) | No | - |
| `use_metadata_labels` | Use docker/metadata-action for labels | No | `true` |
| `metadata_images` | Base image names for metadata-action | No | auto-detected |

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
| `load` | Load into Docker daemon | No | `false` |
| `pull` | Always pull base images | No | `false` |

## Outputs

| Output | Description |
|--------|-------------|
| `imageid` | Image ID |
| `digest` | Image content-addressable digest |
| `metadata` | Build result metadata |
| `tags_list` | Newline-delimited list of image:tag combinations |

## Examples

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

### Using with Custom Metadata Labels

```yaml
- uses: KoalaOps/docker-build-push-action@v1
  with:
    tags: myregistry.io/myapp:latest
    use_metadata_labels: true  # This is the default
    metadata_images: myregistry.io/myapp  # Optional, auto-detected from tags
    labels: |
      maintainer=team@example.com
      custom.label=value
```

### Disabling Metadata Labels

```yaml
- uses: KoalaOps/docker-build-push-action@v1
  with:
    tags: myregistry.io/myapp:latest
    use_metadata_labels: false  # Use built-in label generator
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

### With docker/metadata-action (default)

When `use_metadata_labels: true` (default), the action uses docker/metadata-action to generate rich OCI labels including:
- `org.opencontainers.image.created` - Build timestamp  
- `org.opencontainers.image.revision` - Git SHA
- `org.opencontainers.image.source` - Repository URL
- `org.opencontainers.image.url` - Build run URL
- `org.opencontainers.image.version` - Git ref name
- `org.opencontainers.image.title` - Repository name
- `org.opencontainers.image.description` - Repository description
- Plus any custom labels you provide

### Built-in fallback

When `use_metadata_labels: false`, the action uses a simpler built-in label generator that adds:
- `org.opencontainers.image.created` - Build timestamp
- `org.opencontainers.image.revision` - Git SHA
- `org.opencontainers.image.source` - Repository URL
- `org.opencontainers.image.url` - Build run URL
- `org.opencontainers.image.version` - Git ref name
- `org.opencontainers.image.authors` - GitHub actor

## Notes

- Automatically sets up Docker Buildx and QEMU (for multi-arch builds)
- Registry authentication must be configured separately
- When using `targets_json`, all targets are built with the same image layers and pushed to multiple registries
- Cannot use `push: true` and `load: true` together (validated automatically)
- Label generation defaults to docker/metadata-action for richer metadata