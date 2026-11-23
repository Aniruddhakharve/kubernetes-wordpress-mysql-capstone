# Metrics Server and Horizontal Pod Autoscaler (HPA)

## 1. Overview

Kubernetes Metrics Server collects resource utilization metrics from nodes and containers and exposes them through the Kubernetes Metrics API.

The Horizontal Pod Autoscaler (HPA) uses these metrics to automatically adjust the number of Pod replicas based on resource utilization.

### Architecture

```text
              Metrics Server
                    |
                    v
             Metrics API
                    |
                    v
          Horizontal Pod Autoscaler
                    |
                    v
             WordPress Deployment
                    |
                    v
              ReplicaSet
                    |
                    v
              WordPress Pods
```

**Important:** Metrics Server collects metrics; HPA makes scaling decisions based on those metrics.

---

## 2. Metrics Server

### What is Metrics Server?

Metrics Server is a cluster-wide aggregator of resource usage data.

It collects CPU and memory metrics from Kubernetes nodes and containers through the kubelet API.

These metrics are used by:

- `kubectl top nodes`
- `kubectl top pods`
- Horizontal Pod Autoscaler

### Installation in KIND

We used Metrics Server v0.8.0.

Two manifests are maintained:

| File | Purpose |
|---|---|
| `metrics-server/components.yaml` | Upstream Metrics Server manifest |
| `metrics-server/components-kind.yaml` | KIND-compatible manifest |

The KIND-compatible manifest includes an additional argument:

```yaml
- --kubelet-insecure-tls
```

**Why is this required in our lab?**

Our local KIND environment does not have kubelet certificate verification configured in a way that Metrics Server accepts. This argument allows Metrics Server to communicate with kubelets without verifying their TLS certificates.

**Security note:** This is a lab-only workaround. Do not use it as a general production recommendation. Production clusters should use properly configured and trusted kubelet certificates.

### Verify Metrics Server

Check the Deployment:

```bash
kubectl get deployment metrics-server -n kube-system
```

Check the Pod:

```bash
kubectl get pods -n kube-system | grep metrics-server
```

Check the Metrics API:

```bash
kubectl get apiservice v1beta1.metrics.k8s.io
```

The APIService should report:

```text
AVAILABLE   True
```

Check node metrics:

```bash
kubectl top nodes
```

Check Pod metrics:

```bash
kubectl top pods -n capstone
```

Example output from our KIND cluster:

```text
NAME                      CPU(cores)   MEMORY(bytes)
my-cluster-control-plane  196m         679Mi
my-cluster-worker         79m          1025Mi
```

*Resource usage changes over time, so these values are illustrative of our test environment.*

---

## 3. Horizontal Pod Autoscaler (HPA)

### What is HPA?

Horizontal Pod Autoscaler automatically increases or decreases the number of Pod replicas based on observed resource utilization.

For this project, HPA monitors CPU utilization of the WordPress Deployment.

### Why do we need HPA?

Without HPA, we manually change the replica count when application load increases or decreases.

With HPA:

- High CPU utilization can trigger scale-out.
- Lower CPU utilization can trigger scale-in.
- Kubernetes continuously reconciles the desired replica count.

### HPA configuration

File: `manifests/wordpress-hpa.yml`

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler

metadata:
  name: wordpress-hpa
  namespace: capstone

spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: wordpress

  minReplicas: 2
  maxReplicas: 5

  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50

  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Pods
        value: 2
        periodSeconds: 15

    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 1
        periodSeconds: 30
```

### Important configuration fields

| Field | Explanation |
|---|---|
| `scaleTargetRef` | Identifies the Deployment controlled by HPA |
| `minReplicas` | Minimum number of WordPress Pods |
| `maxReplicas` | Maximum number of WordPress Pods |
| `metrics` | Specifies the resource metric to monitor |
| `averageUtilization` | Target average CPU utilization percentage |
| `scaleUp` | Controls scale-out behavior |
| `scaleDown` | Controls scale-in behavior |
| `stabilizationWindowSeconds` | Controls how long HPA considers previous recommendations |

### Understanding CPU utilization

Our WordPress container has:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

The HPA CPU target is 50%.

CPU utilization percentage is calculated relative to the CPU request, not the CPU limit.

For a CPU request of 100m:

- 50m CPU usage = 50% utilization.
- 100m CPU usage = 100% utilization.
- 200m CPU usage = 200% utilization.

**Important:** CPU requests are essential for HPA resource utilization calculations. Without a CPU request for the relevant container, utilization may be reported as unknown.

---

## 4. HPA Scaling Formula

HPA uses the following basic formula:

```text
Desired Replicas =
ceil(Current Replicas × Current Utilization / Target Utilization)
```

Example:

```text
Current replicas       = 2
Current CPU utilization = 200%
Target CPU utilization  = 50%

Desired replicas = ceil(2 × 200 / 50)

Desired replicas = 8
```

However, our HPA has:

```yaml
maxReplicas: 5
```

Therefore, the replica count is capped at 5.

**Note:** This is the basic scaling formula. Kubernetes also considers tolerance, missing metrics, readiness, and scaling behavior when calculating the final recommendation.

---

## 5. Scale-up and Scale-down Behavior

### Scale-up configuration

```yaml
scaleUp:
  stabilizationWindowSeconds: 0
  policies:
  - type: Pods
    value: 2
    periodSeconds: 15
```

Explanation:

- No stabilization delay is configured for scale-up.
- The Pods policy allows an increase of up to 2 Pods within a 15-second period.
- Scaling is still constrained by the HPA target and maximum replica count.

### Scale-down configuration

```yaml
scaleDown:
  stabilizationWindowSeconds: 300
  policies:
  - type: Pods
    value: 1
    periodSeconds: 30
```

Explanation:

- HPA uses a 300-second stabilization window.
- This helps avoid removing Pods immediately after a temporary reduction in CPU usage.
- The configured Pods policy limits scale-down to 1 Pod per 30-second period.

**Why is scale-down stabilization useful?**

It helps prevent frequent scaling changes when application load fluctuates.

---

## 6. Practical HPA Testing

We tested automatic scaling using artificial CPU load inside the WordPress Pods.

### Step 1: Apply HPA

```bash
kubectl apply -f manifests/wordpress-hpa.yml
```

Verify:

```bash
kubectl get hpa -n capstone
```

### Step 2: Monitor CPU usage

```bash
kubectl top pods -n capstone
```

### Step 3: Monitor HPA

```bash
kubectl get hpa -n capstone --watch
```

### Step 4: Generate CPU load

We executed PHP hash loops inside the WordPress containers to generate CPU utilization.

The load processes were started inside the existing WordPress Pods, and their process IDs were saved in:

```text
/tmp/hpa-load.pid
```

### Step 5: Observe scale-out

During the load test:

```text
2 Pods → 3 Pods → 5 Pods
```

CPU utilization increased significantly above the configured 50% target.

HPA automatically increased the WordPress replica count until it reached the configured maximum of 5.

### Step 6: Stop CPU load

The load processes were stopped using:

```bash
kill "$(cat /tmp/hpa-load.pid)"
```

We verified that the CPU-intensive processes had stopped.

### Step 7: Observe scale-in

After CPU utilization returned to normal, HPA gradually reduced the number of replicas:

```text
5 Pods → 4 Pods → 3 Pods → 2 Pods
```

The scale-down stabilization behavior prevented an immediate reduction in replicas.

### Final verification

```bash
kubectl get deployment wordpress -n capstone
kubectl get hpa -n capstone
kubectl top pods -n capstone
```

Final WordPress Deployment state:

```text
READY   UP-TO-DATE   AVAILABLE
2/2     2            2
```

**Practical result:** We successfully demonstrated both automatic scale-out and scale-in using CPU metrics in our KIND cluster.

---

## 7. Important Kubernetes Concepts

### HPA vs Metrics Server

| Metrics Server | HPA |
|---|---|
| Collects resource metrics | Uses metrics for scaling decisions |
| Exposes Metrics API | Watches resource utilization |
| Does not scale Pods | Adjusts replica count |
| Provides CPU and memory data | Can scale based on configured metrics |

### HPA vs Cluster Autoscaler

| HPA | Cluster Autoscaler |
|---|---|
| Scales application Pods | Scales cluster nodes |
| Changes Deployment replica count | Changes the number of worker nodes |
| Responds to configured metrics | Responds to unschedulable Pods and node utilization policies |

**Remember:** HPA does not create additional Kubernetes nodes.

### Deployment and HPA relationship

```text
HorizontalPodAutoscaler
          |
          | Updates desired replica count
          v
      Deployment
          |
          v
       ReplicaSet
          |
          v
         Pods
```

HPA controls the Deployment's scale subresource. The Deployment and ReplicaSet manage the actual Pods.

For this project, the `replicas` field was removed from the WordPress Deployment manifest so that HPA could control the desired replica count without a competing manually configured value.

---

## 8. Troubleshooting

### Problem 1: kubectl top returns an error

Check Metrics Server:

```bash
kubectl get pods -n kube-system | grep metrics-server
```

Check APIService:

```bash
kubectl get apiservice v1beta1.metrics.k8s.io
```

Inspect logs:

```bash
kubectl logs deployment/metrics-server -n kube-system
```

### Problem 2: HPA displays unknown CPU utilization

Check:

```bash
kubectl top pods -n capstone
kubectl describe hpa wordpress-hpa -n capstone
```

Possible causes:

- Metrics Server is not ready.
- Metrics API is unavailable.
- CPU requests are missing.
- Pod metrics are not yet available.

### Problem 3: HPA does not scale up

Check:

```bash
kubectl describe hpa wordpress-hpa -n capstone
kubectl get deployment wordpress -n capstone
kubectl top pods -n capstone
```

Verify:

- CPU utilization is above the target.
- CPU requests are configured.
- Maximum replicas have not already been reached.
- HPA is targeting the correct Deployment.

### Problem 4: HPA does not immediately scale down

Check the scale-down stabilization window.

Our configuration uses:

```yaml
stabilizationWindowSeconds: 300
```

Therefore, scale-down is deliberately delayed to avoid reacting too quickly to temporary reductions in load.

---

## 9. Interview Questions and Answers

**Q1. What is Metrics Server in Kubernetes?**

Metrics Server collects CPU and memory utilization metrics from Kubernetes nodes and Pods and exposes them through the Metrics API.

**Q2. What is HPA?**

Horizontal Pod Autoscaler automatically adjusts the number of Pod replicas based on resource utilization or other configured metrics.

**Q3. Does HPA work without Metrics Server?**

For CPU and memory resource metrics, HPA generally requires a working Metrics API provider, commonly Metrics Server.

**Q4. What is the difference between CPU request and CPU limit?**

A CPU request represents the amount of CPU allocated for scheduling calculations and is the baseline for HPA CPU utilization percentages. A CPU limit defines the maximum CPU usage a container can consume.

**Q5. What happens when CPU utilization exceeds the HPA target?**

HPA calculates a desired replica count and updates the target workload's scale subresource, subject to its configured scaling limits and behavior.

**Q6. What is the purpose of `minReplicas` and `maxReplicas`?**

They define the minimum and maximum number of replicas HPA can maintain.

**Q7. What is a stabilization window?**

It is a period during which HPA considers previous scaling recommendations to avoid unnecessary or rapid scaling changes.

**Q8. Does HPA scale Kubernetes worker nodes?**

No. HPA scales application Pods. Cluster Autoscaler is responsible for adjusting the number of cluster nodes when supported and configured.

---

## 10. How to Explain in an Interview

"In my Kubernetes capstone project, I implemented Metrics Server and Horizontal Pod Autoscaler for a WordPress application running on a KIND cluster.

First, I installed Metrics Server and verified that CPU and memory metrics were available using `kubectl top nodes` and `kubectl top pods`.

Then, I configured HPA to monitor the CPU utilization of the WordPress Deployment, with a target of 50%, a minimum of 2 replicas, and a maximum of 5 replicas.

To test it, I generated artificial CPU load inside the WordPress Pods. HPA detected the increased utilization and automatically scaled the application from 2 to 5 replicas.

After stopping the load, CPU utilization decreased and HPA gradually scaled the application back down to 2 replicas.

This helped me understand the complete relationship between Metrics Server, the Metrics API, HPA, Deployments, ReplicaSets, and Pods."

---

## 11. Quick Revision Cheat Sheet

| Concept | Remember |
|---|---|
| Metrics Server | Collects CPU and memory metrics |
| Metrics API | Exposes resource metrics |
| `kubectl top` | Displays resource utilization |
| HPA | Automatically scales Pod replicas |
| `autoscaling/v2` | API version used in our HPA manifest |
| `minReplicas` | Minimum replica count |
| `maxReplicas` | Maximum replica count |
| `averageUtilization` | Target utilization percentage relative to requests |
| Scale-up | Adds replicas |
| Scale-down | Removes replicas |
| Stabilization window | Helps prevent rapid scaling changes |
| Cluster Autoscaler | Scales cluster nodes |

### Important commands

```bash
# Check Metrics Server
kubectl get deployment metrics-server -n kube-system

# Check Metrics API
kubectl get apiservice v1beta1.metrics.k8s.io

# View node metrics
kubectl top nodes

# View Pod metrics
kubectl top pods -n capstone

# Apply HPA
kubectl apply -f manifests/wordpress-hpa.yml

# Check HPA
kubectl get hpa -n capstone

# Watch HPA
kubectl get hpa -n capstone --watch

# Describe HPA
kubectl describe hpa wordpress-hpa -n capstone

# Check WordPress Deployment
kubectl get deployment wordpress -n capstone
```

---

## 12. Project Outcome

Successfully implemented and tested:

- Metrics Server v0.8.0 in KIND.
- Metrics API availability.
- CPU and memory monitoring.
- Horizontal Pod Autoscaler using `autoscaling/v2`.
- CPU-based automatic scaling.
- Scale-out from 2 to 5 replicas.
- Scale-in from 5 to 2 replicas.
- HPA scale-up and scale-down behavior.

**Status:** Metrics Server and HPA practical implementation completed and verified.