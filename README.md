# Observability Agent Helm Charts

Agent-side observability stack for multi-cluster setup. Each agent cluster forwards **metrics → vmagent**, **logs → Fluent Bit**, **traces → OTel Collector** to the central Teamlease observability cluster via internal ALB.

## Repository Structure

```
opstree-observability-helm-charts/
  Non-prod/
    vmagent/
      <CLUSTER_NAME>/
        Chart.yaml  Chart.lock  values.yaml  charts/
    fluentbit/
      <CLUSTER_NAME>/
        Chart.yaml  Chart.lock  values.yaml  charts/
    otel/
      <CLUSTER_NAME>/
        otel-operator/
          Chart.yaml  Chart.lock  values.yaml  charts/
        otel-collector/
          Chart.yaml  Chart.lock  values.yaml  charts/
          templates/
            instrumentation.yaml
```

## Prerequisites

```bash
# Update kubeconfig for target cluster
aws eks update-kubeconfig --name <cluster-name> --region ap-south-1 --profile <profile>

# Create observability namespace
kubectl create namespace observability

# Verify context
kubectl config current-context
```

## 1. VMAgent

Scrapes metrics from all namespaces and remote-writes to central VictoriaMetrics via internal ALB.

### Check before deploying

```bash
# Check if vm-operator already exists
kubectl get pods -A | grep vm-operator

# Check if kube-state-metrics and node-exporter already exist
kubectl get pods -A | grep -iE "kube-state-metrics|node-exporter"
```

> **Note:** If `kube-state-metrics` or `node-exporter` already exist, set `kube-state-metrics.enabled: false` and `prometheus-node-exporter.enabled: false` in `values.yaml` to avoid duplicate metrics.

### Fresh Install

```bash
cd Non-prod/vmagent/<CLUSTER_NAME>

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
cd Non-prod/vmagent/<CLUSTER_NAME>

helm upgrade <release_name> . \
  -n observability \
  -f values.yaml
```

### Verify

```bash
# Check pod is running
kubectl get pods -n observability | grep vmagent

# Check remoteWrite logs
kubectl logs -n observability \
  $(kubectl get pod -n observability -l app.kubernetes.io/name=vmagent \
  -o jsonpath='{.items[0].metadata.name}') \
  -c vmagent --tail=20 | grep -iE "remotewrite|2XX|error"
```

## 2. Fluent Bit

Runs as a DaemonSet on every node. Collects container logs from `/var/log/containers/` and forwards to central Loki.

### Check before deploying

```bash
# Check for node taints
kubectl describe nodes | grep -i taint

# Check if another log collector exists
kubectl get pods -A | grep -iE "fluentd|fluent-bit|promtail"
```

> **Note for IPv6/dual-stack clusters:** Set `HTTP_Listen ::` in the `[SERVICE]` block of `values.yaml`. Using `0.0.0.0` causes `CrashLoopBackOff` on IPv6 clusters.

> **Note for tainted nodes:** Add matching `tolerations` in `values.yaml` so DaemonSet schedules on all nodes.

### Fresh Install

```bash
cd Non-prod/fluentbit/<CLUSTER_NAME>

helm install <release_name> . \
  -n observability \
  -f values.yaml
```

### Upgrade

```bash
cd Non-prod/fluentbit/<CLUSTER_NAME>

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

# Check HTTP server listening (IPv6: iface=::, IPv4: iface=0.0.0.0)
kubectl logs -n observability \
  $(kubectl get pod -n observability -l app.kubernetes.io/name=fluent-bit \
  -o jsonpath='{.items[0].metadata.name}') | grep "http_server"

# Check no loki errors
kubectl logs -n observability \
  $(kubectl get pod -n observability -l app.kubernetes.io/name=fluent-bit \
  -o jsonpath='{.items[0].metadata.name}') \
  --tail=20 | grep -iE "error|failed|upstream"
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
cd Non-prod/otel/<CLUSTER_NAME>/otel-operator

helm install <release_name> . \
  -n observability \
  -f values.yaml
```

#### Upgrade otel-operator

```bash
cd Non-prod/otel/<CLUSTER_NAME>/otel-operator

helm upgrade <release_name> . \
  -n observability \
  -f values.yaml
```

### Step 3 — Apply Instrumentation CR

```bash
kubectl apply -f Non-prod/otel/<CLUSTER_NAME>/otel-collector/templates/instrumentation.yaml

# Verify
kubectl get instrumentation -n observability
```

### Step 4 — Deploy otel-collector

```bash
cd Non-prod/otel/<CLUSTER_NAME>/otel-collector

helm install <release_name> . \
  -n observability \
  -f values.yaml
```

#### Upgrade otel-collector

```bash
cd Non-prod/otel/<CLUSTER_NAME>/otel-collector

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

> **Node.js:** Dockerfile must use exec-form CMD:
> ```dockerfile
> # Correct
> CMD ["node", "dist/main"]
> # Wrong — NODE_OPTIONS won't propagate
> CMD ["sh", "-c", "node dist/main"]
> ```

### Verify

```bash
# Check collector pod is running
kubectl get pods -n observability | grep otel-collector

# Check health
kubectl port-forward -n observability \
  svc/<release_name>-collector 13133:13133
curl http://localhost:13133/

# Check collector logs
kubectl logs -n observability \
  $(kubectl get pod -n observability -l app.kubernetes.io/name=otel-collector \
  -o jsonpath='{.items[0].metadata.name}') \
  --tail=20 | grep -iE "error|failed"
```

## Central ALB Endpoint Paths

| Signal | Path | Agent |
|---|---|---|
| Metrics | `/insert/0/prometheus/api/v1/write` | vmagent remoteWrite |
| Logs | `/loki/api/v1/push` | Fluent Bit loki output |
| Traces | `/v1/traces` | OTel Collector otlphttp exporter |

## Helm Release Reference

| Component | Typical Release Name | Namespace | Chart Location |
|---|---|---|---|
| VMAgent | `vm-<cluster>` | observability | `Non-prod/vmagent/<CLUSTER>` |
| Fluent Bit | `fluent-bit-<cluster>` | observability | `Non-prod/fluentbit/<CLUSTER>` |
| OTel Operator | `otel-operator-<cluster>` | observability | `Non-prod/otel/<CLUSTER>/otel-operator` |
| OTel Collector | `otel-collector-<cluster>` | observability | `Non-prod/otel/<CLUSTER>/otel-collector` |

## Pushing Changes

```bash
# Push to CodeCommit
git add .
git commit -m "update: <description>"
git push origin observability-agent

# Push to GitHub mirror
git push github observability-agent
```
