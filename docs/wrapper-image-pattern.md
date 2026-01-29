# Wrapper Image Pattern

An alternative approach to deploying stdio MCP servers by combining the transport adapter with the MCP server into a single pre-integrated container image.

## Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         BUILD TIME                              │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────┐    ┌─────────────────────┐
│ User's MCP Server   │    │ Transport Adapter   │
│ Image               │    │ Image               │
│                     │    │                     │
│ python:3.11         │    │ agentgateway:0.9.0  │
│ /app/main.py        │    │ /agentgateway       │
└──────────┬──────────┘    └──────────┬──────────┘
           │                          │
           └────────────┬─────────────┘
                        │
                        ▼
           ┌─────────────────────────┐
           │   Wrapper Image         │
           │                         │
           │   /agentgateway         │
           │   /app/main.py          │
           │   /config/local.yaml    │
           │                         │
           │   ENTRYPOINT:           │
           │   agentgateway -f ...   │
           └─────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                         RUN TIME                                │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  Single Container (no init container needed)                    │
│                                                                 │
│  PID 1: agentgateway -f /config/local.yaml                      │
│           │                                                     │
│           │ spawns                                              │
│           ▼                                                     │
│  PID 2: python /app/main.py                                     │
└─────────────────────────────────────────────────────────────────┘
```

## Comparison with Default Approach

| Aspect | kmcp Default (Init Container) | Wrapper Image |
|--------|-------------------------------|---------------|
| Build time | No extra build | Requires building wrapper |
| Image count | 1 (user's original) | 2 (original + wrapper) |
| Runtime overhead | Init container runs first | None |
| User effort | Zero | Must build wrapper image |
| CI/CD integration | Works with any image | Needs build pipeline |
| Version updates | Change env var | Rebuild wrapper |
| Image size | Larger at runtime (2 images pulled) | Smaller (1 combined image) |

## When to Use

| Scenario | Recommended? | Reason |
|----------|--------------|--------|
| One-off deployment | No | Too much effort |
| Production with CI/CD | Yes | Cleaner, smaller images |
| Air-gapped environment | Yes | Single image to transfer |
| Strict security (no init containers) | Yes | Some policies block init containers |
| Many MCP servers | No | Too many wrapper images to maintain |
| Rapid prototyping | No | Slows down iteration |

## Implementation

### Option A: Multi-stage Dockerfile

```dockerfile
# Stage 1: Get the adapter binary
FROM ghcr.io/agentgateway/agentgateway:0.9.0-musl AS adapter

# Stage 2: User's MCP server
FROM python:3.11-slim AS mcp-server
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY src/ ./src/

# Stage 3: Combined wrapper image
FROM python:3.11-slim

COPY --from=adapter /agentgateway /usr/local/bin/agentgateway
COPY --from=mcp-server /app /app
COPY --from=mcp-server /usr/local/lib/python3.11/site-packages /usr/local/lib/python3.11/site-packages

COPY config/local.yaml /config/local.yaml

ENTRYPOINT ["agentgateway", "-f", "/config/local.yaml"]
```

### Option B: Extend Existing Image

```dockerfile
FROM user-mcp-server:latest

COPY --from=ghcr.io/agentgateway/agentgateway:0.9.0-musl /agentgateway /usr/local/bin/

COPY local.yaml /config/local.yaml

ENTRYPOINT ["agentgateway", "-f", "/config/local.yaml"]
```

### Option C: Parameterized Wrapper

```dockerfile
ARG BASE_IMAGE
ARG ADAPTER_VERSION=0.9.0

FROM ghcr.io/agentgateway/agentgateway:${ADAPTER_VERSION}-musl AS adapter
FROM ${BASE_IMAGE}

COPY --from=adapter /agentgateway /usr/local/bin/agentgateway
COPY local.yaml /config/local.yaml

ENTRYPOINT ["agentgateway", "-f", "/config/local.yaml"]
```

Build with:

```bash
docker build \
  --build-arg BASE_IMAGE=my-mcp-server:latest \
  --build-arg ADAPTER_VERSION=0.9.0 \
  -t my-mcp-server-wrapped:latest .
```

## Config File

The wrapper image requires a `local.yaml` config file:

```yaml
binds:
  - port: 3000
    listeners:
      - name: default
        protocol: HTTP
        routes:
          - routeName: mcp
            matches:
              - path:
                  pathPrefix: /sse
              - path:
                  pathPrefix: /mcp
            backends:
              - weight: 100
                mcp:
                  targets:
                    - name: my-mcp-server
                      stdio:
                        cmd: python
                        args: ["/app/src/main.py"]
                        env:
                          LOG_LEVEL: info
```

## CI/CD Integration

### GitHub Actions Example

```yaml
name: Build MCP Wrapper

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build wrapper image
        run: |
          docker build -t my-mcp-server-wrapped:${{ github.sha }} \
            --build-arg BASE_IMAGE=my-mcp-server:latest \
            --build-arg ADAPTER_VERSION=0.9.0 \
            -f Dockerfile.wrapper .
      
      - name: Push to registry
        run: |
          docker push my-mcp-server-wrapped:${{ github.sha }}
```

## Kubernetes Deployment

With a wrapper image, the MCPServer CR is simpler (no init container configuration needed):

```yaml
apiVersion: kagent.dev/v1alpha1
kind: MCPServer
metadata:
  name: my-mcp-server
spec:
  transportType: stdio
  deployment:
    image: my-mcp-server-wrapped:latest
    port: 3000
```

The controller will still generate the standard resources, but the init container logic becomes redundant since the adapter is already in the image.

## Pros and Cons

### Pros

- No init container at runtime
- Single image to deploy
- Smaller runtime footprint
- Works in restricted environments that block init containers
- Faster pod startup

### Cons

- Extra build step required
- Must rebuild when adapter updates
- More images to manage in registry
- Breaks "use any image" simplicity
- Coupling between MCP server and adapter versions

## Related Documentation

- [MCP Server Deployment Diagrams](./mcp-server-deployment-diagrams.md)
- [MCPServer CRD Schema](./mcp_crd.yaml)
