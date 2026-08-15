Here’s where you stand against that plan:

**Done so far (Phase 1 started)**
- Confirmed cluster: K3s, 3 nodes, Helm ready
- Installed **kube-prometheus-stack** (`kps`) in `monitoring`
- Core pieces running: **Prometheus, Grafana, Alertmanager, kube-state-metrics, operator**
- Fixed / worked around **node-exporter** issues on K3s
- Switched Grafana service toward lab access and learned KodeKloud needs **`port-forward --address 0.0.0.0`**, not just NodePort

**Not done yet (still Phase 1)**
- Open Grafana and explore default cluster dashboards
- Check Prometheus **targets** (what’s being scraped)
- Understand **ServiceMonitor / PodMonitor**
- Build 1–2 simple dashboards
- Create a couple of **alerts**
- Optionally scrape a sample app (still optional for now)

**Not started**
- Phase 2: Docker Compose + app `/metrics` exposure

**Bottom line:** Stack is installed and mostly healthy. Next real learning step is: get into Grafana → look at cluster metrics → then Prometheus targets.
