Prometheus:
Role: A time-series database and monitoring system that collects and stores metrics from applications and infrastructure.
In EKS: Scrapes metrics from EKS nodes, pods, and services using Kubernetes service discovery. It collects metrics like CPU/memory usage, pod health, and custom application metrics.
Key Feature: Uses PromQL for querying metrics and supports alerting.
Grafana:
Role: A visualization tool that creates dashboards from data sources like Prometheus and CloudWatch.
In EKS: Visualizes Prometheus metrics and CloudWatch metrics/logs in dashboards for real-time monitoring of EKS clusters and applications.
Key Feature: Customizable dashboards and integration with multiple data sources.
CloudWatch:
Role: AWS’s monitoring and observability service, collecting logs, metrics, and events from AWS resources and applications.
In EKS: Collects EKS cluster metrics (e.g., node CPU/memory) and application logs forwarded via Fluentd. Provides alarms and insights for AWS-specific resources.
Key Feature: Native AWS integration and log aggregation.
Amazon EKS:
Role: A managed Kubernetes service that runs containerized applications.
Monitoring Needs: Tracks node/pod health, application performance, and cluster metrics, which are fed into Prometheus, Grafana, and CloudWatch.
Fluentd (mentioned in your pipeline):
Role: A log forwarder that collects logs from EKS pods and sends them to CloudWatch for storage and analysis.
Key Feature: Filters and transforms logs before forwarding.
How They Work Hand in Hand
Metrics Collection:
Prometheus: Scrapes metrics from EKS components (nodes, pods, services) via endpoints like /metrics. It uses Kubernetes service discovery to find targets dynamically.
CloudWatch: Collects AWS-specific metrics (e.g., EKS node metrics, ECR pull times) and logs from applications via Fluentd.
Interaction: Prometheus focuses on fine-grained, real-time Kubernetes metrics, while CloudWatch captures broader AWS resource metrics and logs.
Data Storage:
Prometheus: Stores time-series metrics locally or in a persistent volume in EKS.
CloudWatch: Stores metrics and logs in AWS, with long-term retention and querying via CloudWatch Logs Insights.
Interaction: Prometheus is optimized for short-term, high-resolution metrics, while CloudWatch handles long-term storage and AWS-native data.
Visualization and Alerting:
Grafana: Queries Prometheus for Kubernetes metrics and CloudWatch for AWS metrics/logs, displaying them in unified dashboards.
Prometheus: Generates alerts based on PromQL rules, which can be sent to tools like Alertmanager or integrated with AWS SNS via custom setups.
CloudWatch: Creates alarms for AWS metrics (e.g., high CPU usage) and triggers actions like notifications or auto-scaling.
Interaction: Grafana acts as the central visualization layer, pulling data from both Prometheus and CloudWatch. Prometheus and CloudWatch handle alerting independently but can be integrated for unified notifications.
Fluentd’s Role:
Collects logs from EKS pods, filters them, and forwards them to CloudWatch Logs. Grafana can then query CloudWatch Logs for visualization.
Architecture Description
The architecture involves EKS running applications, with Prometheus, Grafana, and Fluentd deployed as pods or services within the cluster. CloudWatch operates as an AWS service outside the cluster, receiving data from EKS and Fluentd. Here’s a step-by-step breakdown of the data flow:

EKS Cluster:
Runs application pods, Kubernetes control plane, and nodes.
Exposes metrics via /metrics endpoints (e.g., kubelet, application metrics).
Generates logs from pods, captured by Fluentd.
Prometheus:
Deployed as a pod in EKS (often via the Prometheus Operator or Helm chart).
Uses Kubernetes service discovery to scrape metrics from nodes, pods, and services.
Stores metrics in a time-series database (e.g., on a persistent volume).
Sends alerts to Alertmanager or external services if thresholds are breached.
Fluentd:
Deployed as a DaemonSet in EKS to collect logs from all pods.
Filters and forwards logs to CloudWatch Logs using the AWS FireLens integration or Fluentd’s CloudWatch plugin.
CloudWatch:
Receives EKS cluster metrics (e.g., node CPU/memory) via the CloudWatch Agent or Container Insights.
Stores logs from Fluentd in CloudWatch Logs.
Provides metrics and logs for querying and alarming.
Grafana:
Deployed as a pod in EKS or externally (e.g., on EC2).
Configured with Prometheus and CloudWatch as data sources.
Queries Prometheus for Kubernetes metrics and CloudWatch for AWS metrics/logs.
Displays dashboards for cluster health, application performance, and logs.




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


Key Flows:

Metrics: EKS pods/nodes → Prometheus → Grafana; EKS nodes → CloudWatch → Grafana.
Logs: EKS pods → Fluentd → CloudWatch Logs → Grafana.
Alerts: Prometheus → Alertmanager; CloudWatch → SNS.


Detailed Integration Example
Prometheus in EKS:
Deployed via Helm (e.g., kube-prometheus-stack chart).
Configured with ServiceMonitor to scrape application metrics (e.g., prometheus.io/scrape: true annotations on pods).
Example PromQL query: rate(container_cpu_usage_seconds_total{namespace="default"}[5m]).
Fluentd to CloudWatch:
Deployed as a DaemonSet with the fluentd-cloudwatch plugin.
Configured to match pod logs (e.g., kubernetes.namespace_name: default) and forward to a CloudWatch log group.
Example Fluentd config:
text


<match kubernetes.**>
  @type cloudwatch_logs
  log_group_name eks-logs
  log_stream_name fluentd
  region us-east-1
</match>
Grafana Dashboards:
Data sources: Prometheus (for Kubernetes metrics) and CloudWatch (for AWS metrics/logs).
Example dashboard: Shows pod CPU/memory (Prometheus), EKS node utilization (CloudWatch), and application logs (CloudWatch Logs Insights).
CloudWatch Setup:
Enable Container Insights on EKS to collect cluster metrics.
Create alarms (e.g., CPUUtilization > 80% for nodes) to trigger SNS notifications.
