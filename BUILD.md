# Building & Publishing the Docker Image

## Prerequisites

- Docker with buildx (e.g. OrbStack, Docker Desktop)
- A multiarch builder: `docker buildx create --name multiarch --use`
- Logged in to Docker Hub: `docker login`

## Build & Push

```bash
# 1. Build the TypeScript source
npm run build

# 2. Build and push multiarch image (linux/amd64 + linux/arm64)
#    Update version tags to match package.json version
docker buildx build \
  --builder multiarch \
  --platform linux/amd64,linux/arm64 \
  --tag alexkreidler/google-drive-mcp:1.7.4 \
  --tag alexkreidler/google-drive-mcp:1.7 \
  --tag alexkreidler/google-drive-mcp:1 \
  --tag alexkreidler/google-drive-mcp:latest \
  --push \
  .
```

## Tagging Convention

| Tag | Meaning |
|-----|---------|
| `1.7.4` | Exact patch version — immutable, pinned in production |
| `1.7` | Minor version — floats to latest patch (e.g. 1.7.5) |
| `1` | Major version — floats to latest minor |
| `latest` | Always the most recent build |

## Version Bump Workflow

1. Update `version` in `package.json`
2. `npm run build`
3. Run the `docker buildx build` command above with updated tags