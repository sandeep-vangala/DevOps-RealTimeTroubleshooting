1. Scenario: Broken Jenkins Pipeline
Question: Your team reports that the Jenkins pipeline, which builds a Docker image from a GitHub repository and deploys it to an EKS cluster, is failing at the Docker build stage with an error: "Cannot connect to the Docker daemon." How would you troubleshoot and resolve this issue?


Verify Docker Daemon: First, I’d log into the Jenkins agent (or node) where the pipeline runs and check if the Docker daemon is running using sudo systemctl status docker or docker info. If it’s not running, I’d start it with sudo systemctl start docker.
Check Permissions: The Jenkins user might lack permissions to access the Docker daemon. I’d verify if the Jenkins user is in the docker group using groups jenkins. If not, I’d add it with sudo usermod -aG docker jenkins and restart the Jenkins service.
Jenkins Configuration: If the agent is a Docker container itself, I’d ensure the Docker socket (/var/run/docker.sock) is mounted into the Jenkins container. The Docker daemon might not be accessible otherwise.
Pipeline Script: I’d review the Jenkinsfile to ensure the Docker build command is correct (e.g., docker build -t image-name .). I’d also check if the Dockerfile is in the correct directory relative to the workspace.
Network Issues: If the Jenkins agent is on a remote machine, I’d verify network connectivity to the Docker host and ensure no firewall rules block the connection.
Logs: I’d check Jenkins build logs for additional context and test the Docker build command manually on the agent to replicate the issue.
Resolution: After identifying the root cause (e.g., stopped daemon, permissions, or misconfiguration), I’d apply the fix, test the pipeline, and document the issue to prevent recurrence.
Thought Process: This question tests your ability to troubleshoot systematically. The Docker daemon error is common in Jenkins setups and often stems from permissions, configuration, or service issues. The answer demonstrates a structured approach: checking services, permissions, configuration, and logs.

2. Scenario: Slow Deployment to EKS
Question: Your team notices that deployments to the EKS cluster using Helm charts are taking longer than expected, causing delays in the CI/CD pipeline. How would you diagnose and optimize the deployment process?


Check Helm Chart: I’d review the Helm chart to ensure it’s optimized. For example, I’d check if resource requests/limits are set appropriately to avoid over-provisioning or under-provisioning, which can slow down pod scheduling.
EKS Cluster Health: Using kubectl get nodes and kubectl describe node, I’d verify if the EKS nodes have sufficient CPU/memory and aren’t overloaded. I’d also check for pod evictions or scheduling delays using kubectl get events.
Horizontal Pod Autoscaling (HPA): If HPA is configured, I’d ensure it’s not over-scaling unnecessarily, causing delays. I’d verify metrics with kubectl describe hpa and check Prometheus/Grafana dashboards for resource usage spikes.
Docker Image Size: Large Docker images can slow down pulls. I’d use docker images to check the image size and optimize the Dockerfile (e.g., use multi-stage builds, remove unnecessary layers, or use a smaller base image like alpine).
Network Latency: I’d check for network bottlenecks between the Jenkins server, container registry (e.g., ECR), and EKS cluster. Tools like traceroute or CloudWatch metrics for ECR pull times could help identify issues.
Helm Optimization: I’d ensure Helm is using --atomic or --wait flags to avoid race conditions and check for unnecessary dependencies in the chart.
Monitoring Insights: I’d use Grafana dashboards to monitor deployment times and CloudWatch logs (via Fluentd) to identify errors or delays in pod startup.
Resolution: Based on findings, I’d optimize the Dockerfile, adjust resource limits, scale EKS nodes if needed, or tweak Helm configurations. I’d test the pipeline and monitor deployment times post-fix.
Thought Process: This tests your ability to optimize a Kubernetes-based deployment. The answer covers multiple layers (Helm, Kubernetes, Docker, and monitoring) and shows how to use tools like Prometheus, Grafana, and CloudWatch to diagnose performance issues.

3. Scenario: Terraform Drift in EKS Infrastructure
Question: Your team uses Terraform to manage EKS infrastructure, but someone manually updated the EKS cluster’s node group size via the AWS Console, causing a drift. How would you detect and resolve this drift?


Detect Drift: I’d run terraform plan against the Terraform state to identify discrepancies between the desired state (defined in Terraform files) and the actual state in AWS. The output would show the node group size mismatch.
Review State: I’d verify the Terraform state file using terraform state list and terraform state show to confirm the expected configuration for the EKS node group.
Resolve Drift: To align the infrastructure with Terraform:
Option 1: Update Infrastructure: Run terraform apply to revert the node group size to the Terraform-defined value. This is ideal if the manual change was unauthorized or incorrect.
Option 2: Update Terraform Code: If the manual change is desired, I’d update the Terraform code (e.g., modify the desired_size in the aws_eks_node_group resource) to match the current state, then run terraform apply to reconcile.
Prevent Future Drift: I’d implement preventative measures, such as:
Using IAM policies to restrict manual changes to EKS resources.
Setting up a CI/CD pipeline with Terraform to enforce infrastructure-as-code (IaC) practices.
Using terraform lock or state file protection in a backend like S3 with DynamoDB for consistency.
Monitoring: I’d configure CloudWatch alarms to detect unexpected changes in EKS resources and integrate them with Grafana for visualization.
Thought Process: This question assesses your understanding of Terraform’s state management and drift resolution. The answer shows a methodical approach to detecting drift, resolving it, and preventing future issues while integrating monitoring tools.

4. Scenario: Monitoring Gaps in Prometheus/Grafana
Question: Your team uses Prometheus and Grafana to monitor an application running on EKS, but you notice that some critical metrics (e.g., application-specific custom metrics) are missing in Grafana dashboards. How would you investigate and fix this?


Verify Prometheus Scraping: I’d check the Prometheus configuration (prometheus.yml) to ensure the application’s metrics endpoint is included in the scrape config. For example, I’d confirm the correct metrics_path and targets are set for the application’s service.
Check Application Metrics: I’d verify that the application exposes metrics in a Prometheus-compatible format (e.g., /metrics endpoint). Using kubectl port-forward, I’d access the application locally and curl the metrics endpoint to confirm it’s working.
Service Discovery in EKS: Since the application runs on EKS, I’d ensure Prometheus uses Kubernetes service discovery to find the application’s pods. I’d check if the correct role (e.g., pod, service) and labels are configured in the scrape config.
Prometheus Operator/Helm: If using the Prometheus Operator or Helm chart, I’d verify the ServiceMonitor or PodMonitor resources are correctly defined and match the application’s labels.
Grafana Dashboards: I’d check the Grafana dashboard configuration to ensure it queries the correct Prometheus metrics. I’d use the Grafana Explore feature to test PromQL queries and validate data availability.
Fluentd/CloudWatch Integration: If metrics are logged via Fluentd to CloudWatch, I’d verify Fluentd’s configuration to ensure it’s forwarding relevant logs and that CloudWatch metric filters are set up to extract custom metrics.
Resolution: I’d update the Prometheus scrape config or ServiceMonitor, ensure the application exposes metrics correctly, and update Grafana dashboards. I’d test by redeploying the application and monitoring the dashboard.
Thought Process: This tests your ability to troubleshoot monitoring setups. The answer covers the Prometheus/Grafana stack, Kubernetes service discovery, and log forwarding with Fluentd, showing a comprehensive approach to diagnosing and fixing monitoring gaps.

5. Scenario: Security Issue in Docker Image
Question: During a security audit, you find that a Docker image built in your Jenkins pipeline and deployed to EKS contains a critical vulnerability in a base image. How would you address this issue and prevent it in the future?


Identify Vulnerability: I’d use a tool like Trivy or Clair to scan the Docker image and identify the specific vulnerability (e.g., a CVE in the base image). I’d check the scan report for details on the affected package and version.
Update Base Image: I’d update the Dockerfile to use a patched version of the base image (e.g., node:18.16.0 to node:18.16.1) or switch to a more secure base image like node:18.16.0-alpine if feasible.
Rebuild and Redeploy: I’d trigger the Jenkins pipeline to rebuild the Docker image, push it to the container registry (e.g., ECR), and update the Helm chart or Kubernetes manifest to use the new image. I’d then deploy it to EKS using helm upgrade or kubectl apply.
Verify Deployment: I’d use kubectl get pods and kubectl describe pod to ensure the new image is running. I’d also re-run the security scan to confirm the vulnerability is resolved.
Prevent Future Issues:
Integrate a security scanning tool (e.g., Trivy) into the Jenkins pipeline to scan images during the build stage and fail the build if critical vulnerabilities are found.
Use Dependabot or RenovateBot in GitHub to automatically update base images and dependencies.
Enforce image signing with tools like Cosign to ensure only trusted images are deployed.
Configure CloudWatch alarms to alert on security scan failures or outdated images.
Monitoring: I’d set up Prometheus metrics to track image vulnerabilities (if supported by the scanner) and visualize them in Grafana for ongoing monitoring.
Thought Process: This question evaluates your approach to security in a DevOps pipeline. The answer demonstrates immediate remediation (patching the image) and proactive measures (automated scanning, dependency management) while integrating monitoring tools.

6. Scenario: Log Aggregation Failure with Fluentd
Question: Your team notices that logs from an application running on EKS are not appearing in CloudWatch, despite Fluentd being configured for log forwarding. How would you troubleshoot and resolve this?


Check Fluentd Pods: I’d use kubectl get pods -n <namespace> to verify that Fluentd pods are running in the EKS cluster. If they’re not, I’d check pod logs with kubectl logs to identify crashes or errors.
Fluentd Configuration: I’d review the Fluentd configuration (e.g., ConfigMap or Helm values) to ensure it’s set up to collect logs from the application’s pods. I’d verify the match and filter directives point to the correct log sources and CloudWatch destination.
Pod Labels: I’d check if the application pods have the correct labels or annotations that Fluentd uses for log collection (e.g., Kubernetes metadata filters).
IAM Permissions: I’d ensure the IAM role associated with the EKS nodes or Fluentd pods has permissions to write to CloudWatch Logs (logs:PutLogEvents, logs:CreateLogStream).
Network Connectivity: I’d verify that Fluentd can reach the CloudWatch Logs endpoint by checking network policies, VPC configurations, and security groups.
CloudWatch Logs: I’d check CloudWatch Logs for any partial logs or errors and use CloudWatch Insights to query for Fluentd-related errors.
Test Locally: Using kubectl port-forward, I’d forward logs from a pod to test if Fluentd processes them correctly.
Resolution: Depending on the issue, I’d fix the Fluentd configuration, update IAM roles, or adjust pod labels. I’d redeploy Fluentd via Helm and verify logs in CloudWatch.
Thought Process: This question tests your ability to troubleshoot log aggregation in a Kubernetes environment. The answer covers Fluentd configuration, Kubernetes integration, IAM permissions, and CloudWatch, showing a systematic debugging process.

7. Scenario: Scaling Issues in EKS
Question: Your application on EKS is experiencing performance issues during peak traffic, and pods are not scaling as expected. How would you diagnose and resolve this using your monitoring tools (Prometheus, Grafana, CloudWatch)?


**Check HPA Configuration:** I’d verify the Horizontal Pod Autoscaler (HPA) configuration using kubectl describe hpa. I’d ensure it’s targeting the correct deployment and metrics (e.g., CPU/memory utilization or custom metrics).
**Prometheus Metrics:** I’d check Prometheus to ensure it’s scraping the application’s metrics correctly. I’d use PromQL queries (e.g., rate(container_cpu_usage_seconds_total[5m])) to verify CPU/memory usage trends.
**Grafana Dashboards:** I’d use Grafana to visualize HPA metrics and pod resource usage. If metrics are missing, I’d check the Prometheus ServiceMonitor or application’s metrics endpoint.
**Cluster Capacity:** I’d check if the EKS cluster has enough nodes to accommodate new pods using kubectl get nodes and kubectl describe node. If nodes are at capacity, I’d scale the node group using Terraform (aws_eks_node_group resource).
**CloudWatch Metrics:** I’d use CloudWatch to check EKS cluster metrics (e.g., CPUUtilization, MemoryUtilization) and Fluentd-forwarded application logs for errors during scaling.
**Pod Scheduling:** I’d check for pod scheduling issues using kubectl get events to identify if pods are stuck in Pending due to resource constraints or taints/tolerations.
**Resolution:** I’d adjust HPA thresholds, scale the node group, or optimize the application’s resource requests/limits. I’d monitor the fix via Grafana and CloudWatch to ensure performance improves.
Thought Process: This tests your ability to troubleshoot Kubernetes autoscaling and leverage monitoring tools. The answer covers HPA, cluster capacity, and monitoring, showing how to use Prometheus, Grafana, and CloudWatch to diagnose and fix scaling issues.

Key Tools and Their Roles in the Pipeline
GitHub: Source code management and version control.
Jenkins: CI/CD pipeline for building, testing, and deploying Docker images.
Docker: Containerizes applications for consistent deployment.
Kubernetes/EKS: Orchestrates containers and manages deployments on Amazon EKS.
Terraform: Manages EKS infrastructure and other AWS resources as code.
Helm: Manages Kubernetes applications and simplifies deployments.
Prometheus: Collects and stores metrics for monitoring.
Grafana: Visualizes metrics from Prometheus for real-time insights.
CloudWatch/Fluentd: Aggregates logs and metrics for monitoring and alerting.
