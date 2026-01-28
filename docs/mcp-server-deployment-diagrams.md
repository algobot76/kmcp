# MCP Server Deployment Diagrams

This document contains architectural diagrams for the MCP Server deployment process and its relationships with Kubernetes resources.

## Diagram 1: MCP Server Deployment Process

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                           MCP SERVER DEPLOYMENT PROCESS                                  │
└─────────────────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────────────────┐
│  PHASE 1: CLI (User-initiated)                                                           │
└──────────────────────────────────────────────────────────────────────────────────────────┘

  ┌─────────────┐      ┌─────────────────┐      ┌──────────────────┐      ┌─────────────┐
  │   User      │      │   kmcp.yaml     │      │   kmcp CLI       │      │  MCPServer  │
  │   runs      │─────▶│   (project      │─────▶│   deploy         │─────▶│  YAML       │
  │   kmcp      │      │    manifest)    │      │   command        │      │  manifest   │
  │   deploy    │      │                 │      │                  │      │             │
  └─────────────┘      └─────────────────┘      └──────────────────┘      └──────┬──────┘
                                                                                  │
        ┌─────────────────────────────────────────────────────────────────────────┘
        │
        ▼
  ┌───────────────────────────────────────────────────────────────────────────────────────┐
  │  Two Deployment Modes:                                                                 │
  │                                                                                        │
  │  A) Project-based: kmcp deploy                                                         │
  │     - Reads kmcp.yaml for image, framework, secrets                                    │
  │     - Generates MCPServer CR with full config                                          │
  │                                                                                        │
  │  B) Package-based: kmcp deploy package --manager npx --args @mcp/server-github         │
  │     - No kmcp.yaml needed                                                              │
  │     - Auto-injects default image (node:24-alpine / uv:debian)                          │
  └───────────────────────────────────────────────────────────────────────────────────────┘
        │
        │  kubectl apply -f
        ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│  PHASE 2: Kubernetes API Server                                                          │
└──────────────────────────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────────────────────┐
  │                         MCPServer Custom Resource                                     │
  │  ┌────────────────────────────────────────────────────────────────────────────────┐  │
  │  │  apiVersion: kagent.dev/v1alpha1                                               │  │
  │  │  kind: MCPServer                                                               │  │
  │  │  metadata:                                                                     │  │
  │  │    name: my-mcp-server                                                         │  │
  │  │  spec:                                                                         │  │
  │  │    transportType: stdio | http                                                 │  │
  │  │    deployment:                                                                 │  │
  │  │      image: my-image:tag                                                       │  │
  │  │      cmd: python                                                               │  │
  │  │      args: [src/main.py]                                                       │  │
  │  │      port: 3000                                                                │  │
  │  │      env: {...}                                                                │  │
  │  │      secretRefs: [...]                                                         │  │
  │  └────────────────────────────────────────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────────────────────────────────────────┘
        │
        │  Watch event triggers
        ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│  PHASE 3: Controller Reconciliation Loop                                                 │
└──────────────────────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────┐      ┌─────────────────────┐      ┌─────────────────────┐
  │  MCPServerReconciler│      │ TransportAdapter    │      │   K8s Resources     │
  │                     │─────▶│ Translator          │─────▶│   Created/Updated   │
  │  - Fetch MCPServer  │      │                     │      │                     │
  │  - Validate config  │      │ - translateService  │      │ - ServiceAccount    │
  │  - Call translator  │      │ - translateDeploymt │      │ - Deployment        │
  │  - Upsert outputs   │      │ - translateConfigMap│      │ - Service           │
  │  - Update status    │      │ - translateSvcAcct  │      │ - ConfigMap         │
  └─────────────────────┘      └─────────────────────┘      └─────────────────────┘
        │
        │  Reconcile loop monitors
        ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│  PHASE 4: Status Updates                                                                 │
└──────────────────────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────────────────┐
  │  Condition Flow:                                                                     │
  │                                                                                      │
  │  Accepted ──────▶ ResolvedRefs ──────▶ Programmed ──────▶ Ready                     │
  │     │                  │                    │                 │                      │
  │     ▼                  ▼                    ▼                 ▼                      │
  │  Config valid?    Image exists?      Resources created?   Pods running?             │
  └─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Diagram 2: MCPServer Relationships with K8s Resources

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                    MCPServer RELATIONSHIPS WITH K8S RESOURCES                            │
└─────────────────────────────────────────────────────────────────────────────────────────┘


                              ┌─────────────────────────────┐
                              │      MCPServer CR           │
                              │   (kagent.dev/v1alpha1)     │
                              │                             │
                              │  Owner of all resources     │
                              │  (OwnerReference set)       │
                              └──────────────┬──────────────┘
                                             │
                    ┌────────────────────────┼────────────────────────┐
                    │                        │                        │
         ┌──────────┴──────────┐  ┌─────────┴─────────┐  ┌───────────┴───────────┐
         ▼                     ▼  ▼                   ▼  ▼                       ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  ServiceAccount │  │   Deployment    │  │    Service      │  │   ConfigMap     │
│                 │  │                 │  │                 │  │                 │
│ name: {server}  │  │ name: {server}  │  │ name: {server}  │  │ name: {server}  │
│                 │  │                 │  │                 │  │                 │
│ Annotations:    │  │                 │  │ port: {port}    │  │ data:           │
│ - IRSA roles    │  │                 │  │ protocol: TCP   │  │   local.yaml:   │
│ - cloud IAM     │  │                 │  │ appProtocol:    │  │   (transport    │
│                 │  │                 │  │  kgateway.dev/  │  │    adapter      │
└─────────────────┘  │                 │  │  mcp            │  │    config)      │
         │           │                 │  │                 │  │                 │
         │           │                 │  │ selector:       │  └─────────────────┘
         │           │                 │  │  app...name:    │           │
         │           │                 │  │   {server}      │           │
         │           │                 │  └────────┬────────┘           │
         │           │                 │           │                    │
         │           └────────┬────────┘           │                    │
         │                    │                    │                    │
         └────────────────────┼────────────────────┼────────────────────┘
                              │                    │
                              ▼                    │
              ┌───────────────────────────────┐    │
              │         Pod Template          │    │
              │                               │◀───┘
              │  serviceAccountName: {server} │
              │                               │
              │  labels:                      │
              │    app.kubernetes.io/name:    │
              │      {server}                 │
              │    app.kubernetes.io/instance:│
              │      {server}                 │
              │                               │
              └───────────────┬───────────────┘
                              │
        ┌─────────────────────┴─────────────────────┐
        │                                           │
        ▼                                           ▼
┌───────────────────────────────────┐   ┌───────────────────────────────────┐
│     STDIO Transport Pod           │   │     HTTP Transport Pod            │
├───────────────────────────────────┤   ├───────────────────────────────────┤
│                                   │   │                                   │
│  initContainers:                  │   │  containers:                      │
│  ┌─────────────────────────────┐  │   │  ┌─────────────────────────────┐  │
│  │ copy-binary                 │  │   │  │ mcp-server                  │  │
│  │ image: agentgateway:0.9.0   │  │   │  │ image: {user-image}         │  │
│  │ args: --copy-self           │  │   │  │ command: {cmd}              │  │
│  │       /adapterbin/agentgw   │  │   │  │ args: {args}                │  │
│  └─────────────────────────────┘  │   │  │ ports: [{port}]             │  │
│                                   │   │  └─────────────────────────────┘  │
│  containers:                      │   │                                   │
│  ┌─────────────────────────────┐  │   │  ┌─────────────────────────────┐  │
│  │ mcp-server                  │  │   │  │ sidecars (optional)         │  │
│  │ image: {user-image}         │  │   │  └─────────────────────────────┘  │
│  │ command:                    │  │   │                                   │
│  │   /adapterbin/agentgateway  │  │   │  volumes:                         │
│  │ args: -f /config/local.yaml │  │   │  ┌─────────────────────────────┐  │
│  └─────────────────────────────┘  │   │  │ config (ConfigMap)          │  │
│                                   │   │  │ + custom volumes            │  │
│  ┌─────────────────────────────┐  │   │  └─────────────────────────────┘  │
│  │ sidecars (optional)         │  │   │                                   │
│  └─────────────────────────────┘  │   └───────────────────────────────────┘
│                                   │
│  volumes:                         │
│  ┌─────────────────────────────┐  │
│  │ config (ConfigMap)          │  │
│  │ binary (EmptyDir)           │  │
│  │ + custom volumes            │  │
│  └─────────────────────────────┘  │
│                                   │
└───────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────────────────┐
│  EXTERNAL REFERENCES (not owned, but referenced)                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘

  MCPServer.spec.deployment
         │
         ├── secretRefs ──────────────────▶ ┌─────────────────┐
         │                                  │  Secret         │  (injected as envFrom)
         │                                  │  (pre-existing) │
         │                                  └─────────────────┘
         │
         ├── configMapRefs ───────────────▶ ┌─────────────────┐
         │                                  │  ConfigMap      │  (mounted as volume)
         │                                  │  (pre-existing) │
         │                                  └─────────────────┘
         │
         └── imagePullSecrets ────────────▶ ┌─────────────────┐
                                            │  Secret         │  (for private registries)
                                            │  (pre-existing) │
                                            └─────────────────┘


┌─────────────────────────────────────────────────────────────────────────────────────────┐
│  RESOURCE NAMING CONVENTION                                                              │
└─────────────────────────────────────────────────────────────────────────────────────────┘

  MCPServer: "my-mcp-server"
       │
       ├──▶ ServiceAccount:  "my-mcp-server"
       ├──▶ Deployment:      "my-mcp-server"
       ├──▶ Service:         "my-mcp-server"
       └──▶ ConfigMap:       "my-mcp-server"

  All resources share the same name as the MCPServer CR for easy correlation.
```

---

## Summary Table

| Resource | Created By | Owned By MCPServer | Purpose |
|----------|------------|-------------------|---------|
| ServiceAccount | Controller | Yes | Pod identity, cloud IAM integration |
| Deployment | Controller | Yes | Runs MCP server container(s) |
| Service | Controller | Yes | Network endpoint (port exposure) |
| ConfigMap | Controller | Yes | Transport adapter config (`local.yaml`) |
| Secret (refs) | User | No (referenced) | Credentials injected as env vars |
| ConfigMap (refs) | User | No (referenced) | User config mounted as volumes |

---

## Key Files

| File | Role |
|------|------|
| `api/v1alpha1/mcpserver_types.go` | CRD type definitions |
| `pkg/controller/mcpserver_controller.go` | Reconciliation loop |
| `pkg/controller/transportadapter/transportadapter_translator.go` | CR to K8s resource translation |
| `pkg/cli/internal/commands/deploy.go` | CLI deploy command |
