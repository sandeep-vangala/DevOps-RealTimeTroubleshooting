As an SRE or DevOps engineer with 8 years of experience, troubleshooting issues in an Amazon Elastic Kubernetes Service (EKS) cluster requires a structured approach, leveraging tools like Jenkins, Terraform, AWS, EKS, Prometheus, Grafana, Fluentd, and CloudWatch. Below, I’ll outline real-time troubleshooting scenarios commonly encountered in EKS clusters, detailing the steps an experienced professional would take to diagnose and resolve issues, using the specified tools. Each scenario includes problem identification, investigation, and resolution, grounded in practical, production-like contexts.

---

### Scenario 1: Pods Failing to Start Due to Insufficient Resources
**Context**: An application deployed on an EKS cluster has pods stuck in a `Pending` state. Users report application downtime, and you need to investigate and resolve the issue.

#### Step-by-Step Troubleshooting

1. **Identify the Issue (Using kubectl and CloudWatch)**
   - **Action**: Run `kubectl get pods -n <namespace>` to check the status of pods. Notice multiple pods in `Pending` state.
   - **Action**: Run `kubectl describe pod <pod-name> -n <namespace>` to inspect events. The output indicates: `FailedScheduling: 0/3 nodes are available: 3 Insufficient cpu.`
   - **Action**: Check CloudWatch Container Insights for cluster-level metrics (e.g., CPU/memory utilization). Navigate to CloudWatch → Container Insights → EKS Cluster → `<cluster-name>` to confirm high CPU usage across nodes.[](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/deploy-container-insights-EKS.html)
   - **Observation**: The cluster is under-provisioned, causing the scheduler to fail to place new pods due to insufficient CPU resources.

2. **Investigate Cluster Metrics (Using Prometheus and Grafana)**
   - **Action**: Access the Grafana dashboard configured for the EKS cluster. Query Prometheus for node-level metrics using PromQL, e.g., `node_cpu_seconds_total` or `kube_node_status_allocatable_cpu_cores`.
   - **Action**: Check the `kube-scheduler` logs in CloudWatch (if EKS control plane logging is enabled) to confirm scheduling failures due to resource constraints. Run `aws logs get-log-events --log-group-name /aws/eks/<cluster-name>/cluster --log-stream-name kube-scheduler`.
   - **Observation**: Grafana shows 90%+ CPU utilization on all nodes, and Prometheus metrics confirm low allocatable CPU. This indicates the need for cluster scaling.

3. **Resolve the Issue (Using Terraform and AWS Autoscaler)**
   - **Action**: Since the infrastructure is managed via Terraform, update the Terraform configuration for the EKS node group to increase the desired capacity. For example:
     ```hcl
     resource "aws_eks_node_group" "example" {
       cluster_name    = "<cluster-name>"
       node_group_name = "example-node-group"
       node_role_arn   = "<node-role-arn>"
       subnet_ids      = ["subnet-1", "subnet-2"]
       scaling_config {
         desired_size = 4  # Increase from 3
         max_size     = 6
         min_size     = 2
       }
     }
     ```
   - **Action**: Apply the Terraform changes (`terraform apply`) to scale the node group.
   - **Action**: Verify that the Kubernetes Cluster Autoscaler is enabled (deployed as a pod in the cluster). Check its logs using `kubectl logs -n kube-system cluster-autoscaler` to ensure it’s scaling nodes based on demand.[](https://www.rst.software/blog/practical-guidelines-for-using-kubernetes-on-aws-eks)
   - **Action**: Monitor CloudWatch metrics to confirm new nodes are added and CPU utilization stabilizes.

4. **Validate and Prevent Recurrence**
   - **Action**: Use Grafana to monitor `kube_pod_status_phase` to ensure pods transition to `Running`. Confirm application accessibility.
   - **Action**: Set up Prometheus alerts (via Alertmanager) for high CPU/memory utilization (e.g., `node_cpu_seconds_total > 0.85`) to proactively notify the team via Slack/email.[](https://dev.to/prodevopsguytech/aws-devops-project-advanced-automated-cicd-pipeline-with-infrastructure-as-code-microservices-service-mesh-and-monitoring-3o13)
   - **Action**: Update Terraform to include auto-scaling policies with higher `max_size` to handle future spikes. Document the incident in a runbook for faster resolution.

**Tools Used**: kubectl, CloudWatch (Container Insights, control plane logs), Prometheus, Grafana, Terraform, AWS (EKS node group management).

---

### Scenario 2: Application Latency Spikes Due to Misconfigured Service Mesh
**Context**: Users report slow response times for a microservices-based application running on EKS with Istio as the service mesh. Metrics show increased latency, and you need to pinpoint the root cause.

#### Step-by-Step Troubleshooting

1. **Identify the Issue (Using Grafana and Prometheus)**
   - **Action**: Access Grafana dashboards integrated with Prometheus to monitor Istio metrics (e.g., `istio_requests_total` and `istio_request_duration_seconds`). Notice a spike in P99 latency for a specific service.
   - **Action**: Use PromQL to query detailed metrics, e.g., `histogram_quantile(0.99, sum(rate(istio_request_duration_seconds_bucket[5m])) by (destination_service))`.
   - **Observation**: One microservice (`service-a`) shows consistently high latency (~2s) compared to others (~200ms).

2. **Investigate Logs and Metrics (Using Fluentd and CloudWatch)**
   - **Action**: Check Fluentd-collected logs in CloudWatch Logs for `service-a`. Run a CloudWatch Logs Insights query: `fields @message | filter kubernetes.pod_name like "service-a" | sort @timestamp desc`.
   - **Action**: Inspect Istio sidecar (Envoy) logs for errors like circuit breaker triggers or connection timeouts. Access these via `kubectl logs <service-a-pod> -c istio-proxy -n <namespace>`.
   - **Observation**: Logs show frequent `503 Service Unavailable` errors from Envoy, indicating a misconfigured circuit breaker or resource limits.

3. **Deep Dive into Configuration (Using kubectl and Terraform)**
   - **Action**: Review the Istio `DestinationRule` for `service-a` using `kubectl get destinationrule -n <namespace> -o yaml`. Notice a circuit breaker setting with a low threshold (e.g., `maxConnections: 5`).
   - **Action**: Check the Terraform configuration for Istio resources (if managed via IaC). Confirm the `DestinationRule` was applied incorrectly:
     ```hcl
     resource "kubernetes_manifest" "destination_rule" {
       manifest = {
         apiVersion = "networking.istio.io/v1alpha3"
         kind       = "DestinationRule"
         metadata = {
           name      = "service-a-dr"
           namespace = "<namespace>"
         }
         spec = {
           host = "service-a"
           trafficPolicy = {
             outlierDetection = {
               consecutive5xxErrors = 5
               interval            = "30s"
               baseEjectionTime    = "30s"
             }
           }
         }
       }
     }
     ```
   - **Observation**: The circuit breaker is too aggressive, ejecting pods prematurely, causing request failures and latency spikes.

4. **Resolve the Issue**
   - **Action**: Update the `DestinationRule` via Terraform to relax the circuit breaker settings (e.g., increase `consecutive5xxErrors` to 10 and `interval` to 60s).
   - **Action**: Apply the Terraform changes and monitor Grafana to confirm latency drops to normal levels.
   - **Action**: Set up a Prometheus alert for high Istio latency (e.g., `istio_request_duration_seconds > 1`) to catch similar issues early.[](https://dev.to/prodevopsguytech/aws-devops-project-advanced-automated-cicd-pipeline-with-infrastructure-as-code-microservices-service-mesh-and-monitoring-3o13)
   - **Action**: Validate with a load test using a tool like K6 (triggered via Jenkins pipeline) to ensure the service handles traffic correctly.

5. **Prevent Recurrence**
   - **Action**: Document the circuit breaker tuning process in the team’s runbook. Add a Jenkins pipeline stage to validate Istio configurations before deployment.
   - **Action**: Schedule regular reviews of Istio configurations using `kubectl diff` or Terraform plan outputs to catch misconfigurations.

**Tools Used**: Grafana, Prometheus, Fluentd, CloudWatch, kubectl, Terraform, Jenkins, AWS (EKS, Istio integration).

---

### Scenario 3: CI/CD Pipeline Failure Blocking Deployments
**Context**: A Jenkins pipeline fails to deploy an application to EKS, halting releases. Developers report build failures, and you need to diagnose the issue.

#### Step-by-Step Troubleshooting

1. **Identify the Issue (Using Jenkins and CloudWatch)**
   - **Action**: Check the Jenkins pipeline logs via the Jenkins UI. Notice an error: `Failed to push image to ECR: unauthorized`.
   - **Action**: Inspect CloudWatch Logs for the Jenkins server (running on an EC2 instance) to find related errors. Run a query in CloudWatch Logs Insights: `fields @message | filter @logStream like "jenkins" | sort @timestamp desc`.
   - **Observation**: The Jenkins server lacks permissions to push to Amazon Elastic Container Registry (ECR).

2. **Investigate IAM Permissions (Using AWS Console and Terraform)**
   - **Action**: Check the IAM role attached to the Jenkins EC2 instance using the AWS Console or `aws iam get-role --role-name <jenkins-role>`.
   - **Action**: Review the Terraform configuration for the Jenkins EC2 instance role:
     ```hcl
     resource "aws_iam_role_policy" "jenkins_ecr_policy" {
       name   = "jenkins-ecr-policy"
       role   = aws_iam_role.jenkins_role.id
       policy = jsonencode({
         Version = "2012-10-17"
         Statement = [
           {
             Effect   = "Allow"
             Action   = ["ecr:GetAuthorizationToken"]
             Resource = "*"
           }
           # Missing ECR push/pull permissions
         ]
       })
     }
     ```
   - **Observation**: The IAM policy lacks `ecr:PutImage` and `ecr:BatchGetImage` permissions needed for pushing images to ECR.

3. **Resolve the Issue**
   - **Action**: Update the Terraform policy to include necessary permissions:
     ```hcl
     Statement = [
       {
         Effect   = "Allow"
         Action   = [
           "ecr:GetAuthorizationToken",
           "ecr:PutImage",
           "ecr:BatchGetImage",
           "ecr:InitiateLayerUpload",
           "ecr:UploadLayerPart",
           "ecr:CompleteLayerUpload"
         ]
         Resource = "*"
       }
     ]
     ```
   - **Action**: Run `terraform apply` to update the IAM role. Restart the Jenkins pipeline to confirm the image push succeeds.
   - **Action**: Verify the deployment on EKS using `kubectl get deployments -n <namespace>` and check Grafana dashboards for application metrics to ensure the new version is running.

4. **Prevent Recurrence**
   - **Action**: Add a Jenkins pipeline stage to validate IAM permissions using `aws iam simulate-principal-policy` before deployment.
   - **Action**: Set up CloudWatch alarms for Jenkins pipeline failures (e.g., monitor `JenkinsBuildStatus` metric) to alert the team via SNS.
   - **Action**: Document the IAM policy requirements for Jenkins in the team’s runbook.

**Tools Used**: Jenkins, CloudWatch, AWS (IAM, ECR), Terraform, kubectl, Grafana, Prometheus.

---

### Scenario 4: Log Aggregation Failure Causing Observability Gaps
**Context**: Application logs are not appearing in CloudWatch, impacting troubleshooting. The team relies on Fluentd for log aggregation, and you need to restore logging.

#### Step-by-Step Troubleshooting

1. **Identify the Issue (Using CloudWatch and kubectl)**
   - **Action**: Check CloudWatch Logs for the application’s log group (`/aws/eks/<cluster-name>/application`). Notice no recent logs.
   - **Action**: Run `kubectl get pods -n <namespace> | grep fluentd` to verify Fluentd pods are running. Notice one pod in `CrashLoopBackOff`.
   - **Action**: Run `kubectl describe pod <fluentd-pod> -n <namespace>` to find the error: `failed to connect to CloudWatch: AccessDenied`.

2. **Investigate Fluentd Configuration (Using kubectl and AWS)**
   - **Action**: Check Fluentd pod logs: `kubectl logs <fluentd-pod> -n <namespace>`. Confirm the `AccessDenied` error when pushing logs to CloudWatch.
   - **Action**: Verify the IAM role used by the Fluentd service account (via IRSA). Run `aws iam get-role --role-name <fluentd-role>`. Notice missing `logs:PutLogEvents` permission.
   - **Observation**: The Fluentd pod’s IAM role lacks permissions to write to CloudWatch Logs.

3. **Resolve the Issue (Using Terraform and AWS)**
   - **Action**: Update the Terraform configuration for the Fluentd IAM role:
     ```hcl
     resource "aws_iam_role_policy" "fluentd_cloudwatch" {
       name   = "fluentd-cloudwatch-policy"
       role   = aws_iam_role.fluentd_role.id
       policy = jsonencode({
         Version = "2012-10-17"
         Statement = [
           {
             Effect   = "Allow"
             Action   = [
               "logs:CreateLogStream",
               "logs:PutLogEvents",
               "logs:DescribeLogGroups"
             ]
             Resource = "arn:aws:logs:*:*:log-group:/aws/eks/*:*"
           }
         ]
       })
     }
     ```
   - **Action**: Apply the Terraform changes and restart the Fluentd pod: `kubectl delete pod <fluentd-pod> -n <namespace>`.
   - **Action**: Monitor CloudWatch Logs to confirm logs are flowing again.

4. **Validate and Prevent Recurrence**
   - **Action**: Use Grafana to monitor Fluentd metrics (e.g., `fluentd_output_status_buffer_queue_length`) to ensure logs are processed correctly.[](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/deploy-container-insights-EKS.html)
   - **Action**: Set up a Prometheus alert for Fluentd pod failures (e.g., `kube_pod_status_phase{phase="CrashLoopBackOff"} > 0`).
   - **Action**: Add a Jenkins pipeline stage to validate Fluentd configurations and IAM permissions before deployment.

Below, I’ll provide additional real-time troubleshooting scenarios for failures in an Amazon Elastic Kubernetes Service (EKS) cluster where an application is deployed. These scenarios are tailored for an experienced SRE/DevOps engineer with 8 years of experience, focusing on practical, production-like issues. I’ll leverage the specified tools—Jenkins, Terraform, AWS, EKS, Prometheus, Grafana, Fluentd, and CloudWatch—and provide detailed steps for identifying, investigating, and resolving each issue, along with preventive measures. Each scenario builds on common failure modes in EKS clusters and assumes a microservices-based application.

---

### Scenario 5: Application Crash Due to Misconfigured ConfigMap
**Context**: A critical microservice in the EKS cluster is repeatedly crashing, causing partial application outages. Developers report that recent configuration changes were deployed via a Jenkins pipeline, and you need to diagnose the root cause.

#### Step-by-Step Troubleshooting

1. **Identify the Issue (Using kubectl and CloudWatch)**
   - **Action**: Run `kubectl get pods -n <namespace>` to check pod statuses. Notice that pods for `service-b` are in `CrashLoopBackOff`.
   - **Action**: Run `kubectl describe pod <service-b-pod> -n <namespace>` to inspect events. The output shows: `Back-off restarting failed container`.
   - **Action**: Check pod logs using `kubectl logs <service-b-pod> -n <namespace>`. The logs indicate: `Error: Invalid configuration key 'API_ENDPOINT'`.
   - **Action**: Check CloudWatch Logs for detailed application logs. Run a CloudWatch Logs Insights query: `fields @message | filter kubernetes.pod_name like "service-b" | sort @timestamp desc`. Confirm the same configuration error.
   - **Observation**: The application is failing due to a misconfigured or missing `ConfigMap` value for `API_ENDPOINT`.

2. **Investigate Configuration (Using kubectl and Jenkins)**
   - **Action**: Inspect the `ConfigMap` used by `service-b` with `kubectl get configmap <configmap-name> -n <namespace> -o yaml`. Notice that `API_ENDPOINT` is set to an invalid URL (e.g., `http://invalid-endpoint`).
   - **Action**: Check the Jenkins pipeline that deployed the latest changes. Review the pipeline logs in the Jenkins UI to find the stage that applied the `ConfigMap`. Identify the commit or configuration change that introduced the invalid value.
   - **Observation**: The `ConfigMap` was updated in the latest deployment with an incorrect endpoint due to a manual override in the Jenkins pipeline parameters.

3. **Resolve the Issue (Using Terraform and kubectl)**
   - **Action**: Update the `ConfigMap` definition in the Terraform configuration (assuming IaC management):
     ```hcl
     resource "kubernetes_config_map" "service_b_config" {
       metadata {
         name      = "service-b-config"
         namespace = "<namespace>"
       }
       data = {
         API_ENDPOINT = "http://correct-endpoint:8080"
       }
     }
     ```
   - **Action**: Run `terraform apply` to update the `ConfigMap`. Alternatively, apply the corrected `ConfigMap` directly using `kubectl apply -f configmap.yaml`.
   - **Action**: Restart the affected pods: `kubectl delete pod -l app=service-b -n <namespace>`.
   - **Action**: Monitor pod status with `kubectl get pods -n <namespace>` to confirm pods are in `Running` state. Check application logs in CloudWatch to verify no further errors.

4. **Validate and Prevent Recurrence**
   - **Action**: Use Grafana to monitor application metrics (e.g., `kube_pod_status_phase` from Prometheus) to ensure all `service-b` pods are stable.
   - **Action**: Set up a Prometheus alert for pod crash loops (e.g., `kube_pod_status_phase{phase="CrashLoopBackOff"} > 0`) to notify the team via Slack.
   - **Action**: Add a validation step in the Jenkins pipeline to lint `ConfigMap` files using a tool like `kubeval` before applying them.
   - **Action**: Document the issue and resolution in the team’s runbook, emphasizing validation of configuration changes.

**Tools Used**: kubectl, CloudWatch, Jenkins, Terraform, Prometheus, Grafana.

---

### Scenario 6: Network Policy Blocking Inter-Service Communication
**Context**: Two microservices (`service-c` and `service-d`) in the EKS cluster cannot communicate, causing application errors. Logs show connection timeouts, and you need to troubleshoot the network issue.

#### Step-by-Step Troubleshooting

1. **Identify the Issue (Using kubectl and Fluentd/CloudWatch)**
   - **Action**: Check `service-c` pod logs with `kubectl logs <service-c-pod> -n <namespace>`. Notice errors: `Connection timed out to service-d:8080`.
   - **Action**: Query CloudWatch Logs (populated by Fluentd) for `service-c`: `fields @message | filter kubernetes.pod_name like "service-c" | sort @timestamp desc`. Confirm timeout errors when calling `service-d`.
   - **Action**: Test connectivity manually using `kubectl exec -it <service-c-pod> -n <namespace> -- curl service-d:8080`. The request fails with a timeout.
   - **Observation**: A network issue is preventing communication between `service-c` and `service-d`.

2. **Investigate Network Policies (Using kubectl and Prometheus)**
   - **Action**: Check for Kubernetes Network Policies in the namespace: `kubectl get networkpolicy -n <namespace>`. Find a policy restricting traffic to `service-d`.
   - **Action**: Inspect the policy with `kubectl describe networkpolicy <policy-name> -n <namespace>`. The policy allows traffic only from pods with specific labels, but `service-c` lacks the required label (e.g., `app: allowed-client`).
   - **Action**: Use Prometheus and Grafana to check network metrics (e.g., `istio_tcp_received_bytes_total` if using Istio, or `kube_pod_network_errors` if available). Confirm zero traffic to `service-d` from `service-c`.
   - **Observation**: The Network Policy is overly restrictive, blocking legitimate traffic.

3. **Resolve the Issue (Using Terraform and kubectl)**
   - **Action**: Update the Network Policy via Terraform to allow traffic from `service-c`:
     ```hcl
     resource "kubernetes_network_policy" "service_d_access" {
       metadata {
         name      = "service-d-access"
         namespace = "<namespace>"
       }
       spec {
         pod_selector {
           match_labels = {
             app = "service-d"
           }
         }
         ingress {
           from {
             pod_selector {
               match_labels = {
                 app = "service-c"
               }
             }
           }
           ports {
             protocol = "TCP"
             port     = 8080
           }
         }
         policy_types = ["Ingress"]
       }
     }
     ```
   - **Action**: Run `terraform apply` to update the policy. Verify with `kubectl describe networkpolicy service-d-access -n <namespace>`.
   - **Action**: Retest connectivity using `kubectl exec -it <service-c-pod> -n <namespace> -- curl service-d:8080`. Confirm the request succeeds.
   - **Action**: Monitor Grafana dashboards to ensure traffic is flowing between services.

4. **Prevent Recurrence**
   - **Action**: Set up a Prometheus alert for network errors (e.g., `kube_pod_network_errors > 0`) to detect similar issues.
   - **Action**: Add a Jenkins pipeline stage to validate Network Policies using `kubectl diff` or a tool like `netpol` before deployment.
   - **Action**: Document Network Policy best practices in the runbook, including label-based access controls.

**Tools Used**: kubectl, CloudWatch, Fluentd, Prometheus, Grafana, Terraform, Jenkins.

---

### Scenario 7: Node Failure Causing Application Downtime
**Context**: An EKS worker node becomes unresponsive, causing pods to be evicted and application downtime. Alerts from Prometheus indicate node failures, and you need to restore service.

#### Step-by-Step Troubleshooting

1. **Identify the Issue (Using Prometheus and Grafana)**
   - **Action**: Receive a Prometheus alert via Alertmanager: `node_unreachable > 0`. Access Grafana to view the `node_exporter` dashboard, specifically `up` metric for nodes. Notice one node with `up=0`.
   - **Action**: Run `kubectl get nodes` to check node status. One node is in `NotReady` state.
   - **Action**: Check CloudWatch Container Insights for node-level metrics (e.g., `node_cpu_utilization`, `node_memory_utilization`). Confirm the node is offline.
   - **Observation**: A worker node has failed, likely due to hardware issues or an EC2 instance crash.

2. **Investigate Node Failure (Using AWS Console and CloudWatch)**
   - **Action**: In the AWS Console, navigate to EC2 → Instances and find the instance corresponding to the `NotReady` node (check the node’s `providerID` in `kubectl describe node <node-name>`).
   - **Action**: Check EC2 instance health status. Notice the instance failed system checks (e.g., hardware failure).
   - **Action**: Inspect CloudWatch Logs for the EKS control plane (`/aws/eks/<cluster-name>/cluster`, stream `kubelet`). Find errors indicating the node lost connectivity to the API server.
   - **Observation**: The node failure is due to an EC2 instance crash, requiring replacement.

3. **Resolve the Issue (Using Terraform and AWS Autoscaler)**
   - **Action**: Since the Cluster Autoscaler is deployed, verify it’s attempting to replace the failed node. Check logs: `kubectl logs -n kube-system cluster-autoscaler`.
   - **Action**: If the autoscaler doesn’t replace the node (e.g., due to taints or scheduling constraints), manually terminate the failed EC2 instance in the AWS Console to trigger a replacement.
   - **Action**: Update the Terraform configuration to add a taint to prevent critical pods from scheduling on unstable nodes:
     ```hcl
     resource "aws_eks_node_group" "example" {
       cluster_name    = "<cluster-name>"
       node_group_name = "example-node-group"
       node_role_arn   = "<node-role-arn>"
       subnet_ids      = ["subnet-1", "subnet-2"]
       taint {
         key    = "node.critical"
         value  = "true"
         effect = "NO_SCHEDULE"
       }
       scaling_config {
         desired_size = 3
         max_size     = 5
         min_size     = 2
       }
     }
     ```
   - **Action**: Run `terraform apply` to apply the taint. Monitor `kubectl get nodes` to confirm the new node is added and in `Ready` state.

4. **Validate and Prevent Recurrence**
   - **Action**: Use Grafana to monitor `kube_node_status_condition{condition="Ready"}` to ensure all nodes are stable.
   - **Action**: Set up a Prometheus alert for node failures (e.g., `up{job="node_exporter"} == 0`) to detect issues proactively.
   - **Action**: Configure CloudWatch alarms for EC2 instance health checks to alert on system failures.
   - **Action**: Document node failure recovery steps in the runbook, including manual EC2 termination and autoscaler tuning.

**Tools Used**: Prometheus, Grafana, kubectl, CloudWatch, AWS (EC2, EKS), Terraform.

---

### Scenario 8: DNS Resolution Failures Causing Service Disruptions
**Context**: Pods in the EKS cluster cannot resolve DNS queries, causing failures in external API calls. Users report intermittent application errors, and you need to troubleshoot DNS issues.

#### Step-by-Step Troubleshooting

1. **Identify the Issue (Using kubectl and CloudWatch)**
   - **Action**: Check application logs with `kubectl logs <pod-name> -n <namespace>`. Notice errors: `getaddrinfo ENOTFOUND external-api.com`.
   - **Action**: Query CloudWatch Logs for DNS-related errors: `fields @message | filter kubernetes.pod_name like "<pod-name>" | sort @timestamp desc`. Confirm DNS resolution failures.
   - **Action**: Test DNS resolution from a pod: `kubectl exec -it <pod-name> -n <namespace> -- nslookup external-api.com`. The query fails with a timeout.
   - **Observation**: The cluster’s DNS service (CoreDNS) is malfunctioning, preventing pods from resolving external domains.

2. **Investigate CoreDNS (Using kubectl, Prometheus, and Grafana)**
   - **Action**: Check CoreDNS pod status: `kubectl get pods -n kube-system | grep coredns`. Notice one CoreDNS pod in `Running` but with high restart counts.
   - **Action**: Check CoreDNS logs: `kubectl logs -n kube-system <coredns-pod>`. Find errors: `upstream server timeout`.
   - **Action**: Use Grafana to monitor CoreDNS metrics from Prometheus (e.g., `coredns_dns_request_duration_seconds`). Notice high latency and dropped queries.
   - **Action**: Check CloudWatch Container Insights for CoreDNS resource usage (`pod_cpu_utilization`, `pod_memory_utilization`). Confirm high memory usage on CoreDNS pods.
   - **Observation**: CoreDNS is overloaded, likely due to insufficient resources or misconfiguration.

3. **Resolve the Issue (Using Terraform and kubectl)**
   - **Action**: Update the CoreDNS deployment to increase replicas and resources via Terraform:
     ```hcl
     resource "kubernetes_deployment" "coredns" {
       metadata {
         name      = "coredns"
         namespace = "kube-system"
       }
       spec {
         replicas = 3  # Increase from 2
         selector {
           match_labels = {
             k8s-app = "kube-dns"
           }
         }
         template {
           spec {
             containers {
               name  = "coredns"
               image = "coredns/coredns:latest"
               resources {
                 limits {
                   cpu    = "200m"
                   memory = "256Mi"
                 }
                 requests {
                   cpu    = "100m"
                   memory = "128Mi"
                 }
               }
             }
           }
         }
       }
     }
     ```
   - **Action**: Run `terraform apply` to scale CoreDNS. Verify with `kubectl get pods -n kube-system`.
   - **Action**: Test DNS resolution again: `kubectl exec -it <pod-name> -n <namespace> -- nslookup external-api.com`. Confirm it succeeds.
   - **Action**: Monitor Grafana to ensure CoreDNS request latency returns to normal.

4. **Prevent Recurrence**
   - **Action**: Set up a Prometheus alert for CoreDNS failures (e.g., `coredns_dns_request_failures_total > 0`).
   - **Action**: Configure CloudWatch alarms for CoreDNS pod resource usage to detect overloads.
   - **Action**: Add a Jenkins pipeline stage to validate CoreDNS configurations before upgrades.
   - **Action**: Document CoreDNS troubleshooting steps in the runbook, including resource tuning and upstream DNS server checks.

**Tools Used**: kubectl, CloudWatch, Fluentd, Prometheus, Grafana, Terraform, Jenkins.

---

### Best Practices for EKS Troubleshooting
- **Observability**: Use Prometheus and Grafana for comprehensive monitoring of cluster, application, and network metrics. Set up alerts for critical failure modes (e.g., pod crashes, node failures, DNS issues).
- **Logging**: Ensure Fluentd is configured to send all pod and control plane logs to CloudWatch for centralized analysis.
- **Infrastructure as Code**: Use Terraform to manage EKS resources, Network Policies, and configurations to ensure consistency and auditability.
- **CI/CD Validation**: Leverage Jenkins to automate deployment pipelines with validation steps (e.g., linting `ConfigMaps`, Network Policies, and CoreDNS configs).
- **Incident Response**: Maintain detailed runbooks for common issues like DNS failures, node crashes, and configuration errors to streamline resolution.
- **Proactive Scaling**: Use Cluster Autoscaler and Horizontal Pod Autoscaler (HPA) to handle resource demands dynamically.
- **Security**: Regularly audit IAM roles and Network Policies to prevent overly restrictive or permissive configurations.

### Final Notes
These scenarios cover a range of EKS failure modes, from configuration errors to network and node issues, reflecting challenges faced in production environments. By systematically using kubectl for cluster inspection, Prometheus and Grafana for monitoring, Fluentd and CloudWatch for logging, Terraform for infrastructure changes, and Jenkins for deployment automation, you can effectively troubleshoot and resolve issues. Always validate fixes with metrics and logs, and implement alerts and documentation to prevent recurrence.

If you need more specific scenarios, deeper dives into any issue, or assistance with a particular tool, let me know!

**Tools Used**: CloudWatch, kubectl, Fluentd, Terraform, AWS (IAM), Grafana, Prometheus, Jenkins.

---

### Best Practices for SRE/DevOps in EKS Troubleshooting
- **Proactive Monitoring**: Use Prometheus and Grafana to set up dashboards for key metrics (e.g., pod status, node utilization, Istio latency). Configure alerts to catch issues early.[](https://medium.com/%40prasad.midde3/monitoring-kubernetes-with-prometheus-and-grafana-on-amazon-eks-18f2371f2597)[](https://bluelight.co/blog/aws-eks-monitoring)
- **Log Centralization**: Ensure Fluentd or Fluent Bit is correctly configured to send logs to CloudWatch for centralized observability.[](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/deploy-container-insights-EKS.html)
- **Infrastructure as Code**: Use Terraform to manage EKS resources and IAM policies consistently, reducing misconfigurations.[](https://www.rst.software/blog/practical-guidelines-for-using-kubernetes-on-aws-eks)
- **CI/CD Automation**: Leverage Jenkins to automate deployments and validate configurations, minimizing human errors.[](https://dev.to/prodevopsguytech/aws-devops-project-advanced-automated-cicd-pipeline-with-infrastructure-as-code-microservices-service-mesh-and-monitoring-3o13)
- **Documentation**: Maintain detailed runbooks for common issues (e.g., resource shortages, IAM errors) to speed up resolution.
- **Security**: Use IAM Roles for Service Accounts (IRSA) to securely grant permissions to pods like Fluentd or Jenkins.[](https://grafana.com/docs/grafana-cloud/monitor-infrastructure/kubernetes-monitoring/configuration/config-other-methods/config-aws-eks/)
- **Backup and Recovery**: Implement tools like Velero for EKS cluster backups to recover from catastrophic failures.[](https://www.rst.software/blog/practical-guidelines-for-using-kubernetes-on-aws-eks)

### Final Notes
These scenarios reflect real-world challenges faced by SREs/DevOps engineers managing EKS clusters. By combining kubectl for cluster interaction, Prometheus and Grafana for monitoring, Fluentd and CloudWatch for logging, Terraform for infrastructure management, and Jenkins for CI/CD, you can systematically diagnose and resolve issues. Always validate fixes with metrics and logs, and implement preventive measures to reduce recurrence. For further details on monitoring setups, refer to AWS documentation or guides like those on setting up Prometheus and Grafana with EKS.[](https://medium.com/%40prasad.midde3/monitoring-kubernetes-with-prometheus-and-grafana-on-amazon-eks-18f2371f2597)[](https://bluelight.co/blog/aws-eks-monitoring)
