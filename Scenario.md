---

# 🚀 DevOps Troubleshooting & Optimization Scenarios

---

### **1. Broken Jenkins Pipeline**

**Scenario:** Jenkins pipeline fails at Docker build stage → *"Cannot connect to the Docker daemon."*

**Troubleshooting Steps:**

* ✅ **Verify Docker Daemon** → `systemctl status docker` / `docker info`
* ✅ **Check Permissions** → Ensure Jenkins user in `docker` group → `usermod -aG docker jenkins`
* ✅ **Jenkins Config** → If Jenkins runs in Docker, mount `/var/run/docker.sock`
* ✅ **Pipeline Script** → Check `docker build` command & Dockerfile location
* ✅ **Network Issues** → Validate connectivity if remote agent used
* ✅ **Logs** → Check Jenkins logs, manually run build on agent

**Resolution:** Restart daemon / fix permissions / mount socket / correct pipeline script.
**Thought Process:** Systematic troubleshooting (service → permissions → config → logs).

---

### **2. Slow Deployment to EKS**

**Scenario:** Helm deployments to EKS taking longer than expected.

**Diagnosis & Fixes:**

* 🔹 **Helm Chart** → Optimize resource requests/limits
* 🔹 **Cluster Health** → Check node resources (`kubectl get/describe node`, events)
* 🔹 **HPA** → Validate scaling configs, avoid over-scaling
* 🔹 **Image Size** → Optimize Dockerfile (multi-stage builds, Alpine base)
* 🔹 **Network Latency** → Check pulls from ECR, CloudWatch metrics
* 🔹 **Helm Optimization** → Use `--atomic`, `--wait`, prune dependencies
* 🔹 **Monitoring** → Use Prometheus/Grafana/CloudWatch for deployment timings

**Resolution:** Optimize Dockerfile/Helm configs, adjust limits, scale nodes.
**Thought Process:** Performance optimization across layers (Helm → K8s → Infra).

---

### **3. Terraform Drift in EKS**

**Scenario:** Node group size changed manually in AWS console → Terraform drift.

**Steps:**

* 🔎 **Detect Drift** → `terraform plan` shows mismatch
* 🔎 **Review State** → `terraform state list/show`
* 🔎 **Resolve Drift:**

  * Option 1: `terraform apply` → Revert to code-defined config
  * Option 2: Update Terraform code → Align with manual change
* 🔎 **Prevent Future Drift:**

  * Restrict manual AWS changes with IAM
  * Enforce IaC via CI/CD pipelines
  * Use remote backend with DynamoDB lock
  * Monitor via CloudWatch + Grafana

**Thought Process:** Drift detection → resolution → prevention strategy.

---

### **4. Monitoring Gaps (Prometheus/Grafana)**

**Scenario:** Missing critical app metrics in Grafana.

**Checks:**

* 🔹 **Prometheus Config** → Ensure metrics endpoint in `prometheus.yml`
* 🔹 **App Metrics** → Validate `/metrics` endpoint via `curl`
* 🔹 **Service Discovery** → Verify K8s labels/roles in scrape config
* 🔹 **ServiceMonitor/PodMonitor** → Correct configs if using Prometheus Operator
* 🔹 **Grafana Dashboards** → Test PromQL in *Explore*
* 🔹 **Logs Integration** → Check Fluentd → CloudWatch filters

**Resolution:** Fix Prometheus scrape configs, expose metrics properly, adjust dashboards.
**Thought Process:** End-to-end monitoring validation (scrape → app → discovery → dashboard).

---

### **5. Security Issue in Docker Image**

**Scenario:** Critical vulnerability found in base image.

**Steps:**

* 🔎 **Identify** → Scan image with Trivy/Clair
* 🔎 **Update Base Image** → Patch to secure version / switch to Alpine
* 🔎 **Rebuild & Redeploy** → Trigger Jenkins, push to ECR, deploy via Helm/K8s
* 🔎 **Verify** → Ensure new image runs, rescan
* 🔎 **Prevent Future Issues:**

  * Integrate security scans in Jenkins pipeline
  * Use Dependabot/Renovate for base image updates
  * Enforce signed images with Cosign
  * Monitor vulnerabilities via Prometheus/Grafana

**Thought Process:** Immediate fix + proactive prevention (security shift-left).

---

### **6. Log Aggregation Failure (Fluentd → CloudWatch)**

**Scenario:** Logs missing in CloudWatch.

**Checks:**

* 🔹 **Fluentd Pods** → Running? Check `kubectl logs`
* 🔹 **Fluentd Config** → Validate ConfigMap match/filter rules
* 🔹 **Pod Labels** → Ensure correct metadata for log forwarding
* 🔹 **IAM Permissions** → Validate CloudWatch write access (`logs:PutLogEvents`)
* 🔹 **Network** → Check connectivity to CloudWatch endpoint
* 🔹 **CloudWatch Logs** → Look for partial/error logs

**Resolution:** Fix config/IAM/labels, redeploy Fluentd, validate logs in CloudWatch.
**Thought Process:** Covers Fluentd config → IAM → K8s integration → CloudWatch.

---

### **7. Scaling Issues in EKS**

**Scenario:** Pods not scaling under peak load.

**Checks:**

* 🔹 **HPA Config** → `kubectl describe hpa`, validate targets/metrics
* 🔹 **Prometheus Metrics** → Check CPU/mem usage with PromQL
* 🔹 **Grafana Dashboards** → Visualize scaling triggers
* 🔹 **Cluster Capacity** → `kubectl get/describe node`, check resource exhaustion
* 🔹 **CloudWatch Metrics** → Check CPUUtilization, MemoryUtilization
* 🔹 **Pod Scheduling** → Look at `kubectl get events` for pending pods

**Resolution:** Adjust HPA thresholds, scale node group (via Terraform), optimize pod requests/limits.
**Thought Process:** End-to-end scaling diagnosis (HPA → metrics → cluster capacity).

---

# 🔧 Key Tools in CI/CD & Observability

* **GitHub** → Source code management
* **Jenkins** → CI/CD pipeline orchestration
* **Docker** → App containerization
* **EKS (Kubernetes)** → Container orchestration & deployment
* **Terraform** → Infra as Code (EKS, AWS resources)
* **Helm** → K8s package management & deployment automation
* **Prometheus** → Metrics collection
* **Grafana** → Metrics visualization
* **CloudWatch + Fluentd** → Centralized logging & alerting

---

👉 This format gives you **quick-scan notes** for interviews: each scenario → checks → resolution → thought process.

