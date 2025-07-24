
We use RBAC (Role-Based Access Control) in real-time Kubernetes environments to manage who can do what on which resources. It's a crucial part of Kubernetes security and multi-tenancy.
**Secure Cluster Access**
Prevent unauthorized access to sensitive resources like secrets, deployments, or persistent volumes.
Control access by principle of least privilege — only allow what is needed.
**Multi-Tenant Isolation**
In a shared cluster (dev, staging, prod), you want to isolate teams, projects, or services.

Developers shouldn't access other teams’ namespaces or workloads.
**Audit & Compliance**
Enforces access policies for compliance (HIPAA, SOC2, ISO).

Easy to audit: “Who has access to what?”

**Avoid Accidental Changes**
Prevent developers or pods from modifying production services accidentally.

Only CI/CD service accounts or operators should have certain write permissions.

 who?

 Non-admins (like developers, SREs, CI/CD tools, monitoring agents) often need limited and scoped access to Kubernetes clusters to do their jobs efficiently without relying on admins for every action.

**1. Developers Need Limited Access**
 why?

 They need to:

1) Check pod logs (kubectl logs)

2) Port-forward to test services (kubectl port-forward)

3) Describe/debug deployments (kubectl describe)

4) Scale or restart pods (in dev/staging only)

🚫 Without RBAC: Admins would have to do all this manually = bottlenecks & frustration
✅ With RBAC: Developers get safe, scoped access in only their namespace


**2.CI/CD Tools Need Cluster Access**
Examples:

ArgoCD, GitHub Actions, Jenkins, etc.

Need to:

Deploy applications (create/update Deployments, Services)

Roll back apps

Watch rollout status

These tools use ServiceAccounts with RBAC permissions
E.g., "ArgoCD can deploy in dev-namespace, but not delete in prod"

**3. Monitoring & Logging Agents Need Read Access**
Examples:

Prometheus, Datadog, Grafana, Fluent Bit

Need to:

Read metrics from pods and services

Discover targets from endpoints, services

Watch for new pods

Use **read-only Roles** to avoid giving these agents too much power.


 #Kubernetes Role-Based Access Control (RBAC) to restrict user and pod permissions

Restrict:

A user (e.g., dev-user) to only list and get pods in a specific namespace (dev-namespace)

A pod/service account to only read ConfigMaps in its namespace.

create a namespace(if you don't have any existing one?)

>kubectl create namespace dev-namespace

Create a role for user(limit to list/get pods)

```
# role-pod-reader.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev-namespace
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
```

>kubectl apply -f role-pod-reader.yaml
```
# rolebinding-dev-user.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: dev-namespace
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io

```
>kubectl apply -f rolebinding-dev-user.yaml

**Test user access(useing kubectl impersonation)**

> kubectl auth can-i list pods --namespace=dev-namespace --as=dev-user

**4. Restrict Pod Access (via ServiceAccount)**

**Create a Role for pods to read ConfigMaps:**

```
# role-configmap-reader.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev-namespace
  name: configmap-reader
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
```

>kubectl apply -f role-configmap-reader.yaml
**creeate Serviceaccount:**

>kubectl create serviceaccount pod-reader-sa -n dev-namespace
```
# rolebinding-pod-sa.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: bind-configmap-read
  namespace: dev-namespace
subjects:
- kind: ServiceAccount
  name: pod-reader-sa
  namespace: dev-namespace
roleRef:
  kind: Role
  name: configmap-reader
  apiGroup: rbac.authorization.k8s.io
```

>kubectl apply -f rolebinding-pod-sa.yaml
```
# pod-using-sa.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-configmap-access
  namespace: dev-namespace
spec:
  serviceAccountName: pod-reader-sa
  containers:
  - name: busybox
    image: busybox
    command: ["sleep", "3600"]
```
>kubectl auth can-i get configmaps --as=system:serviceaccount:dev-namespace:pod-reader-sa -n dev-namespace

**Alterrnatives to RBAC:**

 Open Policy Agent (OPA) / Gatekeeper
✅ Most powerful & popular alternative/complement to RBAC

OPA lets you write Rego policies to enforce complex, organization-specific rules.

🔍 Examples:

Only users in a specific group can deploy to prod

Deny hostPath volumes or privileged containers

Require specific labels/annotations on resources

🔧 Works as an admission controller + policy engine.

**User -> kube-apiserver -> RBAC -> OPA/Gatekeeper -> Admit or Deny**

**IAM-Based Access (Cloud Provider Managed Clusters)**

Used with EKS (AWS), GKE (Google), AKS (Azure)

External users authenticate using cloud IAM identities

IAM policies grant access at the cluster level, often mapped to Kubernetes RBAC

Note: In EKS, IAM roles are mapped to Kubernetes users/groups via aws-auth config map. This is authentication, not fine-grained authorization. Still needs RBAC.

**Service Mesh-Level Access Control**

Controls traffic between services, not user access to the cluster.

Tools like Istio, Linkerd, or Consul can:

Control which services can talk to others

Enforce TLS, mTLS, and traffic policies

Note: Useful for zero-trust networking, but doesn't replace RBAC for API access.
