# Exercise 10 – Loki Logging & Troubleshooting

## Objective

Deploy Loki and Promtail into the EKS cluster and integrate them with Grafana for centralized log collection.

Architecture:

Application Pods
        │
        ▼
Promtail
        │
        ▼
Loki
        │
        ▼
Grafana

---

# Prerequisites

- Amazon EKS Cluster
- kubectl configured
- Helm installed
- kube-prometheus-stack already running
- Grafana accessible

---

# Project Structure

```
ex10/
└── README.md
```

---

# Step 1 – Add Grafana Helm Repository

```powershell
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

---

# Step 2 – Install Loki Stack

Used the Loki Stack chart because it includes both:

- Loki
- Promtail

Installation:

```powershell
helm install loki grafana/loki-stack `
    --namespace monitoring `
    --create-namespace
```

---

# Step 3 – Verify Installation

Pods

```powershell
kubectl get pods -n monitoring
```

Expected:

```
loki-0
loki-promtail-xxxxx
```

Helm Release

```powershell
helm list -n monitoring
```

---

# Step 4 – Add Loki Datasource in Grafana

Open Grafana

Connections
↓

Data Sources
↓

Add data source
↓

Loki

URL:

```
http://loki:3100
```

Save the datasource.

---

# Step 5 – Verify Log Collection

Check Promtail logs

```powershell
kubectl logs daemonset/loki-promtail -n monitoring
```

Observed:

- Promtail discovered Kubernetes pods.
- Promtail started tailing container logs.
- Promtail attempted to push logs to Loki.

---

# Troubleshooting Performed

Promtail reported:

```
error sending batch

Post http://loki:3100/loki/api/v1/push

connection refused
```

This indicates:

- Promtail is healthy.
- Log files are being read.
- Loki endpoint is not accepting connections.

Therefore the failure point is:

Application
↓

Promtail
↓

❌ Loki

↓

Grafana

---

# Investigation Commands

View Loki pods

```powershell
kubectl get pods -n monitoring
```

Describe Loki

```powershell
kubectl describe pod loki-0 -n monitoring
```

View Loki logs

```powershell
kubectl logs loki-0 -n monitoring
```

Verify Promtail

```powershell
kubectl logs daemonset/loki-promtail -n monitoring
```

---

# Root Cause Identified

Promtail successfully discovered pod logs and attempted to send them.

The push request failed because Loki was not accepting connections on port 3100.

Failure point:

```
Application
      │
      ▼
Promtail
      │
      ▼
Loki ❌
      │
      ▼
Grafana
```

---

# Key Learning

Understanding the Kubernetes logging pipeline:

```
Application
      │
      ▼
Container Log File
      │
      ▼
Promtail
      │
      ▼
Loki
      │
      ▼
Grafana
```

A logging issue should always be debugged in this order:

1. Is the application generating logs?
2. Is Promtail reading logs?
3. Is Promtail pushing logs?
4. Is Loki receiving logs?
5. Is Grafana querying Loki?

---

# Useful Commands

Check Helm releases

```powershell
helm list -n monitoring
```

Check monitoring pods

```powershell
kubectl get pods -n monitoring
```

Describe Loki

```powershell
kubectl describe pod loki-0 -n monitoring
```

View Loki logs

```powershell
kubectl logs loki-0 -n monitoring
```

View Promtail logs

```powershell
kubectl logs daemonset/loki-promtail -n monitoring
```

Port-forward Grafana

```powershell
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring
```

---

# Outcome

✔ Installed Loki

✔ Installed Promtail

✔ Integrated Loki with Grafana

✔ Verified Promtail was collecting Kubernetes logs

✔ Traced the complete log pipeline

✔ Practiced production-style troubleshooting by identifying the failure point between Promtail and Loki

---

## Skills Practiced

- Helm
- Kubernetes Logging
- Loki
- Promtail
- Grafana Datasources
- Log Pipeline Debugging
- Production Incident Investigation