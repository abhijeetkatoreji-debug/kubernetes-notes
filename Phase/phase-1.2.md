You **don’t need an app yet**. The cluster already produces metrics via kubelet, kube-state-metrics, cAdvisor, and (if healthy) node-exporter. That’s enough to learn the full loop.

## What next (no app)

### 1) Open Grafana and explore default dashboards
Keep this running on student-node:

```bash
kubectl port-forward -n monitoring svc/kps-grafana 31346:80 --address 0.0.0.0
```

Open your KodeKloud URL → login **admin / admin123**.

In Grafana:
- Go to **Dashboards**
- Open anything like:
  - **Kubernetes / Compute Resources / Cluster**
  - **Kubernetes / Compute Resources / Node (Pods)**
  - **Kubernetes / Compute Resources / Namespace (Pods)**

Click around: time range, one panel → **Explore**, notice PromQL under the panel.

### 2) Check Prometheus targets
```bash
kubectl port-forward -n monitoring svc/kps-kube-prometheus-stack-prometheus 9090:9090 --address 0.0.0.0
```

Open `https://9090-port-<your-lab>.labs.kodekloud.com/`  
Go to **Status → Targets**.

You should see jobs like:
- `apiserver`
- `kubelet`
- `kube-state-metrics`
- `prometheus`
- `grafana` (sometimes)

**Up** = scraping works. **Down** = misconfig / network / missing exporter.

### 3) Understand ServiceMonitor / PodMonitor (no app needed)
These tell the Prometheus Operator *what to scrape*.

```bash
# List existing ones (kube-prometheus-stack already created many)
kubectl get servicemonitor -n monitoring
kubectl get podmonitor -n monitoring

# Inspect one
kubectl get servicemonitor -n monitoring -o name | head
kubectl describe servicemonitor -n monitoring kps-kube-prometheus-stack-kubelet
```

Mental model:
- **ServiceMonitor** → scrape via a **Service** (most common)
- **PodMonitor** → scrape **Pods** directly (when no Service)

### 4) Build 1–2 simple dashboards (still no app)
In Grafana → **Explore** → Prometheus datasource, try:

```promql
count(kube_pod_info)
```

```promql
sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (namespace)
```

```promql
sum(kube_pod_status_phase{phase="Running"}) by (namespace)
```

Then: **Add to dashboard** → save as e.g. `My Cluster Overview`.

### 5) Create a couple of alerts
Easiest path in this stack: PrometheusRule.

Example — alert if any pod not ready in `monitoring`:

```bash
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
    - alert: PodNotReady
      expr: kube_pod_status_ready{condition="true", namespace="monitoring"} == 0
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

Check in Prometheus UI: **Alerts**, and in Grafana: **Alerting** (depending on setup).

Also verify rule picked up:

```bash
kubectl get prometheusrule -n monitoring
```

### 6) Sample app — only later (optional)
After the above feels clear, deploy a tiny metrics app + ServiceMonitor. Skip until steps 1–5 are comfortable.

---

## Suggested order today
1. Grafana dashboards (10–15 min)  
2. Prometheus Targets (10 min)  
3. `kubectl get/describe servicemonitor` (10 min)  
4. One custom dashboard from Explore  
5. One PrometheusRule alert  

If you want, next message we do **step 1 live**: you say what dashboards you see, and I’ll tell you exactly what to click and what each panel means.
