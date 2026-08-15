Got it — **Kubernetes first**, Docker Compose later.

**Plan**
- **Phase 1 (now):** Learn monitoring on K8s — deploy Prometheus + Grafana (usually kube-prometheus-stack), scrape cluster/node/pod metrics, build dashboards, set a couple of alerts.
- **What you skip early:** Writing an app from scratch, perfect app code, full microservices design.
- **What you focus on:** How Prometheus discovers targets in K8s (ServiceMonitor/PodMonitor), Grafana dashboards for cluster health, and basic alerting.
- **App for Phase 1:** Use a ready sample (or even just monitor the cluster itself first — nodes, pods, kube-state-metrics).
- **Phase 2 (later):** Docker Compose path — run a sample app locally, expose `/metrics`, scrape with Prometheus, dashboards in Grafana.

When you’re ready, say the word and we’ll start Phase 1 with a simple K8s + Prometheus + Grafana setup.

---

Yes — on KodeKloud, `port-forward` to `localhost:3000` usually won’t open in the lab browser. Use **NodePort**.

```bash
# Switch Grafana to NodePort
helm upgrade kps prometheus-community/kube-prometheus-stack \
  -n monitoring \
  --reuse-values \
  --set grafana.service.type=NodePort

# Get the NodePort
kubectl get svc -n monitoring kps-grafana
```

You’ll see something like `80:3xxxx/TCP` — that **3xxxx** is the NodePort.

In KodeKloud, open the app using that NodePort (lab “Port” / “NodePort” access), then login:
- user: **admin**
- pass: **admin123**

Optional — same for Prometheus UI:

```bash
helm upgrade kps prometheus-community/kube-prometheus-stack \
  -n monitoring \
  --reuse-values \
  --set prometheus.service.type=NodePort

kubectl get svc -n monitoring | grep prometheus
```

Paste `kubectl get svc -n monitoring` if you want help picking the exact URL/port.

----
The KodeKloud port URL proxies to **student-node’s localhost** on that port — but **NodePort opens the port on the cluster nodes**, not on student-node. That’s why `31346-port-...` can’t reach it.

So on KodeKloud you actually **do need port-forward on student-node**, bound to all interfaces, using a port that matches the URL prefix.

```bash
kubectl port-forward -n monitoring svc/kps-grafana 31346:80 --address 0.0.0.0
```

Keep that running, then open:

`https://31346-port-4fqyv56lw3oppdqe.labs.kodekloud.com/`

Notes:
- The **number in the URL must match the local port** you forward to. The URL says `31346`, so forward `31346:80`.
- `--address 0.0.0.0` is required so the lab proxy (not just localhost) can reach it.
- Leave the command running in that terminal; open a second terminal for other commands.
- Login: **admin** / **admin123**

So: NodePort matters only if you hit a **node IP directly**; for the KodeKloud `*-port-*` URL, use `port-forward` on student-node as above. Want the same for Prometheus? Use a different port, e.g. `9090:9090`.



---
