# Step 3 — Traefik and MetalLB Installation on Kubernetes

## Overview

This guide describes how to install and configure **Traefik** and **MetalLB** in a local Kubernetes cluster using Helm.

Traefik acts as the Kubernetes Ingress Controller and routes incoming HTTP/HTTPS requests to the appropriate applications. MetalLB provides external IP addresses for Kubernetes Services of type `LoadBalancer`, making it possible to expose applications in a local or bare-metal environment without a cloud load balancer.

This configuration prepares the cluster for the deployment of Harbor, including its future DNS, TLS, and Ingress configuration.

## 1. Architecture Overview

### 1.1. Network configuration

The following table summarizes the network configuration used in this environment.

| Component | IP address |
|---|---|
| Kubernetes master | `192.168.1.10` |
| Kubernetes worker | `192.168.1.11` |
| NFS server | `192.168.1.20` |
| Windows client | `192.168.1.224` |
| Network gateway | `192.168.1.1` |
| Local network | `192.168.1.0/24` |

### 1.2. Request flow

The expected request flow is as follows:

1. A client sends an HTTP or HTTPS request to the IP address assigned to the Traefik Service.
2. MetalLB announces the assigned IP address on the local network using Layer 2 mode.
3. Traefik receives the incoming request.
4. Traefik evaluates the configured Ingress rules and matches the requested hostname.
5. The request is forwarded to the corresponding Kubernetes Service and application Pods.

**Important:** MetalLB handles external IP address allocation and advertisement. Traefik handles HTTP/HTTPS routing. TLS certificates, hostname resolution, and Harbor-specific Ingress rules will be configured in the subsequent deployment steps.

## 2. Prerequisites

Before proceeding, ensure that:

- The Kubernetes master and worker nodes are available.
- `kubectl` is configured to communicate with the cluster.
- Helm is installed and functional.
- The cluster has working network connectivity between its nodes.
- An IP address range has been identified for MetalLB and can be reserved without conflicting with existing devices or the DHCP pool.

### 2.1. Verify the Kubernetes nodes

Run the following command on the master node:

```bash
kubectl get nodes -o wide
```

![Screenshot: Kubernetes nodes and their status](../images/traefik-metallb/01-kubernetes-nodes.png)

Both nodes should report a `Ready` status.

If a node is not ready, investigate its status before proceeding with the installation.

### 2.2. Check the cluster Pods

List the Pods across all namespaces:

```bash
kubectl get pods -A
```

![Screenshot: Kubernetes Pods across all namespaces](../images/traefik-metallb/02-kubernetes-pods.png)

Review the output and check for Pods that are in unexpected states, such as `Pending`, `CrashLoopBackOff`, or `ImagePullBackOff`.

Resolve any relevant cluster issues before installing the networking components.

## 3. Install Traefik with Helm

Traefik will serve as the Ingress Controller for applications deployed in the cluster.

Its Kubernetes Service will use the `LoadBalancer` type so that MetalLB can allocate an external IP address once its address pool is configured.

### 3.1. Add the official Traefik Helm repository

Add the official Helm repository:

```bash
helm repo add traefik https://traefik.github.io/charts
```

### 3.2. Update the Helm repositories

Refresh the repository index:

```bash
helm repo update
```

This ensures that Helm has access to the latest chart metadata available from the configured repositories.

### 3.3. Search for the Traefik chart

Verify that the Traefik chart is available:

```bash
helm search repo traefik/traefik
```

![Screenshot: Creating the Traefik namespace](../images/traefik-metallb/03-create-traefik-helm.png)

Record the chart version displayed in the output. Pinning a specific version can help make future installations and upgrades more reproducible.

### 3.4. Inspect the Traefik chart configuration

Before installation, inspect the default values provided by the chart.

Download the default configuration:

```bash
helm show values traefik/traefik > traefik-values-default.yaml
```

![Screenshot: Available Traefik Helm chart](../images/traefik-metallb/07-traefik-chart-search.png)

Inspect the Service configuration:

```bash
grep -n -A25 '^service:' traefik-values-default.yaml
```

![Screenshot: Traefik Service configuration](../images/traefik-metallb/09-traefik-service-values.png)

Inspect the entrypoints and port configuration:

```bash
grep -n -A35 '^ports:' traefik-values-default.yaml
```

![Screenshot: Traefik entrypoint and port configuration](../images/traefik-metallb/10-traefik-port-values.png)

These checks help identify the chart options that control the Service type, entrypoints, and exposed ports.

**Note:** Helm chart values can change between versions. Always check the configuration supported by the version you intend to install rather than assuming that a particular option or default value is available.

### 3.5. Create the Traefik namespace

Create a dedicated namespace for Traefik:

```bash
kubectl create namespace traefik
```

If the namespace already exists, verify it instead of attempting to create it again:

```bash
kubectl get namespace traefik
```

Using a dedicated namespace keeps the Traefik resources separate from applications deployed in other namespaces.

### 3.6. Install Traefik

Install Traefik with a `LoadBalancer` Service:

```bash
helm upgrade --install traefik traefik/traefik \
  --namespace traefik \
  --set service.type=LoadBalancer \
  --wait \
  --timeout 5m
```

![Screenshot: Installing Traefik with Helm](../images/traefik-metallb/13-install-traefik.png)

The `helm upgrade --install` command installs the release if it does not exist or upgrades it if it already exists.

The `--namespace traefik` option specifies where the release will be installed. The `--wait` option makes Helm wait for the relevant resources to become ready, subject to the specified timeout.

At this stage, the Service's external IP address may remain `Pending`. This is expected if no load balancer implementation is available to allocate an address yet. MetalLB will provide that functionality once it is installed and configured.

### 3.7. Verify the Traefik Pods

Check the status of the Traefik Pods:

```bash
kubectl get pods -n traefik -o wide
```

![Screenshot: Traefik Pod status](../images/traefik-metallb/14-traefik-pods.png)

Wait until the relevant Pods show a `Running` status and their readiness checks pass.

If a Pod fails to start, inspect its logs and events before continuing.

### 3.8. Verify the Traefik Service

List the Services in the Traefik namespace:

```bash
kubectl get svc -n traefik
```

![Screenshot: Traefik Service and external IP status](../images/traefik-metallb/15-traefik-service.png)

Verify that the Traefik Service has the expected type:

```text
LoadBalancer
```

Before MetalLB is configured, the `EXTERNAL-IP` field may show `Pending`.

Depending on the chart version and configuration, the Service may also expose NodePorts. For example, `30080` and `30443` may be used for HTTP and HTTPS, respectively, but these values must be confirmed from the actual command output.

Do not assume that the NodePorts are fixed across installations.

### 3.9. Check the Kubernetes IngressClass

List the available IngressClasses:

```bash
kubectl get ingressclass
```

![Screenshot: Available Kubernetes IngressClasses](../images/traefik-metallb/16-ingressclass-list.png)

If IngressClass creation is enabled in the selected Traefik chart configuration, an IngressClass named `traefik` should be available.

Verify it directly:

```bash
kubectl get ingressclass traefik
```

The IngressClass identifies the controller responsible for processing matching Kubernetes Ingress resources.

### 3.10. Test HTTP connectivity through a NodePort

If the Traefik Service exposes HTTP on NodePort `30080`, test connectivity through the master node:

```bash
curl -I http://192.168.1.10:30080
```

![Screenshot: HTTP connectivity test through the master NodePort](../images/traefik-metallb/18-traefik-http-test.png)

A `404 Not Found` response can be expected when Traefik receives a request but no configured route matches it. This can indicate that the request reached Traefik, but it does not by itself confirm that an application route is working.

This test applies only if NodePort `30080` is actually exposed by the Service. If the Service uses a different NodePort, replace `30080` with the value shown by:

```bash
kubectl get svc -n traefik
```

## 4. Install MetalLB

MetalLB enables Kubernetes Services of type `LoadBalancer` to receive external IP addresses in environments where a cloud-provider load balancer is not available.

This guide uses **Layer 2 mode**, in which MetalLB announces IP addresses on the local network using ARP for IPv4.

### 4.1. Add the official MetalLB Helm repository

Add the MetalLB Helm repository:

```bash
helm repo add metallb https://metallb.github.io/metallb
```

### 4.2. Update the Helm repositories

Refresh the repository index:

```bash
helm repo update
```

### 4.3. Create the MetalLB namespace

Create the namespace dedicated to MetalLB:

```bash
kubectl create namespace metallb-system
```

![Screenshot: Updating Helm repositories for MetalLB](../images/traefik-metallb/20-metallb-repo-update.png)

If the namespace already exists, verify it instead:

```bash
kubectl get namespace metallb-system
```

### 4.4. Configure namespace security labels

MetalLB's networking components may require permissions that are restricted by Kubernetes Pod Security Admission.

In clusters where the applicable Pod Security Admission policy blocks the required workloads, the `metallb-system` namespace may need the `privileged` enforcement level.

Apply the labels:

```bash
kubectl label namespace metallb-system \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/audit=privileged \
  pod-security.kubernetes.io/warn=privileged \
  --overwrite
```

![Screenshot: Applying Pod Security Admission labels to MetalLB](../images/traefik-metallb/22-metallb-security-labels.png)

These labels affect the Pod Security Admission policy applied to the namespace. Use them only when appropriate for the cluster's security requirements, and keep the privileged scope limited to the components that require it.

### 4.5. Install MetalLB

Install MetalLB using Helm:

```bash
helm upgrade --install metallb metallb/metallb \
  --namespace metallb-system
```

This command installs the release if it does not exist or upgrades it if it already exists.

The exact resources created depend on the chart version and configuration. Allow the installation to complete, then verify the status of the deployed components.

### 4.6. Verify the MetalLB Pods

Check the Pods in the MetalLB namespace:

```bash
kubectl get pods -n metallb-system -o wide
```

![Screenshot: MetalLB Pods and their status](../images/traefik-metallb/24-metallb-pods.png)

Wait for the required MetalLB components to become ready.

If a Pod is not ready, inspect its events and logs to identify the cause before configuring the address pool.

### 4.7. Verify the MetalLB Custom Resource Definitions

Check that the MetalLB Custom Resource Definitions (CRDs) are installed:

```bash
kubectl get crd | grep metallb
```

The required CRDs must be available before you create the IP address pool and Layer 2 advertisement resources.

If the expected CRDs are missing, check the Helm release status and installation logs.

## 5. Configure the MetalLB IP Address Pool

MetalLB can allocate external IP addresses only from a configured address pool.

The example environment uses the local network `192.168.1.0/24`. The candidate address range for MetalLB is:

```text
192.168.1.240-192.168.1.250
```

**Important:** This range is an example based on the network configuration above. Before using it, verify that the addresses are available, belong to the correct subnet, and do not overlap with the router's DHCP pool or addresses assigned to other devices. Reserve the selected range according to your network's addressing policy.

### 5.1. Create the MetalLB configuration file

Create the configuration file:

```bash
nano metallb-config.yaml
```

Add the following YAML configuration:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: local-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.1.240-192.168.1.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: local-l2
  namespace: metallb-system
spec:
  ipAddressPools:
    - local-pool
```

Save the file and exit the editor.

The configuration contains two resources:

- **`IPAddressPool`** defines the range of external IP addresses that MetalLB is allowed to allocate.
- **`L2Advertisement`** enables MetalLB to announce addresses from the specified pool on the local network using Layer 2.

The second resource references the pool through the `local-pool` name.

### 5.2. Apply the configuration

Apply the YAML file to the cluster:

```bash
kubectl apply -f metallb-config.yaml
```

Check the command output for errors. If the resources cannot be created, verify that the MetalLB CRDs are installed and that the YAML configuration is valid.

### 5.3. Verify the IP address pool

Confirm that the pool exists:

```bash
kubectl get ipaddresspools -n metallb-system
```

The output should include the `local-pool` resource.

### 5.4. Verify the Layer 2 advertisement

Check the Layer 2 advertisement resource:

```bash
kubectl get l2advertisements -n metallb-system
```

![Screenshot: MetalLB IP address pool](../images/traefik-metallb/28-ipaddresspool.png)

The output should include the `local-l2` resource.

If the cluster nodes have multiple network interfaces, verify which interface MetalLB should use to announce the addresses. When necessary, the `interfaces` field can be configured in the `L2Advertisement` resource using the actual interface names on the nodes.

## 6. Verify External IP Assignment to Traefik

Once MetalLB is installed and the IP address pool and Layer 2 advertisement are configured, verify that an external IP address is assigned to the Traefik Service.

### 6.1. Watch the Traefik Service

Run the following command to monitor the Service:

```bash
kubectl get svc traefik -n traefik -w
```

Wait until the `EXTERNAL-IP` field displays an address from the configured pool. Press `Ctrl+C` after the address has been assigned.

Example output (illustrative only):

```text
NAME      TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)
traefik   LoadBalancer   10.x.x.x       192.168.1.240   ...
```

The displayed IP address, cluster IP, and ports depend on the actual cluster configuration. The example address is not a guarantee that MetalLB will allocate that specific IP.

### 6.2. Verify the Traefik Service

Check the Service again:

```bash
kubectl get svc traefik -n traefik
```

![Screenshot: Traefik Service with its assigned external IP](../images/traefik-metallb/31-traefik-external-ip.png)

Confirm that:

- The Service type is `LoadBalancer`.
- The `EXTERNAL-IP` field contains an address from the configured MetalLB pool.
- The expected ports are listed.

### 6.3. Inspect the Service details

Get detailed information about the Traefik Service:

```bash
kubectl describe svc traefik -n traefik
```

Review the Service type, ports, endpoints, and events.

If the external IP remains `Pending`, verify the following:

1. The MetalLB Pods are running and ready.
2. The MetalLB CRDs are installed.
3. The `IPAddressPool` and `L2Advertisement` resources exist.
4. The configured address range is valid and does not conflict with other devices.
5. The Traefik Service is configured with the `LoadBalancer` type.
6. The local network allows the required Layer 2 announcements.

## 7. Next Steps: Harbor DNS, TLS, and Ingress

After Traefik and MetalLB are configured, the next stage is to prepare Harbor for access through its local hostname.

The remaining tasks include:

1. Configure local DNS resolution for `harbor.k8s.local`.
2. Prepare and configure the TLS certificate.
3. Deploy Harbor using the previously configured `nfs-harbor` StorageClass.
4. Configure the appropriate Ingress resources and hostname.
5. Verify HTTPS access to Harbor from the client machine.

These steps will connect the cluster's storage, networking, and application-access configuration into a complete Harbor deployment.

## 8. Official References

- [Traefik Kubernetes Documentation](https://doc.traefik.io/traefik/setup/kubernetes/)
- [Official Traefik Helm Chart](https://github.com/traefik/traefik-helm-chart)
- [MetalLB Installation Documentation](https://metallb.io/installation/)
- [MetalLB Configuration Documentation](https://metallb.io/configuration/)
- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
