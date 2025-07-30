Here's your Jenkins notes on debugging Kubernetes issues in production, formatted for clarity and ease of use:

---

## Debugging Kubernetes Issues in Production: A Step-by-Step Guide

Kubernetes has revolutionized application deployment, offering scalability and automation. However, running Kubernetes in production can present challenges, from crashing pods to unreachable services. Effective debugging is crucial to maintaining system reliability and performance. This guide outlines the process of diagnosing and resolving common Kubernetes issues to keep your clusters stable and efficient.

---

### Understanding the Basics of Kubernetes Debugging

Kubernetes is a complex system of **Pods**, **Nodes**, **Services**, and **Deployments**. When an issue arises, understanding which component is malfunctioning is key. Debugging involves examining logs, events, and resource statuses.

**Key Concepts:**
* **Logs:** Provide insights into operations and errors within Pods, Nodes, and other components.
* **Events:** Records of what happened in the cluster, offering clues about scheduling, resource allocation, and more.
* **Status Commands:** `kubectl get`, `kubectl describe`, and `kubectl logs` are essential for gathering information about the cluster's state.

**Common Production Issues:**
* **Pod Failures:** `CrashLoopBackOff`, `ImagePullBackOff` due to misconfigurations, image problems, or resource constraints.
* **Node Resource Issues:** Disk pressure, memory shortages, or network issues affecting cluster performance.
* **Service Unavailability:** Due to networking issues, incorrect configurations, or service discovery problems.
* **Persistent Storage Problems:** Issues with PersistentVolumes (PVs) and PersistentVolumeClaims (PVCs) disrupting data availability.

---

### Debugging Pods and Containers

Pods are the smallest deployable units; they're often the first place to investigate.

1.  **Checking Pod Status:**
    * **Command:** `kubectl get pods`
    * **Purpose:** Overview of all pods in a namespace, showing statuses (e.g., `Running`, `Pending`, `CrashLoopBackOff`).

2.  **Describing the Pod:**
    * **Command:** `kubectl describe pod <pod-name>`
    * **Purpose:** Detailed information: configuration, events, and error messages. Helps diagnose `CrashLoopBackOff` by examining events and logs.

3.  **Viewing Pod Logs:**
    * **Command:** `kubectl logs <pod-name> -c <container-name>`
    * **Purpose:** Understand what went wrong inside a container. Analyze application-level errors, failed initializations, or misconfigurations.
    * *Note:* Omit `-c <container-name>` if the pod has only one container.

4.  **Example Scenario: Troubleshooting `CrashLoopBackOff`**
    * A pod in `CrashLoopBackOff` indicates repeated container failure.
    * **Steps:**
        * Use `kubectl describe pod <pod-name>` for error messages or misconfigurations.
        * Check pod logs using `kubectl logs <pod-name>` for specific application errors.
        * Ensure the container image is correct and accessible.
        * Verify required environment variables, secrets, and config maps are properly configured.

5.  **Debugging Container Performance Issues:**
    * **Command:** `kubectl top pod <pod-name>`
    * **Purpose:** Shows CPU and memory usage of the pod to identify resource bottlenecks or insufficient allocations.

---

### Investigating Node-Level Issues

Nodes are the backbone of your cluster. Node issues can lead to widespread disruptions.

1.  **Checking Node Status:**
    * **Command:** `kubectl get nodes`
    * **Purpose:** Lists all nodes and their statuses (e.g., `Ready`, `NotReady`). `NotReady` requires immediate attention.

2.  **Describing the Node:**
    * **Command:** `kubectl describe node <node-name>`
    * **Purpose:** Comprehensive view of node condition, including events, resource usage, and warnings (e.g., disk pressure).

3.  **Monitoring Node Resource Usage:**
    * **Command:** `kubectl top nodes`
    * **Purpose:** Identifies nodes running out of CPU or memory, which can cause pod evictions or scheduling failures.

4.  **Addressing Common Node Issues:**
    * **Disk Pressure:** Clear unused data, adjust pod resource requests/limits, or add storage.
    * **Network Problems:** Check network configurations and logs.
    * **Kubelet Issues:** If the Kubelet (node agent) is down or misconfigured, restart it or inspect its logs.

---

### Troubleshooting Network and Service Issues

Networking issues can lead to unreachable services, communication failures, or DNS problems.

1.  **Inspecting Services:**
    * **Command:** `kubectl get svc`
    * **Purpose:** Lists services with details like Cluster IPs, External IPs, and ports. Helps identify incorrect configurations or missing endpoints.

2.  **Describing Services:**
    * **Command:** `kubectl describe svc <service-name>`
    * **Purpose:** Detailed information: type (ClusterIP, NodePort, LoadBalancer), port configurations, and linked pods. Helps identify misconfigured ports or selectors.

3.  **Checking Endpoints:**
    * **Command:** `kubectl get endpoints <service-name>`
    * **Purpose:** Ensures the service is correctly linked to pods. No listed endpoints indicate traffic routing issues due to label mismatches or pod readiness.

4.  **Troubleshooting DNS Issues:**
    * **Command:** `kubectl get pods -n kube-system -l k8s-app=kube-dns`
    * **Purpose:** Verifies CoreDNS pod status. If CoreDNS pods are problematic, services relying on DNS will fail. Restart CoreDNS pods or check their logs.

5.  **Example Scenario: Debugging a Failed Service:**
    * Suppose a service is not reachable:
        * Use `kubectl describe svc <service-name>` for misconfigurations.
        * Verify correct endpoints with `kubectl get endpoints <service-name>`.
        * If `LoadBalancer` type, ensure the external load balancer is configured and operational.

---

### Analyzing Persistent Storage Problems

Persistent storage is crucial for stateful applications. Issues can cause data loss or application startup failures.

1.  **Inspecting PersistentVolumeClaims (PVCs):**
    * **Command:** `kubectl get pvc`
    * **Purpose:** Lists PVCs and their statuses (`Bound`, `Pending`, `Lost`). `Pending` or `Lost` indicates underlying storage problems.

2.  **Describing PersistentVolumeClaims (PVCs):**
    * **Command:** `kubectl describe pvc <pvc-name>`
    * **Purpose:** Details: associated PersistentVolume (PV), storage class, and events. Helps diagnose why a PVC is stuck in `Pending` (e.g., insufficient resources in storage class, storage provider issue).

3.  **Inspecting PersistentVolumes (PVs):**
    * **Command:** `kubectl get pv`
    * **Purpose:** Lists all PVs, showing capacity, access modes, and status. An unbound PV or insufficient capacity can cause PVCs to remain `Pending`.

4.  **Debugging Common Storage Issues:**
    * **Unbound PVCs:** Ensure available PVs match the requested storage class and capacity.
    * **Storage Class Issues:** Check `kubectl get storageclass <storageclass-name>` for incorrect configurations leading to PVC binding failures.
    * **Access Mode Mismatches:** Ensure PVC access mode matches PV (e.g., `ReadWriteOnce`, `ReadOnlyMany`).

---

### Leveraging Advanced Tools for Kubernetes Debugging

Beyond basic `kubectl` commands, advanced tools offer deeper insights.

1.  **Using `kubectl debug`:**
    * Creates a copy of a pod with debugging tools or runs a new container in an existing pod.
    * **Example:** `kubectl debug pod/<pod-name> -it --image=busybox -- /bin/sh` creates a debugging container.

2.  **Executing Commands Inside a Container:**
    * **Command:** `kubectl exec -it <pod-name> -c <container-name> -- /bin/sh`
    * **Purpose:** Opens an interactive shell session to run commands (e.g., check configurations, test connectivity).

3.  **Monitoring with Prometheus and Grafana:**
    * **Prometheus:** Collects metrics from Kubernetes components (nodes, pods, services).
    * **Grafana:** Provides visualizations and dashboards for real-time monitoring and alerting.
    * **Benefit:** Proactive monitoring to identify and resolve issues before they impact production (CPU usage, memory consumption, network traffic).

4.  **Using Third-Party Debugging Tools:**
    * **K9s:** Terminal UI for managing and troubleshooting Kubernetes clusters.
    * **Lens:** Powerful IDE for Kubernetes with advanced monitoring and management features.
    * **Stern:** Tails multiple pod logs simultaneously, simplifying debugging across containers.

---

