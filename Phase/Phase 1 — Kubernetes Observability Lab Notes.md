# Phase 1 — Kubernetes Observability Lab Notes  
**Environment:** KodeKloud / K3s (3 nodes) · **Stack:** kube-prometheus-stack (`kps`) · **Namespace:** `monitoring` (+ `demo` for sample app)

Use this as your full copy-paste notes. Insert your screenshots where marked **`[SS: ...]`**.

---

## 1) Goal of Phase 1

Learn production-style monitoring on Kubernetes **without writing an app from scratch**:

1. Install Prometheus + Grafana + Alertmanager via Helm  
2. Understand cluster metrics (kube-state-metrics, kubelet, etc.)  
3. Create custom alerts with `PrometheusRule`  
4. Scrape a sample app with `ServiceMonitor`  
5. Query metrics with PromQL and verify in UI  

**Phase 2 later:** Docker Compose path (local app expose `/metrics`).

---

## 2) Cluster baseline (what we started with)

```bash
kubectl cluster-info
kubectl get nodes -o wide
kubectl get ns
helm version
```

**What we had:**
- K3s control plane + 2 workers  
- Helm v3 installed  
- Fresh cluster (no monitoring yet)  
- `metrics-server` already present  

**K3s note (interview useful):**  
No separate `kubelet.service`. Kubelet is inside `k3s` / `k3s-agent`. `systemctl` may not exist on Alpine lab nodes.

`[SS: kubectl get nodes]`

---

## 3) Install kube-prometheus-stack

```bash
kubectl create namespace monitoring

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install kps prometheus-community/kube-prometheus-stack \
  -n monitoring \
  --set grafana.adminPassword=admin123 \
  --set prometheus.prometheusSpec.retention=7d
```

### What gets installed
| Component | Role |
|---|---|
| Prometheus | Scrapes + stores metrics, evaluates alerts |
| Grafana | Dashboards / visualization |
| Alertmanager | Receives firing alerts, routes notifications |
| prometheus-operator | Watches CRs (`ServiceMonitor`, `PrometheusRule`, etc.) |
| kube-state-metrics | K8s object metrics (pods, deploys, etc.) |
| node-exporter | Host/node metrics (CPU/disk/mem) |

```bash
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

`[SS: kubectl get pods -n monitoring after install]`

---

## 4) node-exporter issue on K3s (and fix)

### Symptom
```text
kps-prometheus-node-exporter-*  CreateContainerError
path "/" is mounted on "/" but it is not a shared or slave mount
```

### Why
Common on nested/K3s lab clusters. Host root mount propagation breaks node-exporter.

### Is node-exporter needed for learning?
**Useful in production/interviews, not required for first learning loop.**

- **node-exporter** → host metrics  
- **kube-state-metrics** → K8s object state  
- **kubelet/cAdvisor** → container metrics  

### Fix used
```bash
helm upgrade kps prometheus-community/kube-prometheus-stack \
  -n monitoring \
  --reuse-values \
  --set prometheus-node-exporter.enabled=false \
  --set nodeExporter.enabled=false
```

Why `--reuse-values`? Keeps previous settings (password, retention, etc.).

Optional cleanup leftovers:
```bash
kubectl delete daemonset -n monitoring -l app.kubernetes.io/name=prometheus-node-exporter --ignore-not-found
```

`[SS: pods after disabling node-exporter]`

---

## 5) Accessing UIs on KodeKloud

### Important lab lesson
- **NodePort alone is often not enough** for KodeKloud `https://<port>-port-....labs.kodekloud.com/`
- That URL proxies to **student-node localhost**
- So use **port-forward with `--address 0.0.0.0`**

### Grafana
```bash
kubectl port-forward -n monitoring svc/kps-grafana 31346:80 --address 0.0.0.0
```
Open: `https://31346-port-<lab>.labs.kodekloud.com/`  
Login: `admin` / `admin123`

### Prometheus
```bash
kubectl port-forward -n monitoring svc/kps-kube-prometheus-stack-prometheus 9090:9090 --address 0.0.0.0
```
Open: `https://9090-port-<lab>.labs.kodekloud.com/`

### Alertmanager
```bash
kubectl port-forward -n monitoring svc/kps-kube-prometheus-stack-alertmanager 9093:9093 --address 0.0.0.0
```

Keep port-forward terminals running.

`[SS: Grafana login]`  
`[SS: Prometheus home / query page]`

---

## 6) Concepts: Monitoring vs Observability (quick)

- **Monitoring:** is it healthy? (known metrics/thresholds)  
- **Observability:** why is it broken? (explore metrics/logs/traces)  
- **Pillars:** metrics, logs, traces  

This Phase 1 focused on **metrics + alerts**.

---

## 7) ServiceMonitor vs PodMonitor vs PrometheusRule

| Resource | Purpose |
|---|---|
| **ServiceMonitor** | Tell Prometheus **what to scrape** via a Service |
| **PodMonitor** | Scrape pods directly (less common) |
| **PrometheusRule** | Alert / recording rules (**what to evaluate**) |

In this stack:
```bash
kubectl get servicemonitor -n monitoring   # many exist
kubectl get podmonitor -n monitoring       # often empty (normal)
```

Empty PodMonitor ≠ broken.

### Critical kube-prometheus-stack label
Prometheus selects resources with:
```bash
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.ruleSelector}' ; echo
# {"matchLabels":{"release":"kps"}}
```

So custom `PrometheusRule` / `ServiceMonitor` usually need:

```yaml
labels:
  release: kps
```

`[SS: kubectl get servicemonitor -n monitoring]`

---

## 8) Custom alerts with PrometheusRule

### Mental model
```text
PrometheusRule (label release=kps)
   → Prometheus Operator syncs
   → Prometheus loads rules
   → Inactive / Pending / Firing
   → Alertmanager
```

### Check selectors
```bash
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.ruleSelector}' ; echo
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.ruleNamespaceSelector}' ; echo
```

### First realistic alerts
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: learning-alerts
  namespace: monitoring
  labels:
    release: kps
    app: kube-prometheus-stack
spec:
  groups:
  - name: learning.rules
    rules:
    - alert: PodNotReady
      expr: kube_pod_status_ready{condition="true",namespace="monitoring"} == 0
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} is not ready"
    - alert: HighPodRestart
      expr: increase(kube_pod_container_status_restarts_total{namespace="monitoring"}[15m]) > 3
      for: 1m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} restarting often"
EOF
```

```bash
kubectl get prometheusrule learning-alerts -n monitoring --show-labels
kubectl get prometheusrule learning-alerts -n monitoring -o yaml
```

Look for: `prometheus-operator-validated: "true"`

---

## 9) Why alerts may “not show” + how we debugged

### Common reasons
1. Label mismatch (`release` missing)  
2. Looking only at `/alerts` while state is **Inactive**  
3. Bad in-pod `wget` check (Prometheus image may not have wget)  
4. Need **Status → Rules** first  
5. Keep PromQL `expr` on **one line**

### Unreliable check
```bash
kubectl exec ... -- wget ... /api/v1/rules | grep learning
# can falsely say NOT FOUND
```

### Reliable checks
```bash
# labels match?
kubectl get prometheusrule learning-alerts -n monitoring --show-labels
# release=kps  ✅ matched ruleSelector

# API from student-node (with port-forward)
curl -s http://127.0.0.1:9090/api/v1/rules  | grep -i learning
curl -s http://127.0.0.1:9090/api/v1/alerts | grep -i learning
```

### Prove pipeline with always-firing test
```bash
kubectl delete prometheusrule learning-alerts -n monitoring --ignore-not-found

cat <<'EOF' | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: learning-alerts
  namespace: monitoring
  labels:
    release: kps
spec:
  groups:
  - name: learning.rules
    rules:
    - alert: LearningAlwaysFiring
      expr: vector(1)
      for: 0m
      labels:
        severity: info
      annotations:
        summary: "Learning alert always true"
EOF
```

### Confirmed working via API
```bash
curl -s http://127.0.0.1:9090/api/v1/alerts | grep -i LearningAlwaysFiring
```

Saw:
```json
"alertname":"LearningAlwaysFiring",
"state":"firing",
"summary":"Learning alert always true"
```

### UI places to check
- **Status → Rules (`/rules`)** → group `learning.rules` (loaded?)  
- **Alerts (`/alerts`)** → Inactive / Pending / Firing  
- **Alertmanager** → receives firing alerts  

Also saw noisy K3s alerts (`KubeSchedulerDown`, `KubeProxyDown`, etc.) → ignore for learning.  
`Watchdog` always fires → pipeline health check.

`[SS: Prometheus Alerts with LearningAlwaysFiring firing]`  
`[SS: Status → Rules showing learning.rules]`

---

## 10) Item 5 — Scrape a sample app (end-to-end)

### Flow
```text
App exposes /metrics
  → Service (named port web)
  → ServiceMonitor (label release=kps)
  → Prometheus target UP
  → PromQL / alerts / Grafana
```

### Deploy app + Service
```bash
kubectl create namespace demo

cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-app
  namespace: demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: example-app
  template:
    metadata:
      labels:
        app: example-app
    spec:
      containers:
      - name: example-app
        image: quay.io/brancz/prometheus-example-app:v0.5.0
        ports:
        - name: web
          containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: example-app
  namespace: demo
  labels:
    app: example-app
spec:
  selector:
    app: example-app
  ports:
  - name: web
    port: 8080
    targetPort: web
EOF
```

```bash
kubectl get pods,svc -n demo
```

### ServiceMonitor
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: example-app
  namespace: demo
  labels:
    release: kps
spec:
  selector:
    matchLabels:
      app: example-app
  namespaceSelector:
    matchNames:
      - demo
  endpoints:
  - port: web
    path: /metrics
    interval: 15s
EOF
```

```bash
kubectl get servicemonitor -n demo --show-labels
```

### Generate traffic
App behavior:
| Path | Result | Metric |
|---|---|---|
| `/` | 200 | `http_requests_total{code="200"}` ↑ |
| `/err` | 404 | `http_requests_total{code="404"}` ↑ |
| `/metrics` | scrape endpoint | Prometheus reads this |

```bash
# normal + error traffic
kubectl run curlbox --rm -it --restart=Never -n demo --image=curlimages/curl -- \
  sh -c 'for i in $(seq 1 20); do curl -s http://example-app:8080/; curl -s http://example-app:8080/err; done; echo done'
```

More errors (for alert demos):
```bash
kubectl run curlbox --rm -it --restart=Never -n demo --image=curlimages/curl -- \
  sh -c 'for i in $(seq 1 50); do curl -s -o /dev/null -w "%{http_code}\n" http://example-app:8080/err; done'
```

### Verify Targets (SUCCESS)
Prometheus UI → **Status → Target health** → search `example-app`

Expected:
- Endpoint pool: `serviceMonitor/demo/example-app/0`
- Green bar
- **2 targets UP** (2 pods)
- Labels include `job="example-app"`, `namespace="demo"`, pod names, `endpoint="web"`

`[SS: Targets page — example-app UP (2 endpoints)]`
<img width="1121" height="631" alt="image" src="https://github.com/user-attachments/assets/cf1e1926-48c0-47c0-b94e-9e01cd3f8f5f" />

---

## 11) PromQL checks we ran

### A) Raw counters
```promql
http_requests_total
```
Saw series for both pods with `code="200"` and `code="404"` (proof scrape + traffic).

`[SS: Query table http_requests_total]`
<img width="1515" height="515" alt="image" src="https://github.com/user-attachments/assets/d4cc751f-04ae-490e-a915-83388b97d161" />


### B) Rate over 1m (can be 0 if no fresh traffic)
```promql
rate(http_requests_total[1m])
```
If idle for >1m → values become `0` even if counters are large. Normal.

`[SS: rate(http_requests_total[1m]) showing 0 when idle]`
<img width="1540" height="529" alt="image" src="https://github.com/user-attachments/assets/1bf040f8-9442-41ae-84da-ea8f100c7454" />

### C) Error rate after hitting `/err`
```promql
rate(http_requests_total{code="404",job="example-app"}[1m])
```
Saw non-zero values on both pods (~0.53 / ~0.57) = ~0.5+ 404s per second recently.

`[SS: rate(...code="404"...) non-zero]`
<img width="1513" height="474" alt="image" src="https://github.com/user-attachments/assets/8082864b-3162-4fe5-97bb-421f2d4dfc52" />

### D) Alert-style aggregate
```promql
sum(rate(http_requests_total{code="404",job="example-app"}[2m]))
```
If this `> 0.1`, an alert with that threshold can fire.

---

## 12) Optional app alert

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: example-app-alerts
  namespace: monitoring
  labels:
    release: kps
spec:
  groups:
  - name: example-app.rules
    rules:
    - alert: ExampleAppHighErrors
      expr: sum(rate(http_requests_total{code="404",job="example-app"}[2m])) > 0.1
      for: 1m
      labels:
        severity: warning
      annotations:
        summary: "example-app returning many 404s"
EOF
```

Then:
1. Hit `/err` many times  
2. Wait ~1m (`for: 1m`)  
3. Check Prometheus **Alerts** for `ExampleAppHighErrors`  
4. Optionally check Alertmanager  

Meaning of “Hit `/err` a few more times if you want it to fire”:  
→ generate fresh 404 traffic so `rate(...404...)` stays above threshold long enough for the alert to leave Pending and become Firing.

`[SS: Alerts page ExampleAppHighErrors Pending/Firing]`

---

## 13) Full verification checklist

```bash
# stack
kubectl get pods -n monitoring

# sample app
kubectl get pods,svc -n demo
kubectl get servicemonitor -n demo --show-labels

# rules
kubectl get prometheusrule -n monitoring --show-labels

# prometheus access
kubectl port-forward -n monitoring svc/kps-kube-prometheus-stack-prometheus 9090:9090 --address 0.0.0.0

# api
curl -s http://127.0.0.1:9090/api/v1/targets | grep example-app
curl -sG http://127.0.0.1:9090/api/v1/query --data-urlencode 'query=http_requests_total{job="example-app"}'
curl -s http://127.0.0.1:9090/api/v1/alerts | grep -iE 'LearningAlwaysFiring|ExampleAppHighErrors'
```

UI checklist:
- [ ] Targets: `example-app` UP  
- [ ] Query: `http_requests_total` has series  
- [ ] Query: `rate(...404...)` moves after `/err` traffic  
- [ ] Rules: custom groups visible  
- [ ] Alerts: test/app alerts visible  

---

## 14) Troubleshooting matrix

| Symptom | Cause | Fix |
|---|---|---|
| KodeKloud URL blank | need PF on student-node | `port-forward --address 0.0.0.0` |
| node-exporter CreateContainerError | K3s mount issue | disable node-exporter |
| ServiceMonitor exists, no target | missing `release: kps` | label ServiceMonitor |
| Rule exists, not in Prometheus | label/selector mismatch | `release: kps` |
| Rule in `/rules` but not obvious in `/alerts` | Inactive | condition false; use `vector(1)` test |
| `rate()` is 0 | no recent traffic | hit `/` or `/err` again |
| Many Kube*Down alerts on K3s | expected noise | ignore for learning |

---

## 15) Interview talking points

1. kube-prometheus-stack = Prometheus + Grafana + Alertmanager + Operator + defaults.  
2. Operator uses CRs: ServiceMonitor, PodMonitor, PrometheusRule, Prometheus, Alertmanager.  
3. Scraping and alerting are separate: ServiceMonitor vs PrometheusRule.  
4. Label selectors (`release: kps`) control discovery.  
5. Verify alerts via Status → Rules, then Alerts, then Alertmanager.  
6. `rate()` needs fresh data; counters alone don’t mean current traffic.  
7. node-exporter = host metrics; kube-state-metrics = object metrics.  
8. Can demo end-to-end with a public sample metrics app (no custom coding).  

---

## 16) Phase 1 status

| Item | Status |
|---|---|
| Cluster + Helm ready | Done |
| Install `kps` | Done |
| Core components running | Done |
| node-exporter workaround | Done |
| KodeKloud access pattern | Done |
| Custom PrometheusRule alerts | Done (`LearningAlwaysFiring` proven) |
| ServiceMonitor understanding | Done |
| Sample app scrape | Done (`example-app` targets UP) |
| PromQL (`http_requests_total`, `rate`, 404 filter) | Done |
| Grafana deep-dive / custom dashboards | Optional remaining |
| Docker Compose Phase 2 | Later |

---

## 17) Cleanup (optional)

```bash
kubectl delete prometheusrule learning-alerts example-app-alerts -n monitoring --ignore-not-found
kubectl delete ns demo
# or keep them for demos
```

Stop port-forwards with `Ctrl+C`.

---

## 18) One-page summary

We installed **kube-prometheus-stack** on K3s, worked around **node-exporter**, learned KodeKloud needs **`port-forward --address 0.0.0.0`**, created **`PrometheusRule`** alerts (proven with `LearningAlwaysFiring`), then deployed **`example-app`** + **`ServiceMonitor`** (`release: kps`). Prometheus showed **targets UP**, scraped `http_requests_total`, and after hitting **`/err`**, `rate(...code="404"...)` became non-zero — full **app → scrape → query → alert** loop.

---

Paste your screenshots into every **`[SS: ...]`** slot and this becomes a complete interview/project writeup.


Then **Phase 1 is done.**

You’ve covered the full plan:

- kube-prometheus-stack on K8s  
- cluster/app metrics scraping  
- ServiceMonitor + targets UP  
- custom alerts (`PrometheusRule`)  
- premade Grafana dashboards explored  
- custom Grafana dashboards from PromQL  

**Where you are now:** ready for **Phase 2** (Docker Compose: sample app → `/metrics` → Prometheus → Grafana), whenever you want to start.
