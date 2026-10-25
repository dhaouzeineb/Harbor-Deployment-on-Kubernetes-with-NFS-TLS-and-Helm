# 02 — Kubernetes Storage with NFS and Helm

This guide explains how to deploy the NFS Subdirectory External Provisioner on a Kubernetes cluster using Helm. It also covers StorageClass creation, dynamic PersistentVolume provisioning, and storage validation before deploying Harbor.

## 1. Architecture and Prerequisites

The NFS server provides shared storage for the Kubernetes cluster. The NFS Subdirectory External Provisioner automatically creates subdirectories in the exported NFS directory when PersistentVolumeClaims request storage through the configured StorageClass.

| Component | Configuration |
|---|---|
| Kubernetes master | `192.168.1.10` |
| Kubernetes worker | `192.168.1.11` |
| NFS server | `192.168.1.20` |
| NFS export path | `/srv/nfs/harbor` |
| Provisioner namespace | `storage-system` |
| StorageClass | `nfs-harbor` |
| Access mode | `ReadWriteMany` |

Before proceeding, ensure that:

- The NFS server is configured and accessible from both Kubernetes nodes.
- Write tests from the master and worker have succeeded.
- `kubectl` is configured to access the cluster.
- Helm is installed on the master node.

## 2. Add the Helm Repository

Add the official NFS Subdirectory External Provisioner Helm repository:

```bash
helm repo add nfs-subdir-external-provisioner \
  https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/

helm repo update
```

Verify that the chart is available:

```bash
helm search repo nfs-subdir-external-provisioner
```

List the available chart versions:

```bash
helm search repo nfs-subdir-external-provisioner --versions
```

Select a version from the returned list. Do not assume that a particular version is available without checking the repository.

## 3. Inspect the Chart Configuration

Download the default values for the selected chart version:

```bash
helm show values \
  nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --version VERSION \
  > nfs-values-default.yaml
```

Replace `VERSION` with the version selected in the previous step.

Inspect the NFS configuration:

```bash
grep -n -A25 '^nfs:' nfs-values-default.yaml
```

Inspect the StorageClass configuration:

```bash
grep -n -A35 '^storageClass:' nfs-values-default.yaml
```

These commands help confirm the configuration keys supported by the selected chart version.

## 4. Create the Namespace

Create a dedicated namespace for the provisioner:

```bash
kubectl create namespace storage-system
```

If the namespace already exists, verify it instead:

```bash
kubectl get namespace storage-system
```

## 5. Install the NFS Provisioner

The NFS Subdirectory External Provisioner dynamically provisions PersistentVolumes by creating subdirectories within the configured NFS export.

Set the chart version selected and verified in Section 2:

```bash
CHART_VERSION="VERSION"
```

Replace `VERSION` with the actual chart version.

Install the provisioner:

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
  --set storageClass.accessModes[0]=ReadWriteMany
```

**Important:** Confirm that every configuration key matches the selected chart version. In particular, check the default values for `nfs`, `storageClass`, and `accessModes` before installation.

The installation creates the provisioner resources and the `nfs-harbor` StorageClass, provided the chart configuration is valid.

The `Retain` reclaim policy helps prevent automatic reclamation of the underlying PersistentVolume when its claim is deleted. It does not replace regular backups.

## 6. Verify the Installation

Check the Helm release:

```bash
helm list -n storage-system
```

Check the provisioner pods:

```bash
kubectl get pods -n storage-system -o wide
```

Check the Deployment:

```bash
kubectl get deployments -n storage-system
```

Check the StorageClass:

```bash
kubectl get storageclass
```

Inspect the StorageClass details:

```bash
kubectl describe storageclass nfs-harbor
```

Inspect recent events:

```bash
kubectl get events -n storage-system \
  --sort-by=.lastTimestamp
```

The provisioner pod should reach the `Running` state and become ready. The `nfs-harbor` StorageClass should also appear in the cluster.

If the provisioner pod fails, retrieve its logs:

```bash
kubectl logs -n storage-system deployment/DEPLOYMENT_NAME
```

Replace `DEPLOYMENT_NAME` with the actual Deployment name returned by:

```bash
kubectl get deployments -n storage-system
```

Do not proceed to the PVC test until the provisioner is operational.

## 7. Test Dynamic PersistentVolume Provisioning

A running provisioner does not necessarily mean that it can successfully create and mount volumes. The following test validates the complete provisioning workflow.

### 7.1. Create the Test Manifest

On the Kubernetes master, create a file named `nfs-test.yaml`:

```bash
nano nfs-test.yaml
```

Add the following manifest:

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

This manifest creates:

- A PersistentVolumeClaim requesting `1Gi` of storage.
- A Pod that mounts the claim at `/data`.
- A test file written to the mounted volume.

### 7.2. Apply the Manifest

```bash
kubectl apply -f nfs-test.yaml
```

Check the PersistentVolumeClaim:

```bash
kubectl get pvc nfs-test-pvc
```

Check the PersistentVolumes:

```bash
kubectl get pv
```

The PVC should eventually reach the `Bound` state.

Check the Pod:

```bash
kubectl get pod nfs-test-pod -o wide
```

The Pod should reach the `Running` state.

### 7.3. Verify the Written Data

Check the Pod logs:

```bash
kubectl logs nfs-test-pod
```

Read the test file directly:

```bash
kubectl exec nfs-test-pod -- cat /data/test.txt
```

Expected output:

```text
NFS fonctionne
```

You can also verify that the file exists on the NFS server:

```bash
sudo find /srv/nfs/harbor -type f -name test.txt
```

The exact directory depends on the subdirectory created by the provisioner.

**Storage capacity note:** The `1Gi` request in this PVC does not, by itself, enforce a strict physical quota on the NFS share. Monitor the actual available disk space on the NFS server.

### 7.4. Troubleshooting the PVC Test

If the PVC remains `Pending`, inspect the claim and its events:

```bash
kubectl describe pvc nfs-test-pvc
kubectl get events --sort-by=.lastTimestamp
```

If the Pod fails to start, inspect its status and events:

```bash
kubectl describe pod nfs-test-pod
kubectl logs nfs-test-pod
```

Check the provisioner logs and confirm that the NFS server address, export path, network connectivity, and directory permissions are correct.

## 8. Clean Up the Test Resources

After verifying that the test succeeded, delete the test Pod and PVC:

```bash
kubectl delete pod nfs-test-pod
kubectl delete pvc nfs-test-pvc
```

Because the StorageClass uses the `Retain` reclaim policy, the PersistentVolume may remain in the `Released` state after the PVC is deleted. This is expected.

Do not manually delete the PersistentVolume or remove its NFS directory until you have confirmed that it contains only disposable test data.

## 9. Prepare Storage for Harbor

Once the PVC test succeeds, the NFS provisioner is ready for the next stage of the deployment.

Harbor contains multiple components with different storage requirements. The registry stores container images, while other components may store job logs, database data, Redis data, and vulnerability-scanning data, depending on the chosen deployment configuration.

Before installing Harbor, inspect the values file for the exact Harbor Helm chart version and configure the persistence settings required by the selected architecture.

Pay particular attention to:

- Registry persistence and its StorageClass.
- Jobservice persistence and its StorageClass.
- Database and Redis persistence settings.
- The required access modes for each PersistentVolumeClaim.
- Backup, retention, and recovery requirements.

Do not assume that every Harbor component can use the same storage configuration. Verify the chart values and the requirements of the selected Harbor deployment mode.


## References

- [NFS Subdirectory External Provisioner — Official GitHub Repository](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)
- [NFS Subdirectory External Provisioner — Helm Chart Repository](https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/)
- [Kubernetes PersistentVolumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Kubernetes StorageClasses](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [Harbor Documentation](https://goharbor.io/docs/)
