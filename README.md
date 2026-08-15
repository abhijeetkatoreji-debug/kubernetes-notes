# kubernetes-notes

- **Don't stress building a "real" app from scratch** — no need for auth, a database, clean architecture, tests, or production-grade code. Interviewers care that you can *observe* a running system, not that you wrote the app yourself.

- **Skip perfect application code** — bugs, messy structure, and "just enough to run" are fine. Your goal is metrics, logs, traces, alerts, and dashboards — not becoming a full-stack dev overnight.

- **Use an existing sample app** — smart move. Things like Spring Boot PetClinic, a Go microservices demo, OpenTelemetry demo, or a simple Node/Python Flask todo API work well. Clone it, run it with Docker, and move on.

- **Focus on the observability stack** — Prometheus scraping, Grafana dashboards, alert rules, log aggregation (Loki/ELK), and optionally tracing (Jaeger/Tempo). That’s what DevOps/cloud interviews probe.

- **Run everything in containers** — Docker Compose is your friend. One `docker-compose up` beats days of local version hell.

- **Instrument > invent** — learn to expose metrics (`/metrics`, Prometheus client libs, OTel) on an app that already exists. You don’t need to design the business logic — just make it *visible*.

- **Build a small but complete story** — e.g. “app runs → Prometheus collects metrics → Grafana shows CPU/latency/errors → alert fires on high error rate.” That end-to-end demo beats a fancy app with no monitoring.

- **Document what you did, not what you coded** — a short README with architecture diagram, dashboard screenshots, and sample alert rules looks strong in interviews.

Yes — we can pick an existing sample app and I’ll help you wire Prometheus + Grafana around it. Want to start with that next?
