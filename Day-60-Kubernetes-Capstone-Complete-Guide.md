# Day 60 Capstone --- Kubernetes WordPress + MySQL

## Complete Project Record, Concepts, Commands, Troubleshooting and Cleanup

**Repository:**
https://github.com/Aniruddhakharve/kubernetes-wordpress-mysql-capstone\
**Branch:** `main`\
**Completion commit:** `9d0cad8049c15c3fed00a3fdb8962570a773ef13`\
**Commit message:**
`feat: complete Day 60 Kubernetes capstone with HPA and RBAC`

This guide consolidates the project's committed documentation and the
cleanup work completed after the capstone. IPs and versions are specific
to the tested lab.

------------------------------------------------------------------------

# 1. Project goal and outcome

The goal was to deploy WordPress backed by MySQL on a local Kubernetes
cluster created with KIND, then add persistent storage, shared NFS
uploads, resource metrics, automatic scaling, a dashboard, and RBAC.

**What was completed and verified:**

-   WordPress Deployment and Service deployed on KIND.
-   MySQL StatefulSet and Service deployed.
-   Persistent database storage configured with a separate RWO PVC.
-   Shared NFS storage configured for WordPress uploads using
    NFS-Ganesha.
-   Both WordPress Pods read the same file from the shared NFS export.
-   Actual WordPress media upload verified on both Pods and the NFS
    server.
-   Metrics Server v0.8.0 installed; Metrics API availability verified.
-   Node and Pod CPU/memory inspected with `kubectl top`.
-   HPA configured for 50% CPU target, minimum 2 replicas, maximum 5.
-   Scale-out from 2 to 5 replicas and scale-in from 5 to 2 replicas
    verified.
-   Headlamp installed using Helm chart 0.46.0.
-   Separate viewer and administrator ServiceAccounts configured.
-   RBAC tested with `kubectl auth can-i`.
-   Token-based identity checked using `kubectl auth whoami`.
-   Project committed and pushed to GitHub.

## Technology and concepts covered

Kubernetes, KIND, Docker, WordPress, MySQL, Namespaces, Deployment,
StatefulSet, Service, ConfigMap, Secret, PV, PVC, RWO, RWX, NFS,
NFS-Ganesha, NFSv4.1, Docker named volumes, Docker bridge networks,
Metrics Server, Metrics API, HPA, CPU requests and limits, Helm,
Headlamp, ServiceAccounts, Role, ClusterRole, RoleBinding,
ClusterRoleBinding, authentication, authorization, short-lived tokens,
kubeconfig, Git, GitHub, `.gitignore`, troubleshooting and cleanup.

------------------------------------------------------------------------

# 2. Tested environment

  Component                    Observed value
  ---------------------------- ----------------------------
  Host                         Windows 11
  Terminal                     Git Bash (MSYS) in VS Code
  Docker Desktop               4.99.0
  Docker Engine                29.9.3
  Docker context               `desktop-linux`
  Kubernetes                   KIND, server v1.37.0
  kubectl client               v1.36.1
  KIND context                 `kind-my-cluster`
  Cluster name                 `my-cluster`
  NFS-Ganesha                  v6.5
  Metrics Server               v0.8.0
  Headlamp chart/application   0.46.0
  MySQL image observed later   `mysql:8.4`
  WordPress image documented   `wordpress:latest`

## KIND and Docker network topology

The cluster contained `my-cluster-control-plane` and
`my-cluster-worker`.

  Resource                Observed address
  ----------------------- ------------------
  Docker network `kind`   `172.19.0.0/16`
  Gateway                 `172.19.0.1`
  Control plane           `172.19.0.2`
  Worker                  `172.19.0.3`
  NFS-Ganesha container   `172.19.0.10`

These addresses are specific to this lab. Inspect the network before
reusing any static IP.

------------------------------------------------------------------------

# 3. Architecture

``` text
Windows 11
└── Docker Desktop Linux environment
    ├── Docker network: kind
    │   ├── my-cluster-control-plane
    │   ├── my-cluster-worker
    │   └── nfs-server (172.19.0.10)
    │       └── Docker volume nfs-data
    └── KIND Kubernetes cluster
        ├── capstone namespace
        │   ├── WordPress Deployment + Service
        │   ├── MySQL StatefulSet + Service
        │   ├── ConfigMaps and Secret configuration
        │   ├── MySQL PVC (RWO)
        │   ├── NFS PV + WordPress PVC (RWX)
        │   └── wordpress-hpa
        ├── headlamp namespace
        │   ├── Headlamp Helm release
        │   ├── headlamp-viewer ServiceAccount + RBAC
        │   └── headlamp-admin ServiceAccount + RBAC
        └── kube-system namespace
            └── Metrics Server + Metrics API
```

## Application and storage flow

1.  Browser traffic reaches the WordPress Service.
2.  The Service routes traffic to WordPress Pods.
3.  WordPress uses MySQL for database-backed application data.
4.  MySQL data uses its own persistent claim.
5.  WordPress writes media to `/var/www/html/wp-content/uploads`.
6.  That directory is mounted from the NFS-backed PVC.
7.  Both WordPress Pods see the same uploads directory.
8.  Metrics Server collects CPU and memory metrics.
9.  HPA uses CPU metrics to adjust WordPress replica count.
10. Headlamp provides a browser-based dashboard; RBAC controls the
    permissions of its login identities.

------------------------------------------------------------------------

# 4. Kubernetes workloads and namespaces

## `capstone` namespace

  -----------------------------------------------------------------------
  Resource                            Purpose
  ----------------------------------- -----------------------------------
  Namespace                           Logical isolation for application
                                      resources

  WordPress Deployment                Manages stateless WordPress
                                      replicas

  WordPress Service                   Stable network endpoint for
                                      WordPress Pods

  MySQL StatefulSet                   Manages database Pod identity and
                                      storage association

  MySQL Service                       Stable endpoint for MySQL

  ConfigMaps                          Non-confidential configuration

  Secret                              Sensitive application/database
                                      configuration

  MySQL PVC                           Persistent database storage,
                                      observed as RWO

  NFS PV                              Represents the manually configured
                                      NFS export

  WordPress NFS PVC                   Requests shared RWX storage

  `wordpress-hpa`                     Automatically controls WordPress
                                      replica count
  -----------------------------------------------------------------------

## `headlamp` namespace

-   Headlamp application installed using Helm.
-   `headlamp-viewer` and `headlamp-admin` ServiceAccounts.
-   Viewer ClusterRole and ClusterRoleBinding.
-   Administrator ClusterRoleBinding to `cluster-admin`.

## `kube-system` namespace

-   Metrics Server Deployment and Service.
-   Metrics Server ServiceAccount and supporting RBAC.
-   APIService registration `v1beta1.metrics.k8s.io`.

## Core workload concepts

**Deployment:** Manages stateless application Pods through ReplicaSets.
Supports declarative updates and rollout management. WordPress used a
Deployment.

**StatefulSet:** Suited to workloads that need stable Pod identity and
persistent storage association. MySQL used a StatefulSet.

**Service:** Provides a stable network abstraction for a changing set of
Pods.

**Namespace:** Logically groups and scopes namespaced resources.
`capstone` and `headlamp` separated application and dashboard resources.

**ConfigMap vs Secret:** ConfigMaps hold non-sensitive configuration.
Secrets hold sensitive values. Real Secret manifests must not be
committed to a public repository. Base64 encoding is not encryption by
itself.

Useful commands:

``` bash
kubectl get namespaces
kubectl get all -n capstone
kubectl get pods -n capstone -o wide
kubectl get deployments,statefulsets,services -n capstone
kubectl describe deployment wordpress -n capstone
kubectl rollout status deployment/wordpress -n capstone
kubectl get replicasets -n capstone
kubectl get events -n capstone --sort-by=.metadata.creationTimestamp
```

------------------------------------------------------------------------

# 5. Persistent storage: PV, PVC, RWO and RWX

**PersistentVolume (PV):** Represents provisioned storage in the
cluster. It is cluster-scoped.

**PersistentVolumeClaim (PVC):** A namespaced request for storage made
by a workload.

  -----------------------------------------------------------------------
  PV                                  PVC
  ----------------------------------- -----------------------------------
  Represents storage                  Requests storage

  Cluster-scoped                      Namespaced

  Defines backend, capacity metadata, Defines requested capacity/access
  access modes, reclaim policy        modes and optional binding
  -----------------------------------------------------------------------

Access modes used/discussed:

-   **RWO (ReadWriteOnce):** read-write mounting by one node at a time,
    subject to backend implementation.
-   **ROX (ReadOnlyMany):** read-only mounting by many nodes when
    supported.
-   **RWX (ReadWriteMany):** read-write mounting by many nodes when
    supported.

MySQL storage was separate and observed as RWO. WordPress uploads used
RWX through NFS.

The NFS PV used `persistentVolumeReclaimPolicy: Retain`. Releasing the
PVC does not automatically delete the underlying NFS data. Manual
reclamation/rebinding may be required. Retain is not a backup.

**Important separation:** MySQL database files were not stored on the
WordPress NFS uploads claim. Only the uploads directory was shared;
WordPress themes/plugins and all application files were not shared by
that mount.

------------------------------------------------------------------------

# 6. NFS-Ganesha shared storage implementation

## 6.1 Why NFS?

WordPress stores media under:

``` text
/var/www/html/wp-content/uploads
```

If multiple WordPress replicas use only their individual writable
container layers, files written through one replica may not be available
through another. The project used NFS so both replicas could access the
same uploads directory.

## 6.2 Why NFS-Ganesha instead of kernel NFS server?

KIND nodes are containers, not independent VMs with separate Linux
kernels. An initial attempt to start the kernel NFS server (`nfsd`)
inside the KIND worker did not succeed: the NFS server thread count
remained zero and no server port was listening.

That attempt was cleaned up. NFS-Ganesha, a user-space NFS server, was
selected and run in a separate Docker container. The NFS client module
was later loaded in the KIND worker for the successful client mount. A
loaded `sunrpc` module alone did not mean an NFS server was running.

Cleanup commands used for the initial kernel-server attempt:

``` bash
docker exec my-cluster-worker umount /proc/fs/nfsd
docker exec my-cluster-worker modprobe -r nfsd
docker exec my-cluster-worker modprobe -r nfs
docker exec my-cluster-worker modprobe -r grace
```

## 6.3 NFS architecture and settings

  Setting                             Tested value
  ----------------------------------- --------------------------------------------
  Image                               `ghcr.io/kubernetes-sigs/nfs-ganesha:v6.5`
  Ganesha                             V6.5
  Container                           `nfs-server`
  Docker network                      `kind`
  Container IP                        `172.19.0.10`
  Export directory inside container   `/exports`
  NFSv4 pseudo path                   `/wordpress-uploads`
  Docker named volume                 `nfs-data`
  Protocol                            NFSv4.1 over TCP
  Port                                2049
  Client subnet                       `172.19.0.0/16`

The Docker named volume stores export data inside Docker's Linux
environment. Windows bind mounts were used for config and logs.

## 6.4 Inspect the image

``` bash
docker pull ghcr.io/kubernetes-sigs/nfs-ganesha:v6.5

docker image inspect ghcr.io/kubernetes-sigs/nfs-ganesha:v6.5 \
  --format 'Architecture: {{.Architecture}} | OS: {{.Os}} | Entrypoint: {{json .Config.Entrypoint}} | Cmd: {{json .Config.Cmd}}'

docker run --rm --entrypoint /usr/local/bin/ganesha.nfsd \
  ghcr.io/kubernetes-sigs/nfs-ganesha:v6.5 -v

docker run --rm --entrypoint /usr/local/bin/ganesha.nfsd \
  ghcr.io/kubernetes-sigs/nfs-ganesha:v6.5 -h
```

Observed: Linux/amd64, Ganesha V6.5, executable
`/usr/local/bin/ganesha.nfsd`, no automatic Ganesha entrypoint, and VFS
FSAL library `libfsalvfs.so`.

## 6.5 Why a Docker named volume for `/exports`?

The initial Windows bind-mounted directory appeared inside the Linux
container as a 9P filesystem. Ganesha VFS FSAL could not obtain the
required file handle and reported `Operation not supported`.

A Docker named volume provided an ext4 filesystem for `/exports`,
resolving the export-root issue. Windows bind mounts remained suitable
for config and logs.

``` bash
docker volume create nfs-data
docker volume inspect nfs-data

docker run --rm \
  -v nfs-data:/exports \
  --entrypoint sh \
  ghcr.io/kubernetes-sigs/nfs-ganesha:v6.5 \
  -c 'df -T /exports && touch /exports/volume-test.txt && ls -l /exports'
```

## 6.6 Ganesha configuration

File: `nfs-server/config/ganesha.conf`

``` conf
NFS_CORE_PARAM {
    Protocols = 4;
    allow_set_io_flusher_fail = true;
    Enable_UDP = false;
    Enable_RQUOTA = false;
}
EXPORT {
    Export_Id = 1;
    Path = /exports;
    Pseudo = /wordpress-uploads;
    Access_Type = RW;
    Squash = Root_Squash;
    SecType = sys;

    CLIENT {
        Clients = 172.19.0.0/16;
    }

    FSAL {
        Name = VFS;
    }
}
LOG {
    COMPONENTS {
        FSAL = FULL_DEBUG;
        EXPORT = FULL_DEBUG;
        NFS4 = FULL_DEBUG;
    }

    Default_Log_Level = INFO;

    Facility {
        name = FILE;
        destination = "/var/log/ganesha/ganesha.log";
        enable = active;
    }
}
```

  Setting                         Meaning
  ------------------------------- ----------------------------------------
  `Protocols = 4`                 NFSv4 protocol family
  `Enable_UDP = false`            UDP disabled; lab uses TCP
  `Enable_RQUOTA = false`         Remote quota service disabled
  `Export_Id = 1`                 Unique export ID
  `Path = /exports`               Actual server-side directory
  `Pseudo = /wordpress-uploads`   NFSv4 path presented to client
  `Access_Type = RW`              Read/write export
  `Squash = Root_Squash`          Maps remote root to anonymous identity
  `SecType = sys`                 AUTH_SYS identity-based authentication
  `Clients`                       Allowed client subnet
  `FSAL Name = VFS`               Ganesha VFS filesystem abstraction

Validate:

``` bash
docker run --rm --entrypoint /usr/local/bin/ganesha.nfsd \
  -v "$(pwd)/nfs-server/config:/etc/ganesha:ro" \
  ghcr.io/kubernetes-sigs/nfs-ganesha:v6.5 \
  -T -f /etc/ganesha/ganesha.conf
```

The parser test exited with code 0.

## 6.7 Start NFS-Ganesha

The image expected a PID file under `/usr/local/var/run/ganesha/`.
Startup failed when that directory did not exist. The final command
created it before launching Ganesha.

``` bash
docker rm -f nfs-server 2>/dev/null || true

docker run -d \
  --name nfs-server \
  --network kind \
  --ip 172.19.0.10 \
  --restart unless-stopped \
  --cap-add DAC_READ_SEARCH \
  -v nfs-data:/exports \
  -v "$(pwd)/nfs-server/config:/etc/ganesha:ro" \
  -v "$(pwd)/nfs-server/logs:/var/log/ganesha" \
  --entrypoint /bin/bash \
  ghcr.io/kubernetes-sigs/nfs-ganesha:v6.5 \
  -c 'mkdir -p /usr/local/var/run/ganesha && exec /usr/local/bin/ganesha.nfsd -F -L /dev/stderr -f /etc/ganesha/ganesha.conf'
```

Verify:

``` bash
docker ps --filter name=nfs-server
docker inspect nfs-server \
  --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
docker logs nfs-server
tail -n 100 nfs-server/logs/ganesha.log
docker exec nfs-server sh -c 'grep -i ":0801" /proc/net/tcp /proc/net/tcp6'
```

`0801` is hexadecimal for TCP port 2049.

The first NFS client mount attempt failed with
`Operation not permitted`. In this setup, VFS FSAL file-handle
operations needed `CAP_DAC_READ_SEARCH`. Adding
`--cap-add DAC_READ_SEARCH` resolved the issue; `--privileged` was not
required.

## 6.8 Mount from KIND worker

The worker had `/usr/sbin/mount.nfs` (util-linux 2.41.5 observed).

``` bash
docker exec my-cluster-worker modprobe nfs
docker exec my-cluster-worker mkdir -p /mnt/nfs-uploads-test

docker exec my-cluster-worker mount -v \
  -t nfs4 \
  -o vers=4.1,proto=tcp \
  172.19.0.10:/wordpress-uploads \
  /mnt/nfs-uploads-test

docker exec my-cluster-worker findmnt /mnt/nfs-uploads-test
```

Observed source/options included `172.19.0.10:/wordpress-uploads`,
`nfs4`, `rw`, `vers=4.1`, and `proto=tcp`.

A temporary pseudo-root mount at `/mnt/nfs-root` was also used for
diagnosis. The direct export mount was the one relevant to the PV. These
manual mounts were diagnostic; Kubernetes/kubelet manages the mount used
by the Pod. Do not unmount a filesystem while a Pod is using its PVC.

## 6.9 Root Squash and UID/GID permissions

Initially `/exports` was `root:root`, mode `755`. With `Root_Squash`,
remote root mapped to anonymous UID/GID `4294967294:4294967294`. The
client could list but could not write.

The WordPress web process was checked:

``` bash
kubectl exec -n capstone <wordpress-pod-name> -- id www-data
```

Observed:

``` text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

The final lab export ownership was changed to WordPress UID/GID 33:

``` bash
docker exec nfs-server chown 33:33 /exports
docker exec nfs-server chmod 755 /exports
docker exec nfs-server stat -c 'UID=%u GID=%g MODE=%a PERMISSIONS=%A' /exports
```

Observed: `UID=33 GID=33 MODE=755`. `Root_Squash` remained enabled and
WordPress could create uploads.

**Production note:** This is a lab arrangement, not a complete
production NFS security design. Production needs deliberate identity
mapping, network restrictions, access controls, and backups.

## 6.10 NFS PV

File: `manifests/nfs-pv.yml`

``` yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: wordpress-nfs-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs-manual
  mountOptions:
    - proto=tcp
    - nfsvers=4.1
  nfs:
    server: 172.19.0.10
    path: /wordpress-uploads
```

Apply and inspect:

``` bash
kubectl apply -f manifests/nfs-pv.yml
kubectl get pv wordpress-nfs-pv
kubectl describe pv wordpress-nfs-pv
```

`nfs-manual` is a manually provisioned StorageClass name, not a dynamic
provisioner. The PV advertises 1Gi as Kubernetes capacity metadata; it
is not a hard quota on the NFS export.

## 6.11 NFS PVC

File: `manifests/nfs-pvc.yml`

``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-nfs-pvc
  namespace: capstone
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-manual
  resources:
    requests:
      storage: 1Gi
  volumeName: wordpress-nfs-pv
```

Apply and inspect:

``` bash
kubectl apply -f manifests/nfs-pvc.yml
kubectl get pvc -n capstone
kubectl describe pvc wordpress-nfs-pvc -n capstone
```

`volumeName` explicitly binds the claim to `wordpress-nfs-pv`.

## 6.12 Mount the claim into WordPress

The Deployment mounted the PVC only at the uploads path:

``` yaml
volumeMounts:
  - name: wordpress-uploads
    mountPath: /var/www/html/wp-content/uploads
volumes:
  - name: wordpress-uploads
    persistentVolumeClaim:
      claimName: wordpress-nfs-pvc
```

Apply and verify:

``` bash
kubectl apply -f manifests/wordpress-deployment.yml
kubectl rollout status deployment/wordpress -n capstone
kubectl get pods -n capstone -o wide
kubectl describe pod -n capstone <wordpress-pod-name>
kubectl exec -n capstone <wordpress-pod-name> -- \
  df -h /var/www/html/wp-content/uploads
```

## 6.13 NFS verification

Worker-to-server:

``` bash
docker exec my-cluster-worker sh -c \
  'echo "Created from KIND worker through NFS" > /mnt/nfs-uploads-test/kind-nfs-test.txt'
docker exec nfs-server cat /exports/kind-nfs-test.txt
```

Pod 1 → Pod 2 → NFS server:

``` bash
kubectl exec -n capstone <wordpress-pod-1> -- \
  su -s /bin/sh www-data -c \
  'echo "NFS shared storage test from WordPress Pod 1" > /var/www/html/wp-content/uploads/nfs-pod1-test.txt'

kubectl exec -n capstone <wordpress-pod-1> -- \
  cat /var/www/html/wp-content/uploads/nfs-pod1-test.txt

kubectl exec -n capstone <wordpress-pod-2> -- \
  cat /var/www/html/wp-content/uploads/nfs-pod1-test.txt

docker exec nfs-server cat /exports/nfs-pod1-test.txt
```

All locations returned the same content.

An actual image was also uploaded through the WordPress Admin Dashboard.
Original and resized variants appeared under a year/month path such as
`2026/11/` and were visible in both WordPress Pods and the NFS server.

``` bash
kubectl exec -n capstone <wordpress-pod> -- \
  find /var/www/html/wp-content/uploads -maxdepth 4 -type f -print

docker exec nfs-server \
  find /exports -maxdepth 4 -type f -print
```

A non-recursive `ls` of the uploads root may show only the year
directory. Use `find` to inspect nested files.

------------------------------------------------------------------------

# 7. Metrics Server

Metrics Server collects CPU and memory metrics from nodes and containers
through the kubelet API and exposes them through the Kubernetes Metrics
API. HPA can use these metrics.

The project used Metrics Server v0.8.0 and kept:

  File                                    Purpose
  --------------------------------------- --------------------------
  `metrics-server/components.yaml`        Upstream manifest
  `metrics-server/components-kind.yaml`   KIND-compatible manifest

The KIND-compatible manifest included:

``` yaml
- --kubelet-insecure-tls
```

This was a lab workaround because kubelet certificate verification in
the local KIND environment was not configured in a way Metrics Server
accepted. It disables kubelet certificate verification and is not a
general production recommendation. Production clusters should use
trusted kubelet certificates.

Verification:

``` bash
kubectl get deployment metrics-server -n kube-system
kubectl get pods -n kube-system | grep metrics-server
kubectl get apiservice v1beta1.metrics.k8s.io
kubectl top nodes
kubectl top pods -n capstone
```

The APIService was verified as `AVAILABLE True`. Example point-in-time
node metrics from the lab:

``` text
NAME                      CPU(cores)   MEMORY(bytes)
my-cluster-control-plane  196m         679Mi
my-cluster-worker         79m          1025Mi
```

Metrics Server collects metrics; it does not scale Pods.

------------------------------------------------------------------------

# 8. Horizontal Pod Autoscaler

## 8.1 HPA concept

HPA automatically adjusts the number of Pod replicas based on observed
metrics. In this project it monitored WordPress CPU utilization.

-   Higher CPU can trigger scale-out.
-   Lower CPU can trigger scale-in.
-   HPA reconciles the desired replica count.

## 8.2 HPA manifest

File: `manifests/wordpress-hpa.yml`

``` yaml
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

  Field                          Meaning
  ------------------------------ -----------------------------------
  `scaleTargetRef`               Deployment controlled by HPA
  `minReplicas`                  Minimum 2
  `maxReplicas`                  Maximum 5
  `averageUtilization`           CPU target 50%
  `scaleUp`                      Scale-out behavior
  `scaleDown`                    Scale-in behavior
  `stabilizationWindowSeconds`   Helps avoid rapid scaling changes

## 8.3 CPU requests and limits

The documented WordPress container resources were:

``` yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

HPA CPU utilization is calculated relative to CPU request, not CPU
limit.

For a 100m CPU request: - 50m usage = 50%. - 100m usage = 100%. - 200m
usage = 200%.

CPU requests are essential for HPA utilization calculations. Without a
request for the relevant container, utilization can be unknown.

## 8.4 Basic HPA formula

``` text
Desired Replicas =
ceil(Current Replicas × Current Utilization / Target Utilization)
```

Example: 2 replicas, 200% current utilization, 50% target:

``` text
ceil(2 × 200 / 50) = 8
```

But this project capped replicas at 5. Kubernetes also considers
tolerance, missing metrics, readiness, and scaling behavior.

## 8.5 Scaling behavior

Scale-up had a zero-second stabilization window and a policy permitting
up to 2 additional Pods per 15-second period.

Scale-down used a 300-second stabilization window and a policy limiting
removal to 1 Pod per 30-second period. This helps avoid rapid scale-in
when load briefly drops.

The `replicas` field was removed from the WordPress Deployment manifest
so HPA could control the desired replica count without a competing
manually configured value.

## 8.6 Practical load test

Artificial CPU load was generated inside WordPress containers using PHP
hash loops. Process IDs were saved in `/tmp/hpa-load.pid`.

``` bash
kubectl apply -f manifests/wordpress-hpa.yml
kubectl get hpa -n capstone
kubectl top pods -n capstone
kubectl get hpa -n capstone --watch
```

During load, the Deployment scaled:

``` text
2 Pods → 3 Pods → 5 Pods
```

The load processes were stopped using:

``` bash
kill "$(cat /tmp/hpa-load.pid)"
```

After CPU utilization returned to normal, HPA scaled down gradually:

``` text
5 Pods → 4 Pods → 3 Pods → 2 Pods
```

Final state:

``` text
READY   UP-TO-DATE   AVAILABLE
2/2     2            2
```

## 8.7 HPA vs Metrics Server vs Cluster Autoscaler

  -----------------------------------------------------------------------
  Component                           Responsibility
  ----------------------------------- -----------------------------------
  Metrics Server                      Collects and exposes CPU/memory
                                      metrics

  HPA                                 Scales application Pod replicas

  Cluster Autoscaler                  Scales Kubernetes nodes when
                                      configured and supported
  -----------------------------------------------------------------------

HPA does not create worker nodes.

Useful commands:

``` bash
kubectl get apiservice v1beta1.metrics.k8s.io
kubectl top nodes
kubectl top pods -n capstone
kubectl get hpa -n capstone
kubectl get hpa -n capstone --watch
kubectl describe hpa wordpress-hpa -n capstone
```

------------------------------------------------------------------------

# 9. Helm and Headlamp

Headlamp is a web-based Kubernetes dashboard. The project installed it
in the `headlamp` namespace using Helm chart version 0.46.0.

## Helm concepts used

-   **Repository:** source of charts.
-   **Chart:** package of Kubernetes templates and default values.
-   **Values:** configuration used to customize a chart.
-   **`helm template`:** renders chart templates locally.
-   **Release:** installed chart instance.
-   **`helm install`:** installs the release.

## Installation workflow

``` bash
helm repo add headlamp https://kubernetes-sigs.github.io/headlamp/
helm repo update
helm search repo headlamp
helm show values headlamp/headlamp
```

The chart could create a ClusterRoleBinding with `cluster-admin` by
default. This was disabled so the project could create separate viewer
and administrator RBAC:

``` yaml
clusterRoleBinding:
  create: false
```

Render before install:

``` bash
helm template headlamp headlamp/headlamp \
  --version 0.46.0 \
  --namespace headlamp \
  --set clusterRoleBinding.create=false \
  > /tmp/headlamp-rendered.yaml
```

Install:

``` bash
helm install headlamp headlamp/headlamp \
  --version 0.46.0 \
  --namespace headlamp \
  --create-namespace \
  --set clusterRoleBinding.create=false \
  --set config.unsafeUseServiceAccountToken=false
```

Verify:

``` bash
helm list -n headlamp
kubectl get all -n headlamp
kubectl get serviceaccount -n headlamp
```

Headlamp Pod was Running, Deployment was 1/1 available, and Service was
ClusterIP.

Access locally:

``` bash
kubectl port-forward -n headlamp svc/headlamp 8080:80
```

Open `http://localhost:8080`. Port forwarding provides local access; it
does not expose the dashboard publicly.

------------------------------------------------------------------------

# 10. Kubernetes RBAC

## 10.1 Four RBAC resources

  -----------------------------------------------------------------------
  Resource                            Purpose
  ----------------------------------- -----------------------------------
  Role                                Defines permissions within a
                                      namespace

  ClusterRole                         Defines permissions at cluster
                                      scope or reusable permissions

  RoleBinding                         Grants a Role or ClusterRole within
                                      a namespace

  ClusterRoleBinding                  Grants a ClusterRole across the
                                      cluster
  -----------------------------------------------------------------------

A Role/ClusterRole defines permissions. A binding grants them to a
subject.

## 10.2 ServiceAccounts

A ServiceAccount provides an identity for a process or client
interacting with the Kubernetes API. Creating a ServiceAccount does not
automatically grant it permissions.

The project created:

-   `headlamp-viewer`: read-only cluster access.
-   `headlamp-admin`: administrative cluster access.

## 10.3 Viewer ServiceAccount

File: `manifests/headlamp-viewer-sa.yml`

``` yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: headlamp-viewer
  namespace: headlamp
```

``` bash
kubectl apply -f manifests/headlamp-viewer-sa.yml
kubectl get serviceaccount -n headlamp
```

## 10.4 Viewer ClusterRole

File: `manifests/headlamp-viewer-role.yml`

``` yaml
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

The empty API group `""` represents the core API group.

  Verb       Meaning
  ---------- --------------------------
  `get`      Read a specific resource
  `list`     List resources
  `watch`    Watch resource changes
  `create`   Create
  `update`   Update
  `patch`    Modify selected fields
  `delete`   Delete

The viewer was granted only `get`, `list`, and `watch`.

``` bash
kubectl apply -f manifests/headlamp-viewer-role.yml
kubectl describe clusterrole headlamp-viewer
```

## 10.5 Viewer ClusterRoleBinding

File: `manifests/headlamp-viewer-binding.yml`

``` yaml
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

ClusterRoleBinding was used because the viewer needed to inspect
cluster-wide resources such as Nodes, Namespaces, and workloads across
namespaces.

``` bash
kubectl apply -f manifests/headlamp-viewer-binding.yml
kubectl get clusterrolebinding headlamp-viewer-binding
kubectl describe clusterrolebinding headlamp-viewer-binding
```

## 10.6 Viewer permission tests

``` bash
kubectl auth can-i list pods --all-namespaces \
  --as=system:serviceaccount:headlamp:headlamp-viewer

kubectl auth can-i delete pods -n capstone \
  --as=system:serviceaccount:headlamp:headlamp-viewer

kubectl auth can-i get secrets -n capstone \
  --as=system:serviceaccount:headlamp:headlamp-viewer
```

  Operation             Viewer result
  --------------------- ---------------
  List Pods             Yes
  Read Deployments      Yes
  Watch resources       Yes
  Delete Pods           No
  Create Deployments    No
  Patch Deployments     No
  Delete StatefulSets   No
  Read Secrets          No

Deployment creation, deployment patching, and StatefulSet deletion were
also tested and denied.

## 10.7 Administrator ServiceAccount and ClusterRoleBinding

File: `manifests/headlamp-admin.yml`

``` yaml
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

This binds the administrator ServiceAccount to the built-in
`cluster-admin` ClusterRole. It grants broad administrative permissions,
including creating, modifying, deleting resources and accessing Secrets.

**Security warning:** `cluster-admin` is highly privileged. Protect its
credentials; do not use it for normal dashboard viewing.

``` bash
kubectl apply -f manifests/headlamp-admin.yml
kubectl get serviceaccount headlamp-admin -n headlamp
kubectl get clusterrolebinding headlamp-admin-binding
kubectl describe clusterrolebinding headlamp-admin-binding
```

## 10.8 Administrator permission tests

``` bash
kubectl auth can-i list pods --all-namespaces \
  --as=system:serviceaccount:headlamp:headlamp-admin

kubectl auth can-i delete pods -n capstone \
  --as=system:serviceaccount:headlamp:headlamp-admin

kubectl auth can-i get secrets -n capstone \
  --as=system:serviceaccount:headlamp:headlamp-admin
```

The administrator identity was allowed to list Pods, delete Pods, read
Secrets, create resources, and modify resources.

## 10.9 Authentication and authorization

  Authentication                       Authorization
  ------------------------------------ ------------------------------
  Verifies identity                    Checks permissions
  Who are you?                         What can you do?
  Uses token/certificate credentials   Uses RBAC policies
  Identifies ServiceAccount            Evaluates allowed operations

## 10.10 ServiceAccount tokens

``` bash
kubectl create token headlamp-viewer -n headlamp --duration=1h
kubectl create token headlamp-admin -n headlamp --duration=1h
```

Tokens can be entered into Headlamp's login page. They are credentials:
never publish them in GitHub, documentation, screenshots, chat, or
committed scripts.

During the exercise, part of an administrator token was accidentally
exposed. It was treated as compromised; the administrator ServiceAccount
was recreated and a fresh token generated. Lesson: once exposed, stop
trusting the credential and rotate/recreate as appropriate.

## 10.11 `kubectl auth whoami` and kubeconfig troubleshooting

``` bash
kubectl auth whoami
```

This displays the identity authenticated by the Kubernetes API server.

The original kubeconfig used client-certificate authentication. Passing
a token through an initial command still showed
`Username: kubernetes-admin`, because the active kubeconfig's
certificate identity was still being used.

The successful test used an isolated temporary kubeconfig:

``` bash
TOKEN="$(kubectl create token headlamp-admin -n headlamp --duration=10m)"

export KUBECONFIG="$(cygpath -w /tmp/headlamp-token-test-config)"

kubectl config set-credentials headlamp-token-user --token="$TOKEN"
kubectl config set-context kind-my-cluster --user=headlamp-token-user
kubectl auth whoami
```

Successful identity:

``` text
Username: system:serviceaccount:headlamp:headlamp-admin
```

Temporary environment cleanup:

``` bash
unset TOKEN
rm /tmp/headlamp-token-test-config
unset KUBECONFIG
kubectl config current-context
```

The original context was `kind-my-cluster`. Lesson: use a separate
kubeconfig for temporary authentication tests instead of modifying the
primary kubeconfig.

Headlamp login was verified with both identities: viewer (read-only) and
admin (cluster administrator).

------------------------------------------------------------------------

# 11. Troubleshooting lessons

## NFS

  -----------------------------------------------------------------------------------------
  Problem                      Cause                   Resolution
  ---------------------------- ----------------------- ------------------------------------
  Kernel `nfsd` did not start  Thread count zero; no   Cleaned up kernel attempt; used
  in KIND worker               server port             NFS-Ganesha

  Ganesha PID open failed      Runtime directory       Created `/usr/local/var/run/ganesha`
                               missing                 before startup

  `PR_SET_IO_FLUSHER` EPERM    Optional                `allow_set_io_flusher_fail = true`
                               process-setting         
                               privilege unavailable   

  VFS FSAL                     Windows bind mount      Docker named volume (ext4) for
  `Operation not supported`,   unsuitable for VFS file `/exports`
  type 9P                      handle                  

  UDP/RQUOTA registration      Not needed in TCP-only  Disabled UDP and RQUOTA
  errors                       NFSv4 lab               

  NFS mount                    Missing                 Added `--cap-add DAC_READ_SEARCH`
  `Operation not permitted`    `CAP_DAC_READ_SEARCH`   

  Root-squashed client could   Export root             Investigated anonymous UID/GID;
  list but not write           ownership/mode mismatch final ownership `33:33`

  WordPress `www-data` could   WordPress UID 33        Export root owned by UID/GID 33,
  not write                    differed from anonymous mode 755
                               UID                     

  Image absent from top-level  WordPress stores media  Used recursive `find`
  listing                      in year/month           
                               directories             
  -----------------------------------------------------------------------------------------

Useful checks:

``` bash
docker logs nfs-server
tail -n 100 nfs-server/logs/ganesha.log
docker exec nfs-server capsh --print
docker exec nfs-server stat -c 'UID=%u GID=%g MODE=%a PERMISSIONS=%A' /exports
kubectl exec -n capstone <wordpress-pod> -- id www-data
kubectl exec -n capstone <wordpress-pod> -- \
  find /var/www/html/wp-content/uploads -maxdepth 4 -type f -print
```

## Metrics Server and HPA

If `kubectl top` fails:

``` bash
kubectl get pods -n kube-system | grep metrics-server
kubectl get apiservice v1beta1.metrics.k8s.io
kubectl logs deployment/metrics-server -n kube-system
```

If HPA shows unknown CPU:

``` bash
kubectl top pods -n capstone
kubectl describe hpa wordpress-hpa -n capstone
```

Possible causes: Metrics Server not ready, Metrics API unavailable,
missing CPU requests, or metrics not yet available.

If HPA does not scale up, verify CPU is above target, CPU requests
exist, max replicas have not been reached, and HPA targets the correct
Deployment.

Scale-down delay is expected because the project configured a 300-second
stabilization window.

## Headlamp/RBAC

If ServiceAccount is missing:

``` bash
kubectl get serviceaccounts -n headlamp
```

If viewer permissions fail:

``` bash
kubectl describe clusterrole headlamp-viewer
kubectl describe clusterrolebinding headlamp-viewer-binding
kubectl auth can-i list pods --all-namespaces \
  --as=system:serviceaccount:headlamp:headlamp-viewer
```

If Headlamp login fails:

``` bash
kubectl get pods -n headlamp
kubectl get service -n headlamp
kubectl logs deployment/headlamp -n headlamp
```

Also verify token validity, the selected ServiceAccount, RBAC
permissions, and port-forward session.

If `kubectl auth whoami` shows an unexpected identity:

``` bash
kubectl config current-context
kubectl config view --minify
```

Check for client-certificate authentication and use an isolated
temporary kubeconfig for token testing.

------------------------------------------------------------------------

# 12. Security and production considerations

1.  Follow least privilege.
2.  Do not give every dashboard user `cluster-admin`.
3.  Use separate viewer and administrator identities.
4.  Avoid granting read access to Secrets unless required.
5.  Use short-lived ServiceAccount tokens.
6.  Never commit tokens or kubeconfig credentials.
7.  Treat exposed credentials as compromised.
8.  Prefer namespace-scoped access where cluster-wide access is
    unnecessary.
9.  ConfigMaps can contain sensitive configuration; viewer read access
    should be considered carefully.
10. `--kubelet-insecure-tls` was a lab-only workaround.
11. NFS AUTH_SYS and broad lab subnet access were not a complete
    production security posture.
12. `cluster-admin` is highly privileged.
13. Docker volumes are persistent storage, not backups.
14. The single NFS container is a single point of failure.
15. Pin tested image versions/digests in production instead of using
    `latest`.

Lab limitations documented in the project:

-   NFS container is a single point of failure.
-   Docker volume is local to Docker Desktop.
-   AUTH_SYS and `172.19.0.0/16` client allowance are lab choices.
-   PV capacity metadata of 1Gi is not an enforced quota on the Docker
    volume.
-   Only uploads are shared; themes and plugins are not.
-   WordPress used `wordpress:latest`; production should pin a tested
    version/digest.
-   Important data needs separate backups.

Production alternatives noted: Amazon EFS for managed shared NFS on AWS,
object storage such as Amazon S3 with a suitable WordPress media
integration, and immutable application images/CI/CD for themes and
plugins.

------------------------------------------------------------------------

# 13. GitHub repository and file hygiene

Repository:

``` text
https://github.com/Aniruddhakharve/kubernetes-wordpress-mysql-capstone
```

Branch: `main`

Completion commit:

``` text
9d0cad8049c15c3fed00a3fdb8962570a773ef13
```

Commit message:

``` text
feat: complete Day 60 Kubernetes capstone with HPA and RBAC
```

The completion commit was pushed to `origin/main`. The working tree was
clean at final verification. The commit contained 11 files and 2,185
insertions. Screenshots were not saved.

Repository structure documented in the project:

``` text
kubernetes-wordpress-mysql-capstone/
├── docs/
│   ├── local-nfs-server-docker-kind.md
│   ├── metrics-server-hpa.md
│   └── headlamp-kubernetes-rbac.md
├── helm-comparison/
├── manifests/
├── metrics-server/
├── nfs-server/
├── kind-config.yml
├── day-60-capstone.md
└── README.md
```

`manifests/` contains YAML for WordPress Deployment/Service, MySQL
StatefulSet/Service, ConfigMaps/Secrets, NFS PV/PVC, HPA, and Headlamp
ServiceAccounts/RBAC.

The project `.gitignore` contained:

``` gitignore
# Kubernetes secrets
manifests/mysql-secret.yml

# NFS runtime data and logs
nfs-server/data/
nfs-server/logs/
```

The local `nfs-server/` directory was deliberately preserved after the
project so its configuration, data artifact, and logs could be inspected
in future learning. Do not delete that directory as part of Docker
cleanup.

------------------------------------------------------------------------

# 14. Final cleanup after Day 60

The runtime infrastructure was removed after implementation/testing,
while the local Git repository and project files were preserved.

## KIND cluster

Command run:

``` bash
kind delete cluster --name my-cluster
```

Control-plane and worker containers were deleted.

Verification:

``` bash
kind get clusters
docker ps -a --filter name=my-cluster
```

No KIND cluster remained under that name and no matching node containers
remained.

## NFS container and volume

``` bash
docker rm -f nfs-server
docker volume rm nfs-data
```

The NFS-Ganesha container and named volume were removed. Removing
`nfs-data` permanently deleted the NFS runtime export data stored in
that Docker volume.

**The local `nfs-server/` directory was preserved.** Its config, data
artifact, and logs remained in the repository working directory.

## OpenClaw and unrelated Docker resources --- protected

The user explicitly required that nothing related to OpenClaw be
deleted, pruned, or modified.

All four volumes were preserved:

``` text
openclaw-data
openclaw_mariadb-data
openclaw_openclaw-data
openclaw_openclaw-logs
```

Also left untouched:

-   OpenClaw containers, images, networks, and data.
-   `openclaw_default` network.
-   `minikube` network.
-   `kind` network (left in place after confirming no containers were
    attached).

## Images deliberately retained

The user chose to keep:

-   `kindest/node` image (observed ID `a1ed56cfb0e7`, approximately 1.34
    GB).
-   `mysql:8.4` image (observed ID `a0225d1ea663`, approximately 1.12
    GB).

The NFS-Ganesha image was removed:

``` text
ghcr.io/kubernetes-sigs/nfs-ganesha:v6.5
```

Build cache was already `0B`; no build cache needed removal.

## Final Docker inventory observed

Images:

``` text
mysql:8.4
<none>  (kindest/node image)
```

The untagged image was identified as:

``` text
kindest/node@sha256:a1ed56cfb0e7b93589bdf97c8cd566405a265939e3620fc4f5de89adff580ae5
```

Containers: none.

Volumes:

``` text
openclaw-data
openclaw_mariadb-data
openclaw_openclaw-data
openclaw_openclaw-logs
```

Build cache: `0B`.

## WSL/Docker Desktop investigation

Only one WSL distribution was registered:

``` text
docker-desktop
```

No general Ubuntu WSL distro was registered. Docker Desktop WSL kernel
observed:

``` text
6.18.58.1-microsoft-standard-WSL2
```

Read-only checks found `sunrpc` loaded, but no evidence of loaded NFS
client/server modules or an NFS filesystem entry. `modules.builtin`
search returned no NFS entry. `modules.builtin.modinfo` returned:

``` text
overlay.parm=nfs_export:Default to on or off for the NFS export feature
overlay.parmtype=nfs_export:bool
```

This is OverlayFS metadata, not proof that NFS client/server support is
available. No kernel modules were unloaded, Docker Desktop kernel files
were not modified, and the `docker-desktop` WSL distro was not
unregistered. The decision was to leave Docker Desktop's managed WSL
environment unchanged.

------------------------------------------------------------------------

# 15. Command quick reference

## KIND

``` bash
kind create cluster --name my-cluster --config kind-config.yml
kind get clusters
kubectl cluster-info --context kind-my-cluster
kubectl get nodes -o wide
kind delete cluster --name my-cluster
```

## Kubernetes inspection

``` bash
kubectl config current-context
kubectl get namespaces
kubectl get all -n capstone
kubectl get pods -n capstone -o wide
kubectl describe pod -n capstone <pod-name>
kubectl logs -n capstone <pod-name>
kubectl exec -n capstone <pod-name> -- <command>
kubectl get events -n capstone --sort-by=.metadata.creationTimestamp
```

## Storage

``` bash
kubectl get pv
kubectl get pvc -n capstone
kubectl describe pv wordpress-nfs-pv
kubectl describe pvc wordpress-nfs-pvc -n capstone
```

## Docker and NFS

``` bash
docker ps -a
docker image ls -a
docker volume ls
docker network ls
docker network inspect kind
docker volume inspect nfs-data
docker logs nfs-server
```

## Metrics and HPA

``` bash
kubectl get apiservice v1beta1.metrics.k8s.io
kubectl top nodes
kubectl top pods -n capstone
kubectl get hpa -n capstone
kubectl get hpa -n capstone --watch
kubectl describe hpa wordpress-hpa -n capstone
```

## Helm and Headlamp

``` bash
helm repo add headlamp https://kubernetes-sigs.github.io/headlamp/
helm repo update
helm search repo headlamp
helm show values headlamp/headlamp
helm list -n headlamp
kubectl get all -n headlamp
kubectl port-forward -n headlamp svc/headlamp 8080:80
```

## RBAC and identity

``` bash
kubectl get serviceaccounts -n headlamp
kubectl get clusterroles
kubectl get clusterrolebindings

kubectl auth can-i list pods --all-namespaces \
  --as=system:serviceaccount:headlamp:headlamp-viewer

kubectl auth can-i delete pods -n capstone \
  --as=system:serviceaccount:headlamp:headlamp-viewer

kubectl auth can-i get secrets -n capstone \
  --as=system:serviceaccount:headlamp:headlamp-admin

kubectl create token headlamp-viewer -n headlamp --duration=1h
kubectl auth whoami
kubectl config current-context
```

------------------------------------------------------------------------

# 16. Interview revision

**Why KIND?**\
KIND creates a local Kubernetes cluster using Docker containers,
allowing Kubernetes practice without a paid managed cluster.

**Why StatefulSet for MySQL?**\
StatefulSet is suited to workloads needing stable identity and
persistent storage association.

**Why NFS for WordPress uploads?**\
Multiple WordPress replicas need access to the same media files. NFS
provides shared RWX storage.

**PV vs PVC?**\
PV represents provisioned storage; PVC is a namespaced request for
storage.

**RWO vs RWX?**\
RWO is read-write mounting by one node at a time, subject to backend
behavior. RWX allows multiple nodes to mount read-write when supported.

**Why NFS-Ganesha?**\
The kernel NFS server did not start successfully inside the KIND worker
environment. NFS-Ganesha is a user-space NFS server and worked in a
separate Docker container.

**Why Docker named volume for export data?**\
The Windows bind mount appeared as 9P and did not support the VFS
file-handle operation Ganesha needed. Docker named volume provided ext4.

**What is Metrics Server?**\
It collects CPU/memory usage from nodes and Pods and exposes it through
the Metrics API.

**What is HPA?**\
HPA adjusts Pod replica count based on configured metrics. This project
used CPU utilization for WordPress.

**HPA vs Cluster Autoscaler?**\
HPA scales Pods; Cluster Autoscaler adjusts cluster nodes when
configured and supported.

**Why CPU requests for HPA?**\
CPU utilization percentage is calculated relative to CPU requests.

**What is Helm?**\
Helm is a package manager for Kubernetes. It installs and manages
applications packaged as charts.

**What is Headlamp?**\
A web-based Kubernetes dashboard for viewing and managing cluster
resources.

**What is RBAC?**\
Kubernetes authorization that controls which operations identities can
perform on resources.

**ServiceAccount vs ClusterRole?**\
ServiceAccount represents an identity; ClusterRole defines permissions.
A binding connects them.

**Authentication vs authorization?**\
Authentication verifies identity; authorization checks what that
identity may do.

**Least privilege?**\
Give each identity only the minimum permissions needed for its task.

**How did you verify RBAC?**\
Used `kubectl auth can-i` to test allowed and denied operations for
viewer and administrator ServiceAccounts.

**How did you verify token identity?**\
Used `kubectl auth whoami` with an isolated temporary kubeconfig so the
original client certificate did not continue authenticating the request.

------------------------------------------------------------------------

# 17. Final completion checklist

-   [x] KIND cluster created and used.
-   [x] WordPress Deployment and Service deployed.
-   [x] MySQL StatefulSet and Service deployed.
-   [x] ConfigMaps and Secret configuration used.
-   [x] MySQL persistent storage configured.
-   [x] NFS-Ganesha v6.5 container started.
-   [x] NFSv4.1/TCP 2049 export configured.
-   [x] KIND worker mounted the NFS export.
-   [x] NFS PV and PVC reached Bound.
-   [x] Both WordPress Pods mounted shared uploads.
-   [x] Pod-to-Pod shared file test passed.
-   [x] Actual WordPress media upload verified on both Pods and NFS
    server.
-   [x] Metrics Server v0.8.0 installed and Metrics API verified.
-   [x] CPU and memory metrics checked.
-   [x] HPA configured and scale-out/scale-in tested.
-   [x] Headlamp installed using Helm.
-   [x] Viewer and administrator ServiceAccounts created.
-   [x] RBAC permission tests completed.
-   [x] Token authentication and `kubectl auth whoami` verified.
-   [x] Headlamp login tested with both identities.
-   [x] Project pushed to GitHub.
-   [x] KIND cluster deleted after completion.
-   [x] NFS container and runtime volume removed.
-   [x] Local repo and entire `nfs-server/` directory preserved.
-   [x] All OpenClaw resources preserved.
-   [x] `kindest/node` and `mysql:8.4` images retained.
-   [x] Docker build cache confirmed as 0B.

------------------------------------------------------------------------

# 18. Final outcome and next milestone

Day 60 combined application deployment, stateful database workload,
persistent storage, shared NFS storage, resource monitoring,
autoscaling, dashboard installation, RBAC, authentication, GitHub
documentation, troubleshooting, and scoped cleanup.

The key practical lessons:

1.  Use Deployment for stateless application replicas and StatefulSet
    for stateful workloads that need stable identity/storage
    association.
2.  Container writable layers are not a substitute for persistent data.
3.  Replicas that need the same uploaded files require shared storage or
    another shared-media design.
4.  NFS networking, filesystem behavior, UID/GID ownership, and root
    squashing all matter; a successful mount does not automatically mean
    the application can write.
5.  Metrics Server supplies metrics; HPA consumes them to scale Pods.
6.  CPU requests are important for HPA utilization percentages.
7.  Inspect Helm chart values and rendered manifests before
    installation.
8.  ServiceAccounts represent identities; RBAC bindings grant
    permissions.
9.  Authentication and authorization are separate.
10. Protect tokens, kubeconfigs, Secrets, and administrative
    permissions.
11. Scope cleanup carefully so unrelated data and reusable resources are
    preserved.

**Day 60 status: Completed.**\
**Next milestone: Day 61 --- Terraform, Day 1.**
