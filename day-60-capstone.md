# Day 60 Capstone – Kubernetes WordPress and MySQL

## Project Overview

A hands-on Kubernetes capstone project deploying a WordPress application with MySQL on a local KIND cluster.

The project covers Kubernetes workload management, persistent storage, shared NFS storage, resource monitoring, automatic scaling, and role-based access control.

## Technology Stack

- Kubernetes
- KIND
- Docker
- WordPress
- MySQL
- Persistent Volumes and Persistent Volume Claims
- NFS
- Metrics Server
- Horizontal Pod Autoscaler (HPA)
- Helm
- Headlamp
- Kubernetes RBAC

## Project Components

| Component | Implementation |
|---|---|
| Kubernetes cluster | KIND |
| Application | WordPress |
| Database | MySQL StatefulSet |
| Application workload | WordPress Deployment |
| Database storage | Persistent Volume and Persistent Volume Claim |
| Shared application storage | NFS |
| Resource monitoring | Metrics Server |
| Automatic scaling | Horizontal Pod Autoscaler |
| Kubernetes dashboard | Headlamp |
| Authentication | ServiceAccount tokens |
| Authorization | Kubernetes RBAC |

## Documentation

Detailed implementation and learning notes:

1. [Local NFS Server Setup for KIND](docs/local-nfs-server-docker-kind.md)
2. [Metrics Server and Horizontal Pod Autoscaler](docs/metrics-server-hpa.md)
3. [Headlamp and Kubernetes RBAC](docs/headlamp-kubernetes-rbac.md)

## Kubernetes Manifests

The `manifests/` directory contains the Kubernetes YAML resources for:

- WordPress Deployment and Service
- MySQL StatefulSet and Service
- ConfigMaps and Secrets
- NFS Persistent Volume and Persistent Volume Claim
- Horizontal Pod Autoscaler
- Headlamp ServiceAccounts and RBAC configuration

## Key Practical Achievements

- Deployed WordPress and MySQL on KIND.
- Configured persistent database storage.
- Configured shared NFS storage for WordPress uploads.
- Verified application data persistence.
- Installed Metrics Server and verified resource metrics.
- Configured CPU-based HPA.
- Successfully tested scale-out from 2 to 5 WordPress replicas.
- Successfully tested scale-in from 5 to 2 WordPress replicas.
- Installed Headlamp using Helm.
- Configured separate viewer and administrator ServiceAccounts.
- Tested RBAC permissions using `kubectl auth can-i`.
- Verified authenticated identity using `kubectl auth whoami`.

## Repository Structure

```text
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

## Security Considerations

- Kubernetes Secret manifests are excluded from Git.
- NFS runtime data and logs are excluded from Git.
- Viewer permissions follow the principle of least privilege.
- Administrator access uses a separate ServiceAccount.
- ServiceAccount tokens must never be committed to the repository.

## Project Status

Implementation and practical testing of the WordPress/MySQL application, NFS storage, Metrics Server, HPA, Headlamp, and RBAC have been completed.

Documentation and final repository verification are in progress.
