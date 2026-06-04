# Kubernetes Observability Stack

Every time I've joined a new team, the first question during an incident is the same: "where are the logs?" followed shortly by "why can't I find anything useful in them?" This stack is the answer I've landed on — it covers metrics, logs, and dashboards in a way that's actually useful during a 2 AM incident rather than just looking good in a demo.

Metrics: Prometheus + Grafana. Logs: Fluentd (DaemonSet) → Elasticsearch → Kibana. Both deployed to a dedicated observability namespace on EKS, provisioned with Terraform.

## What's in here

```
terraform/
  └── EKS cluster, VPC, node groups (same module pattern as eks-production-setup)

kubernetes/
  prometheus/    — ServiceMonitors, PrometheusRule, kube-state-metrics
  grafana/       — Deployment, datasource ConfigMap, dashboard ConfigMaps
  fluentd/       — DaemonSet, ConfigMap with Elasticsearch output config
  elasticsearch/ — StatefulSet with persistent storage
  kibana/        — Deployment, index pattern setup job

dashboards/
  └── Grafana dashboard JSON exports (import these directly in the UI)
```

## How the logging pipeline works

Fluentd runs as a DaemonSet on every node. It reads container logs from `/var/lib/docker/containers` and `/var/log`, enriches them with Kubernetes metadata (pod name, namespace, labels), and ships them to Elasticsearch. Kibana is the query interface.

The Fluentd config includes buffering so log spikes don't cause backpressure on the nodes. The buffer flushes every 5 seconds under normal conditions and backs off exponentially if Elasticsearch is slow.

## Deploying

```bash
# Provision EKS
cd terraform && terraform init && terraform apply

# Create namespaces
kubectl create namespace monitoring
kubectl create namespace logging

# Deploy monitoring stack
kubectl apply -f kubernetes/prometheus/
kubectl apply -f kubernetes/grafana/

# Deploy logging stack (order matters — ES before Kibana before Fluentd)
kubectl apply -f kubernetes/elasticsearch/
kubectl wait --for=condition=ready pod -l app=elasticsearch -n logging --timeout=120s
kubectl apply -f kubernetes/kibana/
kubectl apply -f kubernetes/fluentd/

# Import Grafana dashboards
# In Grafana UI: Dashboards → Import → Upload JSON from dashboards/
```

## Grafana datasources

Two datasources are configured automatically via ConfigMap:
- **Prometheus** at `http://prometheus-server.monitoring.svc.cluster.local`
- **Elasticsearch** at `http://elasticsearch-master.logging.svc.cluster.local:9200`

This means you can query both from Grafana — metrics and logs side by side. Useful for correlating a CPU spike with log events from the same time window.

## What the dashboards cover

**EKS Cluster Overview** — node CPU/memory, pod count per namespace, PVC usage, pending pods
**Application SLIs** — request rate, error rate, latency p50/p95/p99 (requires app to expose Prometheus metrics)
**Log Volume by Namespace** — helps spot noisy services or error rate increases before they show up in metrics

## Kibana index pattern

On first deploy, create the index pattern `kubernetes-logs-*` in Kibana. The important fields to add as column headers: `kubernetes.namespace_name`, `kubernetes.pod_name`, `log`, `level` (if your app uses structured logging).
