# Headlamp and Kubernetes RBAC

## 1. Overview

Headlamp is a web-based Kubernetes dashboard that provides a graphical interface for managing and monitoring Kubernetes clusters.

In this project, we installed Headlamp on our KIND cluster and configured separate ServiceAccounts for viewer and administrator access using Kubernetes Role-Based Access Control (RBAC).

### What is RBAC?

Role-Based Access Control (RBAC) is a Kubernetes authorization mechanism that controls what users, groups, and ServiceAccounts can do within a cluster.

RBAC uses four main resources:

| Resource | Purpose |
|---|---|
| Role | Defines permissions within a namespace |
| ClusterRole | Defines permissions at cluster scope or reusable permissions |
| RoleBinding | Grants a Role or ClusterRole within a namespace |
| ClusterRoleBinding | Grants a ClusterRole across the cluster |

**Important:** A Role or ClusterRole defines permissions. A RoleBinding or ClusterRoleBinding grants those permissions to a subject.

---

## 2. Headlamp Installation Using Helm

We installed Headlamp using its official Helm chart.

### Step 1: Add the Helm repository

```bash
helm repo add headlamp https://kubernetes-sigs.github.io/headlamp/
```

Update Helm repositories:

```bash
helm repo update
```

### Step 2: Check the chart version

```bash
helm search repo headlamp
```

We used:

- Chart version: `0.46.0`
- Application version: `0.46.0`

### Step 3: Inspect chart values

```bash
helm show values headlamp/headlamp
```

During inspection, we noticed that the chart can create a ClusterRoleBinding granting `cluster-admin` permissions by default.

For our implementation, we disabled this automatic binding:

```yaml
clusterRoleBinding:
  create: false
```

This allowed us to configure separate viewer and administrator permissions ourselves.

### Step 4: Render the Helm manifest

Before installation, we generated the Kubernetes resources using:

```bash
helm template headlamp headlamp/headlamp \
  --version 0.46.0 \
  --namespace headlamp \
  --set clusterRoleBinding.create=false \
  > /tmp/headlamp-rendered.yaml
```

We verified that:

- A ServiceAccount named `headlamp` was created.
- The Deployment used `serviceAccountName: headlamp`.
- A ClusterIP Service was created.
- No chart-generated ClusterRoleBinding was present.

### Step 5: Install Headlamp

```bash
helm install headlamp headlamp/headlamp \
  --version 0.46.0 \
  --namespace headlamp \
  --create-namespace \
  --set clusterRoleBinding.create=false \
  --set config.unsafeUseServiceAccountToken=false
```

### Step 6: Verify installation

```bash
helm list -n headlamp
kubectl get all -n headlamp
kubectl get serviceaccount -n headlamp
```

Expected result:

- Headlamp Pod: Running
- Deployment: 1/1 available
- Service: ClusterIP
- ServiceAccount: `headlamp`

---

## 3. Accessing Headlamp

We used Kubernetes port forwarding to access the dashboard locally.

```bash
kubectl port-forward -n headlamp svc/headlamp 8080:80
```

Open the following address in a browser:

```text
http://localhost:8080
```

Headlamp displays a login page where we authenticate using a Kubernetes credential.

**Important:** Port forwarding provides local access to the Service. It does not expose Headlamp publicly.

---

## 4. Understanding ServiceAccounts

### What is a ServiceAccount?

A ServiceAccount provides an identity for processes running inside or interacting with a Kubernetes cluster.

Examples include:

- Applications running inside Pods.
- Monitoring tools.
- Automation systems.
- Kubernetes dashboard users and integrations.

Unlike a normal human user identity, a ServiceAccount is a Kubernetes API resource.

### Why did we create separate ServiceAccounts?

We created separate identities for different access levels:

| ServiceAccount | Purpose |
|---|---|
| `headlamp-viewer` | Read-only cluster access |
| `headlamp-admin` | Administrative cluster access |

This follows the principle of least privilege: give each identity only the permissions it needs.

---

## 5. Viewer ServiceAccount

File: `manifests/headlamp-viewer-sa.yml`

```yaml
apiVersion: v1
kind: ServiceAccount

metadata:
  name: headlamp-viewer
  namespace: headlamp
```

Apply:

```bash
kubectl apply -f manifests/headlamp-viewer-sa.yml
```

Verify:

```bash
kubectl get serviceaccount -n headlamp
```

The ServiceAccount creates an identity, but it does not automatically grant permissions.

We must bind it to an appropriate Role or ClusterRole.

---

## 6. Viewer ClusterRole

File: `manifests/headlamp-viewer-role.yml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole

metadata:
  name: headlamp-viewer

rules:
  - apiGroups: [""]
    resources:
      - pods
      - services
      - namespaces
      - nodes
      - configmaps
      - events
    verbs:
      - get
      - list
      - watch

  - apiGroups: ["apps"]
    resources:
      - deployments
      - replicasets
      - statefulsets
      - daemonsets
    verbs:
      - get
      - list
      - watch

  - apiGroups: ["batch"]
    resources:
      - jobs
      - cronjobs
    verbs:
      - get
      - list
      - watch
```

### Understanding the configuration

**apiGroups**

Identifies the Kubernetes API group containing the requested resources.

| API group | Resources |
|---|---|
| `""` | Core resources such as Pods and Services |
| `apps` | Deployments, ReplicaSets, StatefulSets, DaemonSets |
| `batch` | Jobs and CronJobs |

The empty string represents the core API group, not a missing API group.

**resources**

Defines which Kubernetes resources the ServiceAccount can access.

**verbs**

Defines the permitted operations.

| Verb | Meaning |
|---|---|
| `get` | Read a specific resource |
| `list` | List resources |
| `watch` | Watch for resource changes |
| `create` | Create resources |
| `update` | Update resources |
| `patch` | Modify selected fields |
| `delete` | Delete resources |

Our viewer ClusterRole only grants `get`, `list`, and `watch`.

It does not grant permission to create, modify, or delete resources.

### Apply and verify

```bash
kubectl apply -f manifests/headlamp-viewer-role.yml
```

Inspect:

```bash
kubectl describe clusterrole headlamp-viewer
```

---

## 7. Viewer ClusterRoleBinding

File: `manifests/headlamp-viewer-binding.yml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding

metadata:
  name: headlamp-viewer-binding

subjects:
  - kind: ServiceAccount
    name: headlamp-viewer
    namespace: headlamp

roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: headlamp-viewer
```

### Understanding the fields

**subjects**

Identifies the user, group, or ServiceAccount receiving permissions.

In our case:

- Kind: ServiceAccount
- Name: headlamp-viewer
- Namespace: headlamp

**roleRef**

Identifies the ClusterRole whose permissions are being granted.

### Why ClusterRoleBinding?

We wanted the viewer to inspect cluster-wide resources such as:

- Nodes
- Namespaces
- Pods across namespaces
- Deployments across namespaces

A ClusterRoleBinding grants the referenced ClusterRole permissions across the cluster, subject to the rules defined in that ClusterRole.

### Apply and verify

```bash
kubectl apply -f manifests/headlamp-viewer-binding.yml
```

Verify:

```bash
kubectl get clusterrolebinding headlamp-viewer-binding
kubectl describe clusterrolebinding headlamp-viewer-binding
```

---

## 8. Testing Viewer Permissions

We tested the viewer identity using `kubectl auth can-i`.

### Test 1: List Pods across namespaces

```bash
kubectl auth can-i list pods --all-namespaces \
  --as=system:serviceaccount:headlamp:headlamp-viewer
```

Expected:

```text
yes
```

### Test 2: Delete Pods

```bash
kubectl auth can-i delete pods -n capstone \
  --as=system:serviceaccount:headlamp:headlamp-viewer
```

Expected:

```text
no
```

### Test 3: Read Secrets

```bash
kubectl auth can-i get secrets -n capstone \
  --as=system:serviceaccount:headlamp:headlamp-viewer
```

Expected:

```text
no
```

We also tested deployment creation, deployment patching, and StatefulSet deletion. These operations were denied.

### Viewer permission summary

| Operation | Result |
|---|---|
| List Pods | Yes |
| Read Deployments | Yes |
| Watch resources | Yes |
| Delete Pods | No |
| Create Deployments | No |
| Patch Deployments | No |
| Delete StatefulSets | No |
| Read Secrets | No |

This confirms that our viewer identity has restricted read-only permissions.

---

## 9. Administrator ServiceAccount

File: `manifests/headlamp-admin.yml`

We created a separate ServiceAccount and bound it to the built-in `cluster-admin` ClusterRole.

```yaml
apiVersion: v1
kind: ServiceAccount

metadata:
  name: headlamp-admin
  namespace: headlamp

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding

metadata:
  name: headlamp-admin-binding

subjects:
  - kind: ServiceAccount
    name: headlamp-admin
    namespace: headlamp

roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
```

### Understanding cluster-admin

Kubernetes provides a built-in ClusterRole named `cluster-admin`.

It grants broad administrative permissions across the cluster.

Our administrator ServiceAccount can perform operations such as:

- List resources.
- Create resources.
- Update resources.
- Delete resources.
- Access Secrets.

**Security warning:** `cluster-admin` is highly privileged. Use it only when necessary and protect its credentials carefully.

### Apply and verify

```bash
kubectl apply -f manifests/headlamp-admin.yml
```

Verify:

```bash
kubectl get serviceaccount headlamp-admin -n headlamp
kubectl get clusterrolebinding headlamp-admin-binding
kubectl describe clusterrolebinding headlamp-admin-binding
```

---

## 10. Testing Administrator Permissions

We verified the administrator's permissions.

### Test 1: List Pods

```bash
kubectl auth can-i list pods --all-namespaces \
  --as=system:serviceaccount:headlamp:headlamp-admin
```

Expected:

```text
yes
```

### Test 2: Delete Pods

```bash
kubectl auth can-i delete pods -n capstone \
  --as=system:serviceaccount:headlamp:headlamp-admin
```

Expected:

```text
yes
```

### Test 3: Read Secrets

```bash
kubectl auth can-i get secrets -n capstone \
  --as=system:serviceaccount:headlamp:headlamp-admin
```

Expected:

```text
yes
```

### Administrator permission summary

| Operation | Result |
|---|---|
| List Pods | Yes |
| Delete Pods | Yes |
| Read Secrets | Yes |
| Create resources | Yes |
| Modify resources | Yes |

These tests confirm that the administrator identity has the broad permissions granted by `cluster-admin`.

---

## 11. Authentication Using ServiceAccount Tokens

### What is a ServiceAccount token?

A ServiceAccount token is a credential that can authenticate an identity to the Kubernetes API server.

Kubernetes supports requesting short-lived tokens through the TokenRequest API.

We generated tokens using:

```bash
kubectl create token headlamp-viewer -n headlamp --duration=1h
```

For the administrator:

```bash
kubectl create token headlamp-admin -n headlamp --duration=1h
```

The generated token can be entered into Headlamp's login page.

**Security note:** Tokens are credentials. Never publish them in GitHub, documentation, screenshots, chat messages, or shell scripts committed to a repository.

### Important token security lesson

During our practical exercise, a portion of an administrator token was accidentally exposed.

We treated that credential as compromised, recreated the administrator ServiceAccount, and generated a fresh token.

This reinforced an important security practice: once a credential is exposed, do not continue trusting it.

For future use:

- Generate short-lived tokens.
- Avoid sharing token values.
- Never hardcode tokens in manifests.
- Avoid saving credentials in repository files.
- Rotate or invalidate credentials if exposed.

### Authentication vs Authorization

These are two different concepts.

| Authentication | Authorization |
|---|---|
| Verifies identity | Checks permissions |
| Who are you? | What can you do? |
| Uses credentials such as tokens | Uses RBAC policies |
| Identifies the ServiceAccount | Evaluates its permissions |

Example:

A viewer token authenticates the identity:

```text
system:serviceaccount:headlamp:headlamp-viewer
```

RBAC then determines whether that identity can list Pods, read Secrets, or delete Deployments.

---

## 12. Verifying Identity with kubectl auth whoami

We used:

```bash
kubectl auth whoami
```

This command displays the identity authenticated by the Kubernetes API server.

### Important kubeconfig lesson

Our original kubeconfig used client-certificate authentication.

Initially, we tried passing a token with:

```bash
kubectl auth whoami --token="$(kubectl create token headlamp-admin -n headlamp --duration=10m)"
```

However, the output still showed:

```text
Username: kubernetes-admin
```

Why?

The existing kubeconfig already contained client-certificate credentials. Passing a token this way did not replace the configured certificate identity as we expected.

### Correct verification approach

We copied the kubeconfig to a temporary file and configured a separate user entry using the ServiceAccount token.

The essential commands were:

```bash
TOKEN="$(kubectl create token headlamp-admin -n headlamp --duration=10m)"

export KUBECONFIG="$(cygpath -w /tmp/headlamp-token-test-config)"

kubectl config set-credentials headlamp-token-user --token="$TOKEN"

kubectl config set-context kind-my-cluster --user=headlamp-token-user

kubectl auth whoami
```

The successful output identified:

```text
Username: system:serviceaccount:headlamp:headlamp-admin
```

This confirmed that authentication was performed using the administrator ServiceAccount token rather than the original client certificate.

### Cleanup

After verification, we removed the temporary authentication environment:

```bash
unset TOKEN
rm /tmp/headlamp-token-test-config
unset KUBECONFIG
```

The original kubeconfig context was then checked:

```bash
kubectl config current-context
```

Expected:

```text
kind-my-cluster
```

**Important:** Never modify your primary kubeconfig for a temporary authentication experiment when a separate copy can be used.

---

## 13. Headlamp Login Verification

We successfully logged in to Headlamp using both identities.

| Login identity | Access |
|---|---|
| `headlamp-viewer` | Read-only |
| `headlamp-admin` | Cluster administrator |

The Headlamp administrator session displayed the authenticated identity:

```text
system:serviceaccount:headlamp:headlamp-admin
```

This confirmed that the dashboard was using the intended ServiceAccount identity.

---

## 14. Important Kubernetes RBAC Concepts

### Role vs ClusterRole

| Role | ClusterRole |
|---|---|
| Namespace-scoped resource | Cluster-scoped resource |
| Defines permissions within a namespace | Defines permissions at cluster scope or reusable permissions |
| Can be granted using RoleBinding | Can be granted using RoleBinding or ClusterRoleBinding |

A ClusterRole does not automatically grant access. It must be bound to a subject.

### RoleBinding vs ClusterRoleBinding

| RoleBinding | ClusterRoleBinding |
|---|---|
| Grants permissions within one namespace | Grants permissions across the cluster |
| Can reference a Role or ClusterRole | References a ClusterRole |
| Useful for namespace-specific access | Useful for cluster-wide access |

### ServiceAccount vs ClusterRole

| ServiceAccount | ClusterRole |
|---|---|
| Represents an identity | Defines permissions |
| Answers who is accessing the API | Defines what operations are allowed |
| Does not grant permissions by itself | Does not identify the subject |

---

## 15. Troubleshooting

### Problem 1: ServiceAccount not found

Check the namespace:

```bash
kubectl get serviceaccounts -n headlamp
```

Remember that ServiceAccounts are namespaced resources.

### Problem 2: Viewer cannot access a resource

Check the ClusterRole:

```bash
kubectl describe clusterrole headlamp-viewer
```

Check the ClusterRoleBinding:

```bash
kubectl describe clusterrolebinding headlamp-viewer-binding
```

Test the permission:

```bash
kubectl auth can-i list pods --all-namespaces \
  --as=system:serviceaccount:headlamp:headlamp-viewer
```

### Problem 3: Headlamp login fails

Check:

```bash
kubectl get pods -n headlamp
kubectl get service -n headlamp
kubectl logs deployment/headlamp -n headlamp
```

Also verify:

- The token is valid and has not expired.
- The correct ServiceAccount was used.
- RBAC permissions are configured.
- The port-forward session is active.

### Problem 4: Administrator token does not show the expected identity

Check the active context:

```bash
kubectl config current-context
```

Inspect the active kubeconfig:

```bash
kubectl config view --minify
```

Remember that an existing client certificate may authenticate a different identity from the ServiceAccount token you intended to test.

Use an isolated temporary kubeconfig to test token authentication.

---

## 16. Security Best Practices

1. Follow the principle of least privilege.
2. Avoid granting `cluster-admin` to normal dashboard users.
3. Use separate identities for viewer and administrator access.
4. Avoid granting read access to Secrets unless required.
5. Use short-lived ServiceAccount tokens.
6. Never commit tokens or kubeconfig credentials to GitHub.
7. Protect administrative credentials.
8. Prefer namespace-scoped permissions when cluster-wide access is unnecessary.
9. Avoid unnecessary permissions on ConfigMaps because they may contain sensitive configuration.
10. Review RBAC rules periodically.

**Important:** Our viewer ClusterRole includes read access to ConfigMaps. Although Secrets are excluded, ConfigMaps can still contain sensitive configuration and should be protected appropriately.

---

## 17. Interview Questions and Answers

**Q1. What is Kubernetes RBAC?**

RBAC is an authorization mechanism that controls access to Kubernetes resources using Roles, ClusterRoles, RoleBindings, and ClusterRoleBindings.

**Q2. What is a ServiceAccount?**

A ServiceAccount provides an identity for processes that interact with the Kubernetes API.

**Q3. What is the difference between Role and ClusterRole?**

A Role defines permissions within a namespace, whereas a ClusterRole defines permissions at cluster scope or reusable permissions that can be granted through bindings.

**Q4. What is the difference between RoleBinding and ClusterRoleBinding?**

RoleBinding grants permissions within a namespace. ClusterRoleBinding grants the referenced ClusterRole permissions across the cluster.

**Q5. Does creating a ServiceAccount automatically give it permissions?**

No. A ServiceAccount needs an appropriate RoleBinding or ClusterRoleBinding to receive permissions through RBAC.

**Q6. What is the principle of least privilege?**

It means giving an identity only the minimum permissions required to perform its tasks.

**Q7. What is the purpose of `kubectl auth can-i`?**

It checks whether a particular identity is authorized to perform an operation on a Kubernetes resource.

**Q8. What is the purpose of `kubectl auth whoami`?**

It displays the identity authenticated by the Kubernetes API server.

**Q9. What is the difference between authentication and authorization?**

Authentication verifies who the identity is. Authorization determines what that identity is allowed to do.

**Q10. Why should we avoid using cluster-admin for every user?**

Because it grants extensive cluster permissions. If its credentials are compromised, an attacker could gain control over the cluster.

---

## 18. How to Explain in an Interview

"In my Kubernetes capstone project, I installed Headlamp using Helm on a KIND cluster.

I wanted to understand Kubernetes authentication and authorization, so instead of giving every dashboard user administrator access, I created separate ServiceAccounts for viewer and administrator access.

For the viewer, I created a ClusterRole with read-only permissions such as get, list, and watch. I then connected it to the ServiceAccount using a ClusterRoleBinding.

For the administrator, I created a separate ServiceAccount and bound it to the built-in cluster-admin ClusterRole.

I verified permissions using `kubectl auth can-i`, tested token authentication, and used `kubectl auth whoami` to confirm the authenticated identity.

Finally, I logged in to Headlamp with both accounts and verified that they used the intended identities.

This practical exercise helped me understand ServiceAccounts, RBAC resources, authentication, authorization, token security, and the principle of least privilege."

---

## 19. Quick Revision Cheat Sheet

| Concept | Remember |
|---|---|
| Headlamp | Web-based Kubernetes dashboard |
| Helm | Kubernetes package manager |
| ServiceAccount | Kubernetes identity |
| Role | Namespace-scoped permissions |
| ClusterRole | Cluster-scoped or reusable permissions |
| RoleBinding | Grants permissions within a namespace |
| ClusterRoleBinding | Grants permissions across the cluster |
| `cluster-admin` | Highly privileged built-in ClusterRole |
| `get` | Read a resource |
| `list` | List resources |
| `watch` | Monitor resource changes |
| Authentication | Verifies identity |
| Authorization | Checks permissions |
| ServiceAccount token | Credential for API authentication |
| `kubectl auth can-i` | Checks authorization |
| `kubectl auth whoami` | Displays authenticated identity |
| Least privilege | Minimum required permissions |

### Important commands

```bash
# Check Headlamp installation
helm list -n headlamp

# Check Headlamp resources
kubectl get all -n headlamp

# Check ServiceAccounts
kubectl get serviceaccounts -n headlamp

# Check ClusterRoles
kubectl get clusterroles

# Check ClusterRoleBindings
kubectl get clusterrolebindings

# Verify viewer permissions
kubectl auth can-i list pods --all-namespaces \
  --as=system:serviceaccount:headlamp:headlamp-viewer

# Verify administrator permissions
kubectl auth can-i delete pods -n capstone \
  --as=system:serviceaccount:headlamp:headlamp-admin

# Check authenticated identity
kubectl auth whoami

# Generate a short-lived token
kubectl create token headlamp-viewer -n headlamp --duration=1h

# Access Headlamp locally
kubectl port-forward -n headlamp svc/headlamp 8080:80
```

---

## 20. Project Outcome

Successfully implemented and verified:

- Headlamp installation using Helm.
- Separate viewer and administrator ServiceAccounts.
- Custom read-only ClusterRole.
- Viewer ClusterRoleBinding.
- Administrator ClusterRoleBinding using `cluster-admin`.
- RBAC permission testing using `kubectl auth can-i`.
- ServiceAccount token authentication.
- Identity verification using `kubectl auth whoami`.
- Headlamp login with viewer and administrator identities.
- Practical understanding of authentication, authorization, and least privilege.

**Status:** Headlamp installation and Kubernetes RBAC practical implementation completed and verified.