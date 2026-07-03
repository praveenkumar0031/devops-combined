# Exercise 9 – Kubernetes Monitoring with Prometheus & Grafana

## Objective

Set up a complete monitoring stack on an Amazon EKS cluster using the **kube-prometheus-stack Helm Chart**, visualize Kubernetes metrics in Grafana, and understand how Prometheus collects metrics from the cluster.

---

# Architecture

```
Kubernetes Cluster
        │
        ▼
+---------------------------+
| kube-prometheus-stack     |
|                           |
|  Prometheus               |
|      │                    |
|      ▼                    |
|  Node Exporter            |
|      │                    |
|  kube-state-metrics       |
|      │                    |
|  Alertmanager             |
|      │                    |
|  Grafana                  |
+---------------------------+
```

---

# Prerequisites

- AWS EKS Cluster
- kubectl configured
- Helm installed
- Worker nodes in Ready state

---

# Step 1 – Add Prometheus Helm Repository

```powershell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

---

# Step 2 – Create Monitoring Namespace

```powershell
kubectl create namespace monitoring
```

---

# Step 3 – Install kube-prometheus-stack

```powershell
helm install monitoring prometheus-community/kube-prometheus-stack `
  -n monitoring
```

This installs:

- Prometheus
- Grafana
- Alertmanager
- Node Exporter
- kube-state-metrics
- Prometheus Operator

---

# Step 4 – Verify Installation

Watch the pods until every pod becomes Running.

```powershell
kubectl get pods -n monitoring -w
```

Expected pods:

- monitoring-grafana
- monitoring-kube-prometheus-operator
- prometheus-monitoring-kube-prometheus-prometheus
- alertmanager-monitoring-kube-prometheus-alertmanager
- monitoring-kube-state-metrics
- monitoring-prometheus-node-exporter

---

# Step 5 – Verify Helm Release

```powershell
helm list -n monitoring
```

Example output:

```
NAME         monitoring
STATUS       deployed
CHART        kube-prometheus-stack
```

---

# Step 6 – Port Forward Grafana

```powershell
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring
```

Open

```
http://localhost:3000
```

---

# Step 7 – Get Grafana Password

```powershell
kubectl get secret monitoring-grafana -n monitoring -o jsonpath="{.data.admin-password}" | % { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }
```

Username:

```
admin
```

Password:

```
(Output from previous command)
```

---

# Step 8 – Login to Grafana

Login using

- Username: admin
- Password: Retrieved from Kubernetes Secret

---

# Step 9 – Explore Dashboards

Navigate to

```
Dashboards
```

Useful dashboards explored:

- Kubernetes Cluster Overview
- Kubernetes Nodes
- Kubernetes Pods
- Kubernetes Compute Resources
- Node Exporter Dashboard

---

# Step 10 – Explore Metrics

Open

```
Explore
```

Datasource:

```
Prometheus
```

Example PromQL queries:

### CPU Usage

```promql
rate(node_cpu_seconds_total[5m])
```

### Node Memory

```promql
node_memory_MemAvailable_bytes
```

### Node Filesystem

```promql
node_filesystem_avail_bytes
```

### Pod Count

```promql
count(kube_pod_info)
```

### Running Pods

```promql
kube_pod_status_phase{phase="Running"}
```

### Cluster Nodes

```promql
count(kube_node_info)
```

### HTTP Requests (if applications expose metrics)

```promql
rate(http_requests_total[5m])
```

---

# Experiment Performed

To understand monitoring practically:

- Opened Grafana Explore
- Executed PromQL queries
- Observed live node metrics
- Verified Prometheus scraping cluster metrics
- Explored built-in Kubernetes dashboards

---

# Components Installed

| Component | Purpose |
|------------|---------|
| Prometheus | Collects metrics |
| Grafana | Visualizes metrics |
| Alertmanager | Sends alerts |
| kube-state-metrics | Kubernetes object metrics |
| Node Exporter | Node hardware metrics |
| Prometheus Operator | Manages Prometheus resources |

---

# Useful Commands

Check monitoring pods

```powershell
kubectl get pods -n monitoring
```

Check services

```powershell
kubectl get svc -n monitoring
```

Check Helm release

```powershell
helm list -n monitoring
```

Port forward Grafana

```powershell
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring
```

Port forward Prometheus

```powershell
kubectl port-forward svc/monitoring-kube-prometheus-prometheus 9090:9090 -n monitoring
```

View Prometheus targets

```
Status → Targets
```

---

# What I Learned

- How Prometheus scrapes Kubernetes metrics.
- Role of Prometheus Operator in managing monitoring resources.
- Difference between Node Exporter and kube-state-metrics.
- How Grafana queries Prometheus using PromQL.
- How to visualize cluster health using prebuilt dashboards.
- How monitoring differs from logging:
  - **Prometheus** stores metrics (CPU, memory, network, pod status).
  - **Grafana** visualizes those metrics.
  - Logs require a separate system like **Loki**, which will be configured in the next exercise.

---

# Cleanup (Optional)

Delete the monitoring stack

```powershell
helm uninstall monitoring -n monitoring
```

Delete the namespace

```powershell
kubectl delete namespace monitoring
```

> **Note:** The monitoring stack was intentionally kept running because it will be reused in the next exercises (Loki logging and observability).