# Observability Agent Helm Charts

Agent-side observability stack for multi-cluster setup. Each agent cluster forwards **metrics → vmagent**, **logs → Fluent Bit**, **traces → OTel Collector** to the central Teamlease observability cluster via internal ALB.

---

## Repository Structure

```
opstree-observability-helm-charts/
  vmagent/
    <CLUSTER_NAME>/
      Chart.yaml
      values.yaml
      charts/
  fluentbit/
    <CLUSTER_NAME>/
      Chart.yaml
      values.yaml
      charts/
  otel/
    <CLUSTER_NAME>/
      otel-operator/
        Chart.yaml
        values.yaml
        charts/
      otel-collector/
        Chart.yaml
        values.yaml
        charts/
        templates/
          instrumentation.yaml
```

---

## Prerequisites

```bash
# 1. Update kubeconfig for target cluster
aws eks update-kubeconfig --name <cluster-name> --region ap-south-1 --profile <profile>

# 2. Create observability namespace
kubectl create namespace observability

# 3. Verify context
kubectl config current-context
```

---

## 1. VMAgent

> Scrapes metrics from all namespaces and remote-writes to central VictoriaMetrics via internal ALB.

### Check before deploying

```bash
# Check if vm-operator already exists
kubectl get pods -A | grep vm-operator

# Check if kube-state-metrics and node-exporter already exist
kubectl get pods -A | grep -iE "kube-state-metrics|node-exporter"
```

> **Note:** If `kube-state-metrics` or `node-exporter` already exist in the cluster, set `kube-state-metrics.enabled: false` and `prometheus-node-exporter.enabled: false` in `values.yaml` to avoid duplicate metrics.

### Option A — vm-operator NOT installed (fresh cluster)

```bash
cd vmagent/<CLUSTER_NAME>

helm dependency update .

# First install with vmagent disabled (installs operator only)
helm install vm . \
  -n observability \
  --set vm.vmagent.enabled=false

# Then enable vmagent
helm upgrade vm . \
  -n observability \
  --set vm.vmagent.enabled=true \
  -f values.yaml
```

### Option B — vm-operator already installed (existing cluster)

Deploy VMAgent CR directly to avoid helm ownership conflicts:

```bash
kubectl apply -f - <<EOF
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMAgent
metadata:
  name: <cluster-name>
  namespace: observability
spec:
  selectAllByDefault: true
  serviceScrapeNamespaceSelector: {}
  scrapeInterval: 30s
  externalLabels:
    cluster: <cluster-name>
    env: dev
  remoteWrite:
    - url: "http://<INTERNAL_ALB>/insert/0/prometheus/api/v1/write"
  extraArgs:
    promscrape.maxScrapeSize: "200MB"
    promscrape.dropOriginalLabels: "true"
  useVMConfigReloader: true
EOF
```

### Verify

```bash
# Check pod is running
kubectl get pods -n observability | grep vmagent

# Check remoteWrite logs (no timeout errors)
kubectl logs -n observability \
  $(kubectl get pod -n observability -l app.kubernetes.io/name=vmagent -o jsonpath='{.items[0].metadata.name}') \
  -c vmagent --tail=20 | grep -iE "remotewrite|2XX|error"
```

---

## 2. Fluent Bit

> Runs as a DaemonSet on every node. Collects container logs from `/var/log/containers/` and forwards to central Loki.

### Check before deploying

```bash
# Check for node taints (may block scheduling)
kubectl describe nodes | grep -i taint

# Check if another log collector exists
kubectl get pods -A | grep -iE "fluentd|fluent-bit|promtail"
```

> **Note for IPv6/dual-stack clusters:** Set `HTTP_Listen ::` in the `[SERVICE]` block of `values.yaml`. Using `0.0.0.0` on IPv6 clusters causes `CrashLoopBackOff` because the kubelet health probe uses the pod's IPv6 address.

> **Note for tainted nodes:** Add matching `tolerations` in `values.yaml` so the DaemonSet schedules on all nodes. Missing tolerations = missing logs from those nodes.

### Deploy

```bash
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update

cd fluentbit/<CLUSTER_NAME>

helm upgrade --install fluent-bit fluent/fluent-bit \
  -n observability \
  -f values.yaml
```

### Verify

```bash
# Check DaemonSet — DESIRED should equal total node count
kubectl get daemonset -n observability | grep fluent-bit

# Check all pods are Running
kubectl get pods -n observability -l app.kubernetes.io/name=fluent-bit -o wide

# Check HTTP server is listening (IPv6: iface=::, IPv4: iface=0.0.0.0)
kubectl logs -n observability \
  $(kubectl get pod -n observability -l app.kubernetes.io/name=fluent-bit -o jsonpath='{.items[0].metadata.name}') \
  | grep "http_server"

# Check no loki errors
kubectl logs -n observability \
  $(kubectl get pod -n observability -l app.kubernetes.io/name=fluent-bit -o jsonpath='{.items[0].metadata.name}') \
  --tail=20 | grep -iE "error|failed|upstream"
```

---

## 3. OpenTelemetry Collector

> Receives OTLP traces from instrumented app pods, enriches with Kubernetes metadata, and exports to central Tempo.

### Step 1 — Check if otel-operator already installed

```bash
kubectl get pods -A | grep otel-operator
```

> **If otel-operator already exists** in the cluster, skip Step 2 and go directly to Step 3. The CRDs are cluster-scoped — installing a second operator will conflict.

### Step 2 — Install otel-operator (skip if already exists)

```bash
cd otel/<CLUSTER_NAME>/otel-operator

helm dependency update .

helm upgrade --install otel-operator . \
  -n observability \
  -f values.yaml
```

### Step 3 — Apply Instrumentation CR

```bash
kubectl apply -f otel/<CLUSTER_NAME>/otel-collector/templates/instrumentation.yaml

# Verify
kubectl get instrumentation -n observability
```

### Step 4 — Deploy otel-collector

```bash
cd otel/<CLUSTER_NAME>/otel-collector

helm dependency update .

helm upgrade --install otel-collector . \
  -n observability \
  -f values.yaml
```

### Step 5 — Add annotation to app deployments

Add the matching annotation to each backend app's pod template:

| Language | Annotation |
|---|---|
| Node.js / NestJS | `instrumentation.opentelemetry.io/inject-nodejs: "observability/otel-instrumentation"` |
| Java | `instrumentation.opentelemetry.io/inject-java: "observability/otel-instrumentation"` |
| Python | `instrumentation.opentelemetry.io/inject-python: "observability/otel-instrumentation"` |
| React (frontend) | ❌ No annotation — runs in browser |

Then restart the deployment to pick up the annotation:

```bash
kubectl rollout restart deployment/<app-name> -n <app-namespace>
```

> **Node.js important:** App Dockerfile must use exec-form CMD:
> ```dockerfile
> # ✅ Correct
> CMD ["node", "dist/main"]
>
> # ❌ Wrong — NODE_OPTIONS won't propagate
> CMD ["sh", "-c", "node dist/main"]
> ```

### Verify

```bash
# Check collector pod is running
kubectl get pods -n observability | grep otel-collector

# Send a test trace
kubectl run trace-test --image=curlimages/curl -it --rm \
  --restart=Never -n observability -- \
  curl -s -X POST \
  http://otel-collector-collector.observability.svc.cluster.local:4318/v1/traces \
  -H "Content-Type: application/json" \
  -d '{"resourceSpans":[{"resource":{"attributes":[{"key":"service.name","value":{"stringValue":"test-service"}}]},"scopeSpans":[{"spans":[{"traceId":"DB8EFFF798038103D269B633813FC60C","spanId":"AEE19B7EC3C1B174","name":"test-span","startTimeUnixNano":"1700000000000000000","endTimeUnixNano":"1700000001000000000","kind":2,"status":{"code":1}}]}]}]}'

# Verify trace reached central Tempo (run on central cluster)
kubectl run curl-test --image=curlimages/curl -it --rm \
  --restart=Never -n observability -- \
  curl -s "http://tempo-query-frontend.observability.svc.cluster.local:3200/api/search?tags=service.name%3Dtest-service&limit=5"

# Check Node.js pod has --require flag (after restart)
kubectl exec -n <namespace> <pod-name> -- cat /proc/1/cmdline | tr '\0' ' '
# Expected: node --require /otel-auto-instrumentation-nodejs/autoinstrumentation.js dist/main
```

---

## Central ALB Endpoint Paths

| Signal | Path | Agent |
|---|---|---|
| Metrics | `/insert/0/prometheus/api/v1/write` | vmagent remoteWrite |
| Logs | `/loki/api/v1/push` | Fluent Bit loki output |
| Traces | `/v1/traces` | OTel Collector otlphttp exporter |

---

## Pushing Changes to CodeCommit

```bash
git clone https://git-codecommit.ap-south-1.amazonaws.com/v1/repos/opstree-observability-helm-charts

cd opstree-observability-helm-charts
git checkout observability-agent

# make changes

git add .
git commit -m "update: <description>"
git push origin observability-agent
```

> **Auth:** Ensure AWS CLI is configured with teamlease profile and git credential helper is set:
> ```bash
> export AWS_PROFILE=teamlease
> git config --global credential.helper '!aws codecommit credential-helper $@'
> git config --global credential.UseHttpPath true
> ```# agent
