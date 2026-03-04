# Summit 2026 Guardrails Lab - Deployment

A Helm chart that deploys a multi-tenant lab environment for the Red Hat Summit 2026 Guardrails workshop. Each attendee gets their own namespace with a PeachPal frontend and guardrails infrastructure.

## Architecture

```
Per-User Namespace (user{N}-peachpal)          Shared Namespace (gaas)
┌─────────────────────────────────────┐       ┌──────────────────────────┐
│  PeachPal (FastAPI)                 │       │  HAP Detector (KServe)   │
│  Guardrails Orchestrator            │──────>│  Prompt Injection Detector│
│  System Prompt ConfigMap            │       │  Language Detector       │
│  App Config ConfigMap               │       │  Chunker Service         │
│  ServiceMonitor                     │       │  MinIO (Model Storage)   │
└─────────────────────────────────────┘       └──────────────────────────┘
```

## Chart Structure

```
deploy-lab/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── user-namespaces/          # Per-attendee resources
    │   ├── namespace.yaml
    │   ├── peachpal-deployment.yaml
    │   ├── guardrails-orchestrator.yaml
    │   ├── fms-orchestr8-config-nlp.yaml
    │   ├── fms-orchestr8-config-gateway.yaml
    │   └── vllm-apikey.yaml
    ├── gaas/                     # Shared guardrails services
    │   ├── namespace.yaml
    │   ├── chunker-service.yaml
    │   ├── lingua-deployment.yaml
    │   ├── hap-detector.yaml
    │   ├── minio-detector-models.yaml
    │   └── prompt-injection.yaml
    └── uwm/                      # Monitoring
        └── userworkloadmonitoring.yaml
```

## Configuration

### values.yaml

| Value | Default | Description |
|-------|---------|-------------|
| `attendees` | `10` | Number of user namespaces to create |

### PeachPal ConfigMap (`peachpal-config`)

Environment variables injected into each PeachPal deployment:

| Key | Default | Description |
|-----|---------|-------------|
| `LOG_LEVEL` | `DEBUG` | Application log level |
| `GUARDRAILS_ENABLED` | `false` | Enable guardrails detectors |
| `GUARDRAILS_ORCHESTRATOR_SERVICE_SERVICE_HOST` | `guardrails-orchestrator-gateway` | Orchestrator hostname |
| `GUARDRAILS_ORCHESTRATOR_SERVICE_SERVICE_PORT_GATEWAY` | `8090` | Orchestrator port |
| `VLLM_MODEL` | `llama32-fp8` | LLM model name |
| `MAX_TOKENS` | `200` | Max response tokens |
| `TEMPERATURE` | `0` | LLM temperature |

## Deploying

### Prerequisites

- OpenShift cluster with OpenDataHub operator installed
- KServe configured for model serving
- Helm 3.x

### Install

```bash
# Deploy for 10 attendees (default)
helm install guardrails-lab .

# Deploy for a custom number of attendees
helm install guardrails-lab . --set attendees=25
```

### Update

```bash
helm upgrade guardrails-lab .
```

### Uninstall

```bash
helm uninstall guardrails-lab
```

## What Gets Created

For each attendee (e.g., attendee 1):

- **Namespace**: `user1-peachpal`
- **Deployment**: PeachPal FastAPI app
- **ConfigMaps**: `peachpal-config` (env vars), `peachpal-system-prompt` (LLM system prompt)
- **Secret**: `peachpal-secrets` (vLLM API key)
- **Service + Route**: Exposes PeachPal with TLS edge termination
- **ServiceMonitor**: Prometheus metrics at `/metrics` (3s interval)
- **GuardrailsOrchestrator**: TrustyAI orchestrator CR

Shared (in `gaas` namespace):

- **HAP Detector**: IBM Granite Guardian model via KServe
- **Prompt Injection Detector**: DeBERTa-based model via KServe
- **Language Detector**: Lingua service
- **Chunker**: Text chunking service
- **MinIO**: Model artifact storage (50Gi)
