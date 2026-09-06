# DevOps Project #4 — Kubernetes Observability & Monitoring

A production-style Kubernetes observability stack built with **Prometheus, Grafana, Helm, and kube-prometheus-stack** on an AWS EC2 server running **k3s**.

The project provides real-time infrastructure and Kubernetes monitoring, custom Grafana dashboards, alerting, email notifications, and end-to-end alert testing.

---

## 🚀 Project Overview

The objective of this project was to implement a complete observability solution for a Kubernetes-hosted application.

The monitoring stack collects and visualizes:

* Kubernetes cluster health
* Node CPU utilization
* Memory utilization
* Disk utilization
* Network traffic
* Pod status
* Pod counts by namespace
* Deployment availability
* Container CPU usage
* Container memory usage
* Prometheus target health

The project also implements automated alerts with Gmail notifications and validates both **firing and resolved alert states**.

---

## 🏗️ Architecture

```text
                         AWS EC2
                    Ubuntu Linux + k3s
                           │
                           │
                    Kubernetes Cluster
                           │
          ┌────────────────┼────────────────┐
          │                │                │
      devops-app        Traefik          Monitoring
      Deployment                           Stack
       3 Pods                                │
                                             │
                         ┌───────────────────┼───────────────────┐
                         │                   │                   │
                    Prometheus           Grafana            Alertmanager
                         │                   │                   │
             ┌───────────┼───────────┐      │                   │
             │           │           │      │                   │
        Node Exporter  kube-state-  Kubernetes             Notifications
                       metrics      Metrics                     │
                                                               │
                                                               ▼
                                                            Gmail
```

---

## 🛠️ Technology Stack

| Technology         | Purpose                              |
| ------------------ | ------------------------------------ |
| AWS EC2            | Cloud infrastructure                 |
| Ubuntu 24.04 LTS   | Server operating system              |
| k3s                | Lightweight Kubernetes               |
| Helm               | Kubernetes package management        |
| Prometheus         | Metrics collection and monitoring    |
| Grafana            | Metrics visualization and dashboards |
| Alertmanager       | Alert management                     |
| Node Exporter      | Linux node metrics                   |
| kube-state-metrics | Kubernetes object/state metrics      |
| Kubernetes         | Container orchestration              |
| Gmail SMTP         | Alert notifications                  |
| MobaXterm          | SSH and local port tunneling         |

---

## 📦 Monitoring Stack

The project uses the Prometheus Community Helm chart:

```text
prometheus-community/kube-prometheus-stack
```

Chart version:

```text
89.2.0
```

Application version:

```text
v0.93.1
```

The monitoring components run in the:

```text
monitoring
```

namespace.

---

## 📊 Grafana Dashboard

Custom dashboard:

```text
DevOps Observability
```

The dashboard contains **13 monitoring panels**.

### Dashboard Panels

1. CPU Usage %
2. Memory Usage %
3. Disk Usage %
4. Network Receive
5. Network Transmit
6. Target Health
7. Kubernetes Pod Count
8. Running Pods
9. Pods by Namespace
10. Deployment Desired Replicas
11. Deployment Available Replicas
12. Container CPU Usage
13. Container Memory Usage

Dashboard configuration:

```text
Time Range: Last 6 hours
Refresh: 1 minute
```

---

## 📈 Example Prometheus Queries

### CPU Usage

```promql
100 * (
  1 - avg by(instance) (
    rate(node_cpu_seconds_total{mode="idle"}[5m])
  )
)
```

### Memory Usage

```promql
100 * (
  1 - node_memory_MemAvailable_bytes
  / node_memory_MemTotal_bytes
)
```

### Disk Usage

```promql
100 * (
  1 -
  node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"}
  /
  node_filesystem_size_bytes{fstype!~"tmpfs|overlay"}
)
```

### Kubernetes Pod Count

```promql
count(kube_pod_info)
```

### Running Pods

```promql
count(
  kube_pod_status_phase{phase="Running"}
)
```

### Container CPU Usage

```promql
sum by (namespace, pod) (
  rate(
    container_cpu_usage_seconds_total{
      container!="",
      container!="POD"
    }[5m]
  )
)
```

### Container Memory Usage

```promql
sum by (namespace, pod) (
  container_memory_working_set_bytes{
    container!="",
    container!="POD"
  }
)
```

---

# 🚨 Alerting

Five Grafana-managed alert rules were implemented.

| Alert                   | Condition              | Evaluation      | Notification |
| ----------------------- | ---------------------- | --------------- | ------------ |
| High CPU Usage          | CPU > 80%              | 1m / 5m pending | Gmail        |
| High Storage Usage      | Storage > 80%          | 1m / 5m pending | Gmail        |
| Kubernetes Pod Restart  | Restart increase > 0   | 1m / 5m pending | Gmail        |
| Deployment Availability | Available replicas < 3 | 1m / 5m pending | Gmail        |
| Prometheus Target Down  | `up < 1`               | 1m / 5m pending | Gmail        |

All five alerts were tested using controlled failure/stress scenarios.

---

## 🔥 Alert Testing

### 1. High CPU Usage

CPU load was intentionally generated using:

```bash
stress-ng --cpu 0 --cpu-load 95 --timeout 8m
```

Result:

```text
Normal → Pending → Firing → Normal
```

A real firing notification was received through Gmail, followed by a resolved notification after CPU usage returned to normal.

---

### 2. High Storage Usage

A temporary 6 GB file was created to increase root filesystem utilization:

```bash
sudo fallocate -l 6G /tmp/storage-alert-test
```

The filesystem crossed the 80% threshold and triggered the alert.

After testing:

```bash
sudo rm /tmp/storage-alert-test
```

Result:

```text
Normal → Firing → Normal
```

---

### 3. Kubernetes Pod Restart

A temporary Kubernetes pod was intentionally configured to exit repeatedly:

```bash
kubectl run restart-test \
  --image=busybox:1.36 \
  --restart=Always \
  -- /bin/sh -c "sleep 10; exit 1"
```

The container restarted multiple times and triggered:

```text
Kubernetes Pod Restart
```

The alert email was successfully received.

The temporary test pod was then removed.

---

### 4. Deployment Availability

The application deployment normally runs three replicas:

```text
3/3 Available
```

The deployment was temporarily scaled down:

```bash
kubectl scale deployment devops-app --replicas=2
```

This triggered the availability alert.

The deployment was restored:

```bash
kubectl scale deployment devops-app --replicas=3
```

Result:

```text
3/3 Available
```

---

### 5. Prometheus Target Down

The Node Exporter target was intentionally made unreachable by temporarily blocking TCP port 9100:

```bash
sudo iptables -I INPUT -p tcp --dport 9100 -j DROP
```

Prometheus detected the target as unavailable and triggered:

```text
Prometheus Target Down
```

After testing, the firewall rule was removed:

```bash
sudo iptables -D INPUT -p tcp --dport 9100 -j DROP
```

The Node Exporter target recovered and the alert returned to:

```text
Normal
```

---

# 📧 Email Notifications

Grafana was configured to send alert notifications through Gmail SMTP.

The SMTP credentials are stored securely in a Kubernetes Secret rather than being hard-coded into the configuration.

Secret:

```text
grafana-smtp
```

SMTP server:

```text
smtp.gmail.com:587
```

Email notifications were tested successfully.

Both:

* Firing notifications
* Resolved notifications

were validated.

> Credentials and Gmail App Passwords are intentionally excluded from this repository.

---

# 🔐 Security Considerations

The project follows several basic security practices:

* Gmail App Password used instead of the account password
* SMTP credentials stored in a Kubernetes Secret
* Credentials excluded from Git
* Prometheus and Grafana exposed through controlled port forwarding/tunneling
* Temporary firewall rule removed after testing
* No sensitive credentials committed to the repository

---

# 🔌 Access

Grafana:

```text
http://localhost:3000
```

Prometheus:

```text
http://localhost:9090
```

These local ports are exposed through SSH tunneling using MobaXterm.

Kubernetes services remain internally accessible through ClusterIP.

---

# 🧪 Useful Validation Commands

### Check all pods

```bash
kubectl get pods -A
```

### Check deployments

```bash
kubectl get deployments -A
```

### Check monitoring services

```bash
kubectl get svc -n monitoring
```

### Check Helm release

```bash
helm list -n monitoring
```

### Check monitoring release status

```bash
helm status kube-prometheus-stack -n monitoring
```

### Check Node Exporter

```bash
kubectl get pods \
  -n monitoring \
  -l app.kubernetes.io/name=prometheus-node-exporter
```

### Check Prometheus readiness

```bash
curl http://127.0.0.1:9090/-/ready
```

Expected:

```text
Prometheus Server is Ready.
```

### Grafana health

```bash
curl http://127.0.0.1:3000/api/health
```

---

# 📁 Project Structure

```text
devops-observability-project-4/
│
├── helm/
│   └── kube-prometheus-stack/
│
├── grafana-smtp-values.yaml
│
├── README.md
│
└── screenshots/
    ├── grafana-dashboard.png
    ├── high-cpu-alert.png
    ├── high-storage-alert.png
    ├── pod-restart-alert.png
    ├── deployment-availability-alert.png
    └── prometheus-target-down.png
```

> Actual repository files should reflect the files committed to the project. Secrets and credentials must never be committed.

---

# 🎯 Learning Outcomes

This project provided hands-on experience with:

* Kubernetes observability
* Prometheus metrics
* Grafana dashboards
* PromQL
* kube-state-metrics
* Node Exporter
* Helm
* Kubernetes alerting
* Grafana-managed alerts
* Alert evaluation and pending periods
* Gmail SMTP integration
* Kubernetes Secrets
* Failure simulation
* Infrastructure monitoring
* Incident-style testing
* SSH port forwarding
* Production-oriented monitoring practices

---

# 🏆 Project Result

The final system provides an end-to-end observability pipeline:

```text
Kubernetes Workloads
        ↓
Metrics Exporters
        ↓
Prometheus
        ↓
Grafana
        ↓
Dashboards + Alert Rules
        ↓
Alert Evaluation
        ↓
Gmail Notifications
```

All five custom alerts were successfully tested, including both **firing and recovery scenarios**.

The Kubernetes application and monitoring stack were also validated after testing, with the application running at **3/3 replicas** and the monitoring components healthy.

---

## 👨‍💻 Author

**Bilal Ahmed**

DevOps Engineer | Cloud & DevSecOps | Kubernetes & CI/CD

GitHub: **Bilaal-Ahmed**

---

## 📌 Portfolio Highlights

**DevOps Project #4 — Kubernetes Observability & Monitoring**

Implemented a production-style Kubernetes monitoring stack using **Prometheus, Grafana, Helm, Node Exporter, and kube-state-metrics**, with custom dashboards, five automated alerts, Gmail notifications, and end-to-end failure testing across CPU, storage, pod restarts, deployment availability, and Prometheus target health.
