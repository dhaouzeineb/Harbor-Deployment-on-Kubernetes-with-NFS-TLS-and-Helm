# Step 2 — Kubernetes Storage with NFS and Helm

This guide explains how to deploy and configure the NFS Subdirectory External Provisioner on a Kubernetes cluster using Helm. It covers dynamic PersistentVolume provisioning, StorageClass configuration, PVC validation, troubleshooting, and storage preparation for Harbor.

The objective is to provide a reliable, reusable storage configuration that can be validated before proceeding with the Harbor deployment.

## 1. Architecture and Prerequisites

### 1.1. Architecture Overview

The NFS server provides shared storage for the Kubernetes cluster. The NFS Subdirectory External Provisioner integrates the NFS export with Kubernetes and automatically creates subdirectories when PersistentVolumeClaims request storage through the configured StorageClass.

The provisioning workflow is as follows:

1. A PersistentVolumeClaim requests storage using the `nfs-harbor` StorageClass.
2. The provisioner creates a dedicated subdirectory inside the NFS export.
3. Kubernetes dynamically creates a PersistentVolume associated with the PVC.
4. A Pod mounts the volume and accesses the shared storage.

### 1.2. Infrastructure Configuration

| Component | Configuration |
|---|---|
| Kubernetes master | `192.168.1.10` |
| Kubernetes worker | `192.168.1.11` |
| NFS server | `192.168.1.20` |
| NFS export path | `/srv/nfs/harbor` |
| Provisioner namespace | `storage-system` |
| StorageClass | `nfs-harbor` |
| Access mode | `ReadWriteMany` |
| Reclaim policy | `Retain` |

### 1.3. Prerequisites

Before proceeding, ensure that:

- The NFS server is configured and accessible from both Kubernetes nodes.
- NFS connectivity and write tests have succeeded from the relevant nodes.
- The NFS export `/srv/nfs/harbor` exists and has appropriate permissions.
- `kubectl` is configured to access the Kubernetes cluster.
- Helm is installed on the master node.
- The Kubernetes nodes can access the container images required by the provisioner and test Pod.

The NFS server must be reachable from every Kubernetes node that may host a workload using the provisioned volumes.

## 2. Add the Helm Repository

### 2.1. Add the official repository

Run the following command on the Kubernetes master:

```bash
helm repo add nfs-subdir-external-provisioner \
  https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
```

### 2.2. Update the repository index

```bash
helm repo update
```

### 2.3. Verify chart availability

```bash
helm search repo nfs-subdir-external-provisioner
```

### 2.4. List available chart versions

```bash
helm search repo nfs-subdir-external-provisioner --versions
```

Choose a chart version from the available results and record its exact version. This version will be used in the following steps to ensure a reproducible installation.

## 3. Inspect the Chart Configuration

Before installing the provisioner, inspect the default values of the selected Helm chart. This helps identify the supported configuration options and avoid using parameters that are incompatible with the selected version.

### 3.1. Download the default values

Replace `VERSION` with the chart version selected in Section 2.

```bash
helm show values \
  nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --version VERSION \
  > nfs-values-default.yaml
```

![Screenshot: Downloaded Helm chart values](../images/kubernetes-storage/05-helm-show-values.png)

### 3.2. Inspect the NFS configuration

```bash
grep -n -A25 '^nfs:' nfs-values-default.yaml
```

![Screenshot: NFS configuration options](../images/kubernetes-storage/06-inspect-nfs-values.png)

### 3.3. Inspect the StorageClass configuration

```bash
grep -n -A35 '^storageClass:' nfs-values-default.yaml
```

![Screenshot: StorageClass configuration options](../images/kubernetes-storage/07-inspect-storageclass-values.png)

These commands display the relevant configuration sections, including the NFS server settings and StorageClass parameters.

Verify that the selected chart version supports the options required for the deployment before proceeding.

## 4. Create the Namespace

### 4.1. Create the provisioner namespace

```bash
kubectl create namespace storage-system
```

If the namespace already exists, continue with the verification step.

### 4.2. Verify the namespace

```bash
kubectl get namespace storage-system
```

The namespace should have the `Active` status.

## 5. Install the NFS Provisioner

The NFS Subdirectory External Provisioner dynamically provisions PersistentVolumes by creating subdirectories inside the configured NFS export.

### 5.1. Define the chart version

Replace `VERSION` with the exact chart version selected in Section 2.

```bash
CHART_VERSION="VERSION"
```

Use the same version throughout the installation procedure.

### 5.2. Install the provisioner

The following command configures the NFS server, export path, StorageClass name, reclaim policy, and access mode.

```bash
helm install nfs-harbor \
  nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --namespace storage-system \
  --version "$CHART_VERSION" \
  --set nfs.server=192.168.1.20 \
  --set nfs.path=/srv/nfs/harbor \
  --set storageClass.name=nfs-harbor \
  --set storageClass.defaultClass=false \
  --set storageClass.reclaimPolicy=Retain \
  --set 'storageClass.accessModes[0]=ReadWriteMany'
```

![Screenshot: NFS provisioner Helm installation](../images/kubernetes-storage/11-install-provisioner.png)

The configuration defines the following parameters:

- **NFS server:** `192.168.1.20`
- **NFS export path:** `/srv/nfs/harbor`
- **StorageClass:** `nfs-harbor`
- **Default StorageClass:** Disabled
- **Reclaim policy:** `Retain`
- **Access mode:** `ReadWriteMany`

The StorageClass is not configured as the cluster default. Applications must explicitly specify `storageClassName: nfs-harbor` to use it.

**Important:** Confirm that the selected chart version supports all the specified options. If the Helm release already exists, inspect it with `helm list -n storage-system` before deciding whether to install or upgrade it.

The `Retain` reclaim policy helps prevent automatic reclamation of the underlying PersistentVolume when its claim is deleted. The associated data may remain on the NFS server and must be managed deliberately. This policy does not replace backups.

## 6. Verify the Installation

### 6.1. Check the Helm release

```bash
helm list -n storage-system
```

Confirm that the `nfs-harbor` release appears in the output.

### 6.2. Check the provisioner Pods

```bash
kubectl get pods -n storage-system -o wide
```

The provisioner Pod should eventually reach the `Running` state, with its containers ready.

### 6.3. Check the Deployment

```bash
kubectl get deployments -n storage-system
```

Verify that the Deployment has the expected number of available replicas.

### 6.4. Check the StorageClass

```bash
kubectl get storageclass
```

The `nfs-harbor` StorageClass should appear in the output.

### 6.5. Inspect the StorageClass details

```bash
kubectl describe storageclass nfs-harbor
```

Verify the provisioner name, reclaim policy, and other relevant configuration settings.

### 6.6. Inspect recent events

```bash
kubectl get events -n storage-system \
  --sort-by=.lastTimestamp
```

![Screenshot: Recent storage-system events](../images/kubernetes-storage/17-check-events.png)

Events can help identify image-pulling problems, scheduling errors, configuration issues, or other deployment failures.

### 6.7. Identify the Deployment name

If the provisioner Pod fails, retrieve the actual Deployment name:

```bash
kubectl get deployments -n storage-system
```

### 6.8. Retrieve the provisioner logs

Replace `DEPLOYMENT_NAME` with the actual Deployment name returned by the previous command.

```bash
kubectl logs -n storage-system deployment/DEPLOYMENT_NAME
```

Inspect the logs for errors related to NFS connectivity, directory permissions, or provisioner configuration.

Do not continue to the PVC test until the provisioner is operational and the StorageClass exists.

## 7. Test Dynamic PersistentVolume Provisioning

A running provisioner does not necessarily mean that it can successfully create and mount volumes. This test validates the complete provisioning workflow, including PVC creation, volume mounting, and data access.

### 7.1. Create the test manifest

On the Kubernetes master, create the file:

```bash
nano nfs-test.yaml
```

Add the following YAML configuration:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-test-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-harbor
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: nfs-test-pod
  namespace: default
spec:
  containers:
    - name: test
      image: busybox:1.36
      command:
        - sh
        - -c
        - "echo 'NFS fonctionne' > /data/test.txt && sleep 3600"
      volumeMounts:
        - name: nfs-volume
          mountPath: /data
  volumes:
    - name: nfs-volume
      persistentVolumeClaim:
        claimName: nfs-test-pvc
```

This manifest creates two resources:

1. A PersistentVolumeClaim requesting `1Gi` from the `nfs-harbor` StorageClass.
2. A Pod that mounts the claim at `/data` and writes a test file.

Save the file and exit the editor.

### 7.2. Apply the manifest

```bash
kubectl apply -f nfs-test.yaml
```

### 7.3. Check the PersistentVolumeClaim

```bash
kubectl get pvc nfs-test-pvc
```

The PVC should eventually reach the `Bound` state.

If the PVC remains `Pending`, inspect its events and the provisioner logs before proceeding.

### 7.4. Check the PersistentVolumes

```bash
kubectl get pv
```

Verify that the volume associated with `nfs-test-pvc` exists and is bound to the claim.

### 7.5. Check the test Pod

```bash
kubectl get pod nfs-test-pod -o wide
```

![Screenshot: NFS test Pod status](../images/kubernetes-storage/24-check-test-pod.png)

The Pod should reach the `Running` state.

If it does not start, inspect the Pod events:

```bash
kubectl describe pod nfs-test-pod
```

### 7.6. Check the Pod logs

```bash
kubectl logs nfs-test-pod
```

The command may return no output because the test script writes its message to `/data/test.txt` rather than standard output.

### 7.7. Read the test file

```bash
kubectl exec nfs-test-pod -- cat /data/test.txt
```

![Screenshot: Reading the test file from the mounted volume](../images/kubernetes-storage/26-verify-test-file.png)

Expected output:

```text
NFS fonctionne
```

This confirms that the test Pod can write to and read from the mounted volume.

### 7.8. Verify the file on the NFS server

Run this command on the NFS server, not on the Kubernetes master unless both roles are hosted on the same machine.

```bash
sudo find /srv/nfs/harbor -type f -name test.txt
```

The exact directory depends on the subdirectory created by the provisioner.

**Storage capacity note:** The `1Gi` request does not enforce a strict physical quota on the NFS share. Monitor the actual free disk space on the NFS server.

## 8. Troubleshooting the PVC Test

### 8.1. Inspect a PVC that remains Pending

```bash
kubectl describe pvc nfs-test-pvc
```

Review the PVC events to identify provisioning or binding failures.

### 8.2. Inspect recent cluster events

```bash
kubectl get events --sort-by=.lastTimestamp
```

Look for errors related to persistent volumes, provisioning, scheduling, or volume mounting.

### 8.3. Inspect the test Pod

```bash
kubectl describe pod nfs-test-pod
```

Check the events for mount failures, permission errors, or image-pulling problems.

### 8.4. Retrieve the test Pod logs

```bash
kubectl logs nfs-test-pod
```

If the container is not running, use `kubectl describe pod nfs-test-pod` to identify the underlying issue.

If the PVC or Pod fails, verify:

- The NFS server address and export path.
- Network connectivity between the Kubernetes nodes and the NFS server.
- NFS directory ownership and permissions.
- Provisioner Pod status and logs.
- The StorageClass name and configuration.
- Availability of the required NFS client utilities on the Kubernetes nodes.

## 9. Clean Up the Test Resources

Only perform this cleanup after completing the test and confirming that the test data is no longer needed.

### 9.1. Delete the test Pod

```bash
kubectl delete pod nfs-test-pod
```

### 9.2. Delete the test PVC

```bash
kubectl delete pvc nfs-test-pvc
```

![Screenshot: Test PVC deletion](../images/kubernetes-storage/33-delete-test-pvc.png)

Because the StorageClass uses the `Retain` reclaim policy, the PersistentVolume may remain in the `Released` state after its claim is deleted. This is expected behavior.

Do not manually delete the PersistentVolume or remove its NFS directory until you have confirmed that it contains only disposable test data and that no application depends on it.

## 10. Prepare Storage for Harbor

Once the PVC test succeeds, the NFS provisioner is ready to provide storage for the next stage of the deployment.

Before proceeding with Harbor installation, verify that:

- The provisioner is operational.
- The `nfs-harbor` StorageClass exists.
- Dynamic PVC provisioning works correctly.
- The test Pod can mount and access the volume.
- The NFS export has sufficient free disk space.
- NFS permissions are appropriate for the intended workloads.

During Harbor deployment, configure the relevant persistent storage settings to reference the `nfs-harbor` StorageClass where appropriate.

The exact PVC requirements depend on the Harbor chart version and the persistence configuration selected for its components. Review the Harbor chart values before configuring the claims.

**Next step:** Configure Harbor persistence and install Harbor using Helm. TLS, Ingress, DNS, and registry access are addressed in separate documentation.

## References

- [NFS Subdirectory External Provisioner — Official GitHub Repository](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)
- [NFS Subdirectory External Provisioner — Helm Chart Repository](https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/)
- [Kubernetes PersistentVolumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Kubernetes StorageClasses](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [Harbor Documentation](https://goharbor.io/docs/)
