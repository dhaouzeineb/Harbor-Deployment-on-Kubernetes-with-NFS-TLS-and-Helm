# Harbor Deployment on Kubernetes with NFS, TLS and Helm

![Kubernetes](https://img.shields.io/badge/Kubernetes-Cluster-326CE5?logo=kubernetes&logoColor=white)
![Helm](https://img.shields.io/badge/Helm-Deployment-0F1689?logo=helm&logoColor=white)
![Storage](https://img.shields.io/badge/Storage-NFS-orange)
![Security](https://img.shields.io/badge/Security-TLS-green)

## 1. Overview

This project provides a step-by-step guide to deploying **Harbor**, an open-source container image registry, on an existing Kubernetes cluster.

The deployment covers the complete installation and configuration process, including NFS storage, Helm, Ingress, TLS certificates, and Harbor itself.

The goal is to build a secure and persistent private container registry from an existing Kubernetes environment.

## 2. Objectives

- Prepare and configure an NFS server for persistent storage.
- Install and configure the necessary NFS client components.
- Configure persistent storage in Kubernetes.
- Install and configure Helm.
- Install and configure an Ingress Controller.
- Configure HTTPS/TLS for secure access.
- Deploy Harbor using the official Helm chart.
- Validate registry authentication and image push/pull operations.
- Document troubleshooting and security best practices.

## 3. Architecture

```text
             Docker / Kubernetes Clients
                        |
                        | HTTPS : 443
                        v
                Ingress Controller
                        |
                        v
                  Harbor Services
                        |
          +-------------+-------------+
          |             |             |
          v             v             v
      Harbor Core    Registry       Portal
          |             |
          v             v
       Database     Persistent Volume
                        |
                        v
                   NFS Server
```

The architecture illustrates the main components and their logical relationships. Harbor also includes supporting services such as Jobservice and Redis.

## 4. Prerequisites

The only prerequisite for this guide is:

- **An already installed and accessible Kubernetes cluster.**

The cluster must be operational before starting the deployment.

Verify cluster connectivity:

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

All additional components required for the deployment will be installed or configured in the following steps.

## 5. Repository Structure

```text
harbor-kubernetes-nfs-tls/
├── README.md
├── docs/
│   ├── 01-nfs-server-setup.md
│   ├── 02-kubernetes-storage.md
│   ├── 03-helm-installation.md
│   ├── 04-ingress-configuration.md
│   ├── 05-tls-configuration.md
│   ├── 06-harbor-deployment.md
│   ├── 07-validation.md
│   └── 08-troubleshooting.md
├── helm/
│   └── values.yaml
└── diagrams/
    └── architecture.png
```

## 6. Deployment Guide

The deployment is divided into the following stages.

### Step 1 — NFS Server Setup

Install the NFS server, create the Harbor storage directory, configure authorized clients, and export the shared directory.

**Documentation:** [NFS Server Setup](docs/01-nfs-server-setup.md)

### Step 2 — Kubernetes Persistent Storage

Configure Kubernetes to use the NFS share through PersistentVolumes, PersistentVolumeClaims, and, if required, a StorageClass and NFS provisioner.

**Documentation:** [Kubernetes Storage](docs/02-kubernetes-storage.md)

### Step 3 — Helm Installation

Install Helm, configure the official Harbor chart repository, and select a compatible chart version.

**Documentation:** [Helm Installation](docs/03-helm-installation.md)

### Step 4 — Ingress Configuration

Install and configure an Ingress Controller if one is not already available. Configure external access to Harbor through the cluster's networking infrastructure.

**Documentation:** [Ingress Configuration](docs/04-ingress-configuration.md)

### Step 5 — TLS Configuration

Prepare a certificate for the Harbor hostname and configure the required Kubernetes TLS Secret and HTTPS settings.

**Documentation:** [TLS Configuration](docs/05-tls-configuration.md)

### Step 6 — Harbor Deployment

Configure the Harbor Helm chart, connect its persistent storage, configure its external URL, and deploy Harbor into a dedicated namespace.

**Documentation:** [Harbor Deployment](docs/06-harbor-deployment.md)

### Step 7 — Validation

Verify the status of Harbor, its pods, services, Ingress resources, and persistent storage. Test HTTPS access, authentication, image push, and image pull.

**Documentation:** [Deployment Validation](docs/07-validation.md)

### Step 8 — Troubleshooting

Identify and resolve common issues related to NFS permissions, persistent volumes, pod startup, Ingress routing, TLS certificates, and registry authentication.

**Documentation:** [Troubleshooting](docs/08-troubleshooting.md)

## 7. Expected Result

At the end of the deployment, the environment should provide:

- A functional Harbor registry running on Kubernetes.
- Persistent storage backed by NFS.
- Secure HTTPS access through an Ingress Controller.
- A Helm-managed Harbor installation.
- Successful image push and pull operations.
- A documented procedure for troubleshooting and maintenance.

These are the intended outcomes; they must be verified during the validation stage.

## 8. Security Best Practices

- Restrict access to the NFS share to authorized Kubernetes nodes.
- Apply appropriate ownership and permissions to persistent storage.
- Use valid TLS certificates and verify certificate trust.
- Never commit private keys, passwords, or access tokens to GitHub.
- Protect Kubernetes Secrets and restrict access through RBAC.
- Pin the Harbor Helm chart version for reproducible deployments.
- Back up Harbor's persistent data and database.
- Document and test recovery procedures.

## 9. Official Documentation

- [Harbor Documentation](https://goharbor.io/docs/)
- [Harbor Helm Chart](https://github.com/goharbor/harbor-helm)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Kubernetes Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Helm Documentation](https://helm.sh/docs/)

## 10. Author

**Kubernetes | Linux | DevOps | Infrastructure Security**

This project documents the complete deployment of a private Harbor registry on an existing Kubernetes cluster, from infrastructure preparation to secure deployment and operational validation.
