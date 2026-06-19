# Observability Agent Helm Charts

Agent-side observability stack for multi-cluster setup. Each agent cluster forwards **metrics → vmagent**, **logs → Fluent Bit**, **traces → OTel Collector** to the central observability cluster via internal ALB.

## Repository Structure

```
observability/
├── Non-prod/
│   ├── fluentbit/
│   │   ├── ALCSNG/
│   │   │   ├── Chart.yaml  Chart.lock  values.yaml
│   │   │   └── charts/fluent-bit-0.48.10.tgz
│   │   ├── HCMNG/
│   │   │   ├── Chart.yaml  Chart.lock  values.yaml
│   │   │   └── charts/
│   │   └── Hiretech/
│   │       ├── Chart.yaml  values.yaml
│   │       └── charts/
│   ├── otel/
│   │   ├── ALCSNG/
│   │   │   ├── otel-operator/
│   │   │   │   ├── Chart.yaml  Chart.lock  values.yaml
│   │   │   │   └── charts/opentelemetry-operator-0.105.0.tgz
│   │   │   └── otel-collector/
│   │   │       ├── Chart.yaml  Chart.lock  values.yaml
│   │   │       ├── charts/opentelemetry-kube-stack-0.12.4.tgz
│   │   │       └── templates/instrumentation.yaml
│   │   ├── HCMNG/
│   │   │   ├── otel-operator/
│   │   │   │   ├── Chart.yaml  Chart.lock  values.yaml
│   │   │   │   └── charts/opentelemetry-operator-0.105.0.tgz
│   │   │   └── otel-collector/
│   │   │       ├── Chart.yaml  Chart.lock  values.yaml
│   │   │       ├── charts/opentelemetry-kube-stack-0.12.4.tgz
│   │   │       └── templates/instrumentation.yaml
│   │   └── Hiretech/
│   │       ├── otel-operator/
│   │       │   ├── Chart.yaml  values.yaml
│   │       └── otel-collector/
│   │           ├── Chart.yaml  values.yaml
│   │           └── templates/instrumentation.yaml
│   └── vmagent/
│       ├── ALSCNG/
│       │   ├── Chart.yaml  Chart.lock  values.yaml
│       │   └── charts/victoria-metrics-k8s-stack-0.70.0.tgz
│       ├── HCMNG/
│       │   ├── Chart.yaml  Chart.lock  values.yaml
│       │   └── charts/victoria-metrics-k8s-stack-0.70.0.tgz
│       └── Hiretech/
│           ├── Chart.yaml  values.yaml
│           └── charts/
└── Prod/
    ├── fluentbit/
    │   ├── ALCSNG/
    │   │   ├── Chart.yaml  Chart.lock  values.yaml
    │   │   └── charts/fluent-bit-0.48.10.tgz
    │   ├── HCMNG/
    │   │   ├── Chart.yaml  Chart.lock  values.yaml
    │   │   └── charts/
    │   └── Hiretech/
    │       ├── Chart.yaml  values.yaml
    │       └── charts/
    ├── otel/
    │   ├── ALCSNG/
    │   │   ├── otel-operator/
    │   │   │   ├── Chart.yaml  Chart.lock  values.yaml
    │   │   │   └── charts/opentelemetry-operator-0.105.0.tgz
    │   │   └── otel-collector/
    │   │       ├── Chart.yaml  Chart.lock  values.yaml
    │   │       ├── charts/opentelemetry-kube-stack-0.12.4.tgz
    │   │       └── templates/instrumentation.yaml
    │   ├── HCMNG/
    │   │   ├── otel-operator/
    │   │   │   ├── Chart.yaml  Chart.lock  values.yaml
    │   │   │   └── charts/opentelemetry-operator-0.105.0.tgz
    │   │   └── otel-collector/
    │   │       ├── Chart.yaml  Chart.lock  values.yaml
    │   │       ├── charts/opentelemetry-kube-stack-0.12.4.tgz
    │   │       └── templates/instrumentation.yaml
    │   └── Hiretech/
    │       ├── otel-operator/
    │       │   ├── Chart.yaml  values.yaml
    │       └── otel-collector/
    │           ├── Chart.yaml  values.yaml
    │           └── templates/instrumentation.yaml
    └── vmagent/
        ├── ALSCNG/
        │   ├── Chart.yaml  Chart.lock  values.yaml
        │   └── charts/victoria-metrics-k8s-stack-0.70.0.tgz
        ├── HCMNG/
        │   ├── Chart.yaml  Chart.lock  values.yaml
        │   └── charts/victoria-metrics-k8s-stack-0.70.0.tgz
        └── Hiretech/
            ├── Chart.yaml  values.yaml
            └── charts/
```

## Cluster Reference

| Cluster | Environment | vmagent | Fluent Bit | OTel |
|---|---|---|---|---|
| HCMNG | Non-prod + Prod | `Non-prod/vmagent/HCMNG` | `Non-prod/fluentbit/HCMNG` | `Non-prod/otel/HCMNG` |
| ALCSNG | Non-prod + Prod | `Non-prod/vmagent/ALSCNG` | `Non-prod/fluentbit/ALCSNG` | `Non-prod/otel/ALCSNG` |
| Hiretech | Non-prod + Prod | `Non-prod/vmagent/Hiretech` | `Non-prod/fluentbit/Hiretech` | `Non-prod/otel/Hiretech` |


## 1. VMAgent

Scrapes metrics from all namespaces and remote-writes to central VictoriaMetrics via internal ALB.


### Installation

```bash
cd <Non-prod|Prod>/vmagent/<CLUSTER_NAME>

# First install with vmagent disabled (installs operator only)
helm install <release_name> . \
  -n observability \
  --set vm.vmagent.enabled=false

# Then enable vmagent
helm upgrade <release_name> . \
  -n observability \
  --set vm.vmagent.enabled=true \
  -f values.yaml
```

### Upgrade

```bash
cd <Non-prod|Prod>/vmagent/<CLUSTER_NAME>

helm upgrade <release_name> . \
  -n observability \
  -f values.yaml
```

### Verify

```bash
# Check pod is running
kubectl get pods -n observability | grep vmagent

```

## 2. Fluent Bit

Runs as a DaemonSet on every node. Collects container logs from `/var/log/containers/` and forwards to central Loki.



###  Installation

```bash
cd <Non-prod|Prod>/fluentbit/<CLUSTER_NAME>

helm install <release_name> . \
  -n observability \
  -f values.yaml
```

### Upgrade

```bash
cd <Non-prod|Prod>/fluentbit/<CLUSTER_NAME>

helm upgrade <release_name> . \
  -n observability \
  -f values.yaml
```

### Verify

```bash
# Check DaemonSet — DESIRED should equal total node count
kubectl get daemonset -n observability | grep fluent-bit

# Check all pods are Running
kubectl get pods -n observability -l app.kubernetes.io/name=fluent-bit -o wide

```

## 3. OpenTelemetry Collector

Receives OTLP traces from instrumented app pods, enriches with Kubernetes metadata, and exports to central Tempo.

### Step 1 — Check if otel-operator already installed

```bash
kubectl get pods -A | grep otel-operator
```

> **If otel-operator already exists**, skip Step 2 and go directly to Step 3. Installing a second operator will conflict with existing cluster-scoped CRDs.

### Step 2 — Install otel-operator (skip if already exists)

```bash
cd <Non-prod|Prod>/otel/<CLUSTER_NAME>/otel-operator

helm install <release_name> . \
  -n observability \
  -f values.yaml
```

#### Upgrade otel-operator

```bash
cd <Non-prod|Prod>/otel/<CLUSTER_NAME>/otel-operator

helm upgrade <release_name> . \
  -n observability \
  -f values.yaml
```

### Step 3 — Apply Instrumentation CR

```bash
kubectl apply -f <Non-prod|Prod>/otel/<CLUSTER_NAME>/otel-collector/templates/instrumentation.yaml

# Verify
kubectl get instrumentation -n observability
```

### Step 4 — Deploy otel-collector

```bash
cd <Non-prod|Prod>/otel/<CLUSTER_NAME>/otel-collector

helm install <release_name> . \
  -n observability \
  -f values.yaml
```

#### Upgrade otel-collector

```bash
cd <Non-prod|Prod>/otel/<CLUSTER_NAME>/otel-collector

helm upgrade <release_name> . \
  -n observability \
  -f values.yaml
```

### Step 5 — Add annotation to app deployments

| Language | Annotation |
|---|---|
| Node.js / NestJS | `instrumentation.opentelemetry.io/inject-nodejs: "observability/otel-instrumentation"` |
| Java | `instrumentation.opentelemetry.io/inject-java: "observability/otel-instrumentation"` |
| Python | `instrumentation.opentelemetry.io/inject-python: "observability/otel-instrumentation"` |
| React (frontend) | No annotation — runs in browser |

```bash
# Restart deployment to pick up annotation
kubectl rollout restart deployment/<app-name> -n <app-namespace>
```



## Central ALB Endpoint Paths

| Signal | Path | Agent |
|---|---|---|
| Metrics | `/insert/0/prometheus/api/v1/write` | vmagent remoteWrite |
| Logs | `/loki/api/v1/push` | Fluent Bit loki output |
| Traces | `/v1/traces` | OTel Collector otlphttp exporter |


