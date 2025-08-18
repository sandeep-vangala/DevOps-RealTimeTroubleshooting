
---

# Monitoring & Logging in EKS

## **Prometheus**

* **Role:** Time-series database and monitoring system that collects and stores metrics.
* **In EKS:**

  * Scrapes metrics from nodes, pods, and services using Kubernetes service discovery.
  * Collects CPU/memory usage, pod health, custom app metrics.
* **Key Feature:**

  * Uses **PromQL** for queries.
  * Supports alerting (via Alertmanager).

---

## **Grafana**

* **Role:** Visualization tool for dashboards.
* **In EKS:**

  * Visualizes Prometheus metrics and CloudWatch metrics/logs.
  * Provides real-time dashboards for cluster and app health.
* **Key Feature:**

  * Highly customizable dashboards.
  * Integrates with multiple data sources.

---

## **CloudWatch**

* **Role:** AWS-native monitoring and observability service.
* **In EKS:**

  * Collects cluster metrics (CPU, memory) and app logs (via Fluentd).
  * Provides alarms and insights for AWS resources.
* **Key Feature:**

  * Native AWS integration.
  * Long-term metric/log retention.

---

## **Amazon EKS**

* **Role:** Managed Kubernetes service for containerized applications.
* **Monitoring Needs:**

  * Node/pod health.
  * App performance.
  * Cluster metrics (fed into Prometheus, Grafana, CloudWatch).

---

## **Fluentd**

* **Role:** Log forwarder for EKS pods.
* **Key Feature:**

  * Filters/transforms logs.
  * Forwards logs to CloudWatch.

---

# **How They Work Together**

### **Metrics Collection**

* **Prometheus:** Scrapes EKS metrics (`/metrics` endpoints, pods, nodes, services).
* **CloudWatch:** Collects AWS-specific metrics (EKS nodes, ECR pulls) + app logs (via Fluentd).
* **Interaction:**

  * Prometheus → real-time Kubernetes metrics.
  * CloudWatch → AWS resource metrics + long-term logs.

### **Data Storage**

* **Prometheus:** Local time-series DB (in-cluster, persistent volume).
* **CloudWatch:** Centralized AWS storage with long-term retention.

### **Visualization & Alerting**

* **Grafana:** Queries both Prometheus + CloudWatch → unified dashboards.
* **Prometheus:** Alert rules (via PromQL) → Alertmanager → SNS/email.
* **CloudWatch:** Alarms for AWS metrics → triggers SNS/Auto-Scaling.

### **Fluentd Role**

* Collects pod logs → Filters → Sends to CloudWatch Logs.
* Grafana can query logs via CloudWatch integration.

---

# **EKS Monitoring Architecture**

```
[ EKS Cluster ]
   ├── [ Application Pods ] --> Logs --> [ Fluentd DaemonSet ] --> [ CloudWatch Logs ]
   ├── [ Kubelet/Node Metrics ] --> [ Prometheus Pod ] --> [ Time-Series Storage ]
   ├── [ Control Plane Metrics ] --> [ Prometheus Pod ]
   └── [ Container Insights ] --> [ CloudWatch Metrics ]

[ Prometheus Pod ]
   ├── Scrape Metrics --> [ Time-Series Storage ]
   └── Alerts --> [ Alertmanager ] --> [ SNS/Email ] (optional)

[ Grafana Pod ]
   ├── Query --> [ Prometheus ] (Kubernetes metrics)
   └── Query --> [ CloudWatch ] (AWS metrics, logs)

[ CloudWatch ]
   ├── Metrics (EKS nodes, ECR, etc.)
   ├── Logs (from Fluentd)
   └── Alarms --> [ SNS/Email or Auto-Scaling ]

[ External User ] --> [ Grafana Dashboards ] (view metrics/logs)
```

---

# **Key Flows**

* **Metrics:** EKS pods/nodes → Prometheus → Grafana; EKS nodes → CloudWatch → Grafana.
* **Logs:** EKS pods → Fluentd → CloudWatch Logs → Grafana.
* **Alerts:** Prometheus → Alertmanager; CloudWatch → SNS.

---

# **Detailed Integration Examples**

### **Prometheus in EKS**

* Deployed via **Helm** (e.g., kube-prometheus-stack).
* Uses **ServiceMonitor** to scrape app metrics (`prometheus.io/scrape: true`).
* Example PromQL:

  ```
  rate(container_cpu_usage_seconds_total{namespace="default"}[5m])
  ```

### **Fluentd → CloudWatch**

* Deployed as **DaemonSet** with cloudwatch plugin.
* Example config:

  ```xml
  <match kubernetes.**>
    @type cloudwatch_logs
    log_group_name eks-logs
    log_stream_name fluentd
    region us-east-1
  </match>
  ```

### **Grafana Dashboards**

* Data Sources: Prometheus (K8s metrics), CloudWatch (AWS metrics/logs).
* Example dashboard:

  * Pod CPU/memory (Prometheus).
  * Node utilization (CloudWatch).
  * App logs (CloudWatch Logs Insights).

### **CloudWatch Setup**

* Enable **Container Insights** in EKS.
* Create **alarms** (e.g., node CPU > 80%) → SNS notification.

---

