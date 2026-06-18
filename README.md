# Observability Agent Helm Charts

Agent-side observability stack for multi-cluster setup. Each agent cluster forwards **metrics → vmagent**, **logs → Fluent Bit**, **traces → OTel Collector** to the central Teamlease observability cluster via internal ALB.

---

## 1. VMAgent

> Scrapes metrics from all namespaces and remote-writes to central VictoriaMetrics via internal ALB.

# First install with vmagent disabled (installs operator only)
helm install <release_name>. \
  -n observability \
  --set vm.vmagent.enabled=false

# Then enable vmagent
helm upgrade <release_name> . \
  -n observability \
  --set vm.vmagent.enabled=true \
  -f values.yaml
```

### Verify

```bash
# Check pod is running
kubectl get pods -n observability | grep vmagent
---

## 2. Fluent Bit

> Runs as a DaemonSet on every node. Collects container logs from `/var/log/containers/` and forwards to central Loki.

### Deploy

```bash
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update

helm install <release_name> fluent/fluent-bit \
  -n observability \
  -f values.yaml
```

### Verify

```bash
# Check DaemonSet — DESIRED should equal total node count
kubectl get daemonset -n observability | grep fluent-bit

# Check all pods are Running
kubectl get pods -n observability -l app.kubernetes.io/name=fluent-bit -o wide

---

## 3. OpenTelemetry Collector

> Receives OTLP traces from instrumented app pods, enriches with Kubernetes metadata, and exports to central Tempo.

### Step 2 — Install otel-operator 

```bash
helm dependency update .

helm install <release_name> . \
  -n observability \
  -f values.yaml
```

### Step 3 — Apply Instrumentation CR

```bash
kubectl apply -f otel-collector/templates/instrumentation.yaml

# Verify
kubectl get instrumentation -n observability
```

### Step 4 — Deploy otel-collector

```bash
cd otel-collector

helm dependency update .

helm install <release_name> . \
  -n observability \
  -f values.yaml
```

---
