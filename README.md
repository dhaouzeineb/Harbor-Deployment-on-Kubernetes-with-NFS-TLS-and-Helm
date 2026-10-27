# Harbor Deployment on Kubernetes with NFS, TLS and Helm

## Project Overview

This project documents the deployment of Harbor, a private container image registry, on a Kubernetes cluster. It uses NFS for persistent storage, Helm for installation, Traefik as the Ingress Controller, MetalLB for external IP allocation, BIND9 for local DNS resolution, and TLS certificates to secure access through HTTPS.

## Architecture

The deployment integrates the following components:

- **Kubernetes:** Container orchestration.
- **Harbor:** Private container image registry.
- **NFS:** Persistent storage for Harbor data.
- **Helm:** Application installation and management.
- **Traefik:** Ingress traffic routing.
- **MetalLB:** External IP address allocation.
- **BIND9:** Local DNS resolution for `harbor.k8s.local`.
- **TLS:** Secure HTTPS communication.

## Documentation

1. [NFS Server Setup](docs/01-nfs-server-setup.md)
2. [Kubernetes Storage Configuration](docs/02-kubernetes-storage.md)
3. [Traefik and MetalLB Configuration](docs/03-traefik-metallb.md)
4. [DNS Configuration](docs/04-dns-configuration.md)
5. [Harbor Installation with TLS](docs/05-Install-Harbor-with-tls.md)

## Expected Outcome

The objective is to provide a private Harbor registry accessible through HTTPS, with persistent storage backed by NFS and network access managed through Kubernetes, Traefik, and MetalLB.

## Official Documentation

- [Harbor](https://goharbor.io/docs/)
- [Kubernetes](https://kubernetes.io/docs/)
- [Helm](https://helm.sh/docs/)
- [Traefik](https://doc.traefik.io/traefik/)
- [MetalLB](https://metallb.io/)
- [NFS Subdirectory External Provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)
