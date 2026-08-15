# Prometheus Alerting on Kubernetes — Full Learning Notes

**Stack:** kube-prometheus-stack (Helm release `kps`) in namespace `monitoring`  
**Cluster:** K3s (KodeKloud lab)  
**Goal:** Create a custom alert with `PrometheusRule`, verify Prometheus loads it, see it in UI/API

---

## 0) Mental model (interview-ready)

```text
PrometheusRule (YAML in K8s)
        │
        ▼
Prometheus Operator (watches CRs, must match ruleSelector labels)
        │
        ▼
Prometheus loads rule files + evaluates PromQL
        │
        ├── Inactive  → condition false
        ├── Pending   → condition true, waiting for `for:` duration
        └── Firing    → sends to Alertmanager
                │
                ▼
         Alertmanager (grouping, routing, notifications)
```

**Key objects**
| Object | Role |
|---|---|
| `Prometheus` CR | The Prometheus instance + selectors |
| `PrometheusRule` | Your alert/recording rules |
| `Alertmanager` | Receives firing alerts |
| ServiceMonitor | Scraping targets (separate from alerts) |

**Critical label rule for kube-prometheus-stack**
- Prometheus selects rules with: `ruleSelector.matchLabels.release: kps`
- Your `PrometheusRule` **must** have label: `release: kps`

---

## 1) Prerequisites (stack already installed)

```bash
kubectl get pods -n monitoring
kubectl get prometheus -n monitoring
kubectl get prometheusrule -n monitoring
kubectl get servicemonitor -n monitoring
```

Access Prometheus UI on KodeKloud (port-forward on student-node):

```bash
kubectl port-forward -n monitoring svc/kps-kube-prometheus-stack-prometheus 9090:9090 --address 0.0.0.0
```

Open: `https://9090-port-<lab-id>.labs.kodekloud.com/`

Grafana (optional, separate from PrometheusRule UI):

```bash
kubectl port-forward -n monitoring svc/kps-grafana 31346:80 --address 0.0.0.0
```

Login: `admin` / `admin123` (or get secret password if changed)

---

## 2) Check what Prometheus will accept (selectors)

```bash
# What labels must rules have?
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.ruleSelector}' ; echo
# Expected something like:
# {"matchLabels":{"release":"kps"}}

# Which namespaces are allowed?
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.ruleNamespaceSelector}' ; echo
# {}  => typically all namespaces (or same-ns behavior depending on docs); our rule is in monitoring anyway

# Existing chart rules + labels
kubectl get prometheusrule -n monitoring --show-labels
```

---

## 3) Create a realistic learning alert (first attempt)

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

Verify object exists:

```bash
kubectl get prometheusrule learning-alerts -n monitoring --show-labels
kubectl get prometheusrule learning-alerts -n monitoring -o yaml
```

Look for annotation:
- `prometheus-operator-validated: "true"` → YAML accepted by operator

---

## 4) Why it may “not show” in UI (what we hit)

### Common reasons
1. **Wrong labels** → Operator ignores rule (not our final case; labels matched)
2. **Looking only at `/alerts` while state is Inactive** → easy to miss
3. **Bad in-pod check** (`wget` missing in Prometheus image) → false “NOT FOUND”
4. **YAML line-break on expressions** (e.g. `> 3` split) → avoid by keeping expr on one line
5. **Need Status → Rules**, not only Alerts page

### Unreliable check (don’t trust alone)
```bash
kubectl exec -n monitoring prometheus-kps-kube-prometheus-stack-prometheus-0 -c prometheus -- \
  wget -qO- http://127.0.0.1:9090/api/v1/rules 2>/dev/null \
  | grep -iE 'learning|PodNotReady' || echo "NOT FOUND IN PROMETHEUS API"
```
If image has no `wget`, this prints NOT FOUND even when rules exist.

### Better checks
**A) From student-node via port-forward + curl**
```bash
curl -s http://127.0.0.1:9090/api/v1/rules   | grep -iE 'learning|PodNotReady' || echo "missing in /api/v1/rules"
curl -s http://127.0.0.1:9090/api/v1/alerts  | grep -iE 'learning|PodNotReady' || echo "missing in /api/v1/alerts"
```

**B) Rule files inside Prometheus pod**
```bash
kubectl exec -n monitoring prometheus-kps-kube-prometheus-stack-prometheus-0 -c prometheus -- \
  sh -c 'ls -la /etc/prometheus/rules/ 2>/dev/null; grep -R "PodNotReady\|learning" /etc/prometheus/rules 2>/dev/null || echo "not in rule files"'
```

**C) Operator sync logs**
```bash
kubectl logs -n monitoring deploy/kps-kube-prometheus-stack-operator --tail=100
# Look for: sync prometheus ... key=monitoring/kps-kube-prometheus-stack-prometheus
```

**D) Fix labels if needed**
```bash
kubectl label prometheusrule learning-alerts -n monitoring release=kps --overwrite
kubectl label prometheusrule learning-alerts -n monitoring app=kube-prometheus-stack --overwrite
```

---

## 5) Prove the pipeline with an always-firing test alert

This is the best learning trick.

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

Wait 30–60 seconds for operator sync + evaluation.

### Verify via API (worked for us)
```bash
curl -s http://127.0.0.1:9090/api/v1/alerts | grep -i LearningAlwaysFiring || echo "not in alerts api"
```

Successful response included:

```json
{
  "labels": {
    "alertname": "LearningAlwaysFiring",
    "severity": "info"
  },
  "annotations": {
    "summary": "Learning alert always true"
  },
  "state": "firing"
}
```

Also saw built-in alerts like:
- `Watchdog` (always-firing pipeline health check)
- `KubeControllerManagerDown` / `KubeSchedulerDown` / `KubeProxyDown`  
  → common noise on **K3s**; ignore for learning

### Optional file check
```bash
kubectl exec -n monitoring prometheus-kps-kube-prometheus-stack-prometheus-0 -c prometheus -- \
  sh -c 'grep -R LearningAlwaysFiring /etc/prometheus/rules || echo "not in files yet"'
```

---

## 6) UI walkthrough (Prometheus)

With port-forward to `9090`:

### A) Alerts page
1. Open `/alerts` (menu: **Alerts**)
2. Search: `LearningAlwaysFiring`
3. States:
   - **Inactive** = not matching
   - **Pending** = matching, waiting `for:`
   - **Firing** = active alert
4. Expand alert → see labels, annotations, value, activeSince

### B) Rules page (most important for “was it loaded?”)
1. Open **Status → Rules** (`/rules`)
2. Find group: `learning.rules`
3. See alert name + expression + health
4. If it’s here, Prometheus loaded your `PrometheusRule` successfully

### C) Graph / Explore expression
1. Go to **Graph**
2. Run: `vector(1)`
3. Run your real expr later, e.g.:
   ```promql
   kube_pod_status_ready{condition="true",namespace="monitoring"} == 0
   ```

### D) Targets (related, not alerts)
**Status → Targets** shows scrape jobs (ServiceMonitors), not PrometheusRules.

---

## 7) Alertmanager side (next hop after firing)

```bash
kubectl port-forward -n monitoring svc/kps-kube-prometheus-stack-alertmanager 9093:9093 --address 0.0.0.0
```

UI: `/#/alerts`  
You should eventually see `LearningAlwaysFiring` if routing is default/open.

Check Alertmanager service:
```bash
kubectl get svc -n monitoring | grep alertmanager
kubectl get pods -n monitoring | grep alertmanager
```

---

## 8) Put back “real” alerts (after test)

```bash
kubectl delete prometheusrule learning-alerts -n monitoring

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

Notes:
- Keep each `expr` on **one line**
- These may stay **Inactive** until a pod is not ready / restarts a lot — that’s normal
- Use `LearningAlwaysFiring` when you need to demo the pipeline

---

## 9) Full verification checklist (copy/paste)

```bash
# Object + labels
kubectl get prometheusrule learning-alerts -n monitoring --show-labels

# Selector match
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.ruleSelector}' ; echo

# Port-forward Prometheus (keep running)
kubectl port-forward -n monitoring svc/kps-kube-prometheus-stack-prometheus 9090:9090 --address 0.0.0.0

# API checks (other terminal)
curl -s http://127.0.0.1:9090/api/v1/rules  | grep -i learning || echo "not in rules api"
curl -s http://127.0.0.1:9090/api/v1/alerts | grep -i learning || echo "not in alerts api"

# UI
# /rules  -> learning.rules
# /alerts -> LearningAlwaysFiring (firing) or PodNotReady (maybe inactive)
```

---

## 10) Troubleshooting matrix

| Symptom | Likely cause | Fix |
|---|---|---|
| CR exists, not in `/rules` | label mismatch | add `release: kps` |
| In `/rules` Inactive, not obvious on `/alerts` | condition false | check PromQL; use `vector(1)` test |
| `wget` says NOT FOUND | no wget in image | use `curl` via port-forward |
| Operator validated true but delayed | sync lag | wait 30–60s; check operator sync logs |
| Many `Kube*Down` alerts on K3s | expected scrape gaps | ignore for learning |
| Grafana Alerting empty | different system | PrometheusRule ≠ Grafana-managed alerts by default |

---

## 11) ServiceMonitor vs PodMonitor vs PrometheusRule

| Resource | Purpose |
|---|---|
| **ServiceMonitor** | Tell Prometheus **what to scrape** (via Service) |
| **PodMonitor** | Scrape pods directly (less common) |
| **PrometheusRule** | Alert/recording rules (**what to evaluate**) |

In this stack, `kubectl get podmonitor -n monitoring` can be empty while ServiceMonitors exist — normal.

---

## 12) Interview talking points

1. kube-prometheus-stack installs Prometheus + Grafana + Alertmanager + Operator + default rules.
2. Custom alerts = `PrometheusRule` with labels matching Prometheus `ruleSelector`.
3. Operator syncs rules into Prometheus config/rule files.
4. Verify with `/rules` first, `/alerts` second, then Alertmanager.
5. `Watchdog` / `vector(1)` prove alerting pipeline health.
6. Scraping (ServiceMonitor) and alerting (PrometheusRule) are separate concerns.
7. On K3s, some default control-plane target alerts fire noisily.

---

## 13) Cleanup

```bash
# Remove learning rule
kubectl delete prometheusrule learning-alerts -n monitoring

# Stop port-forwards with Ctrl+C in those terminals
```

---

## 14) What we personally proved in this lab

1. Installed `kps` (kube-prometheus-stack).
2. Disabled broken node-exporter for K3s mount issue.
3. Created `PrometheusRule/learning-alerts` with `release: kps`.
4. Confirmed selector: `{"matchLabels":{"release":"kps"}}`.
5. First UI confusion: inactive alerts + bad wget check.
6. Recreated with `LearningAlwaysFiring` (`expr: vector(1)`).
7. Confirmed via:
   ```bash
   curl -s http://127.0.0.1:9090/api/v1/alerts | grep LearningAlwaysFiring
   ```
8. Saw `"state":"firing"` — full alert pipeline works.

---

**One-liner summary:**  
Create `PrometheusRule` with `release: kps` → Operator syncs → check **Status → Rules** → confirm firing on **Alerts** / `curl /api/v1/alerts` → then Alertmanager.
