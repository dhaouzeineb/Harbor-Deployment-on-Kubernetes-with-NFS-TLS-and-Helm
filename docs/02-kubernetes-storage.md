# Step 2 — Kubernetes Storage with NFS and Helm

## Overview

This guide explains how to deploy and configure the **NFS Subdirectory External Provisioner** on a Kubernetes cluster using Helm.

The objective is to enable dynamic PersistentVolume (PV) provisioning through a Kubernetes StorageClass backed by an existing NFS server.

This procedure covers:

- NFS storage architecture and prerequisites.
- Helm repository configuration and chart version selection.
- Deployment of the NFS provisioner.
- StorageClass verification.
- PersistentVolumeClaim (PVC) creation and validation.
- Read/write and data persistence testing.
- Troubleshooting and resource cleanup.
- Storage preparation for the subsequent Harbor deployment.

> **Scope:** This guide covers NFS-based dynamic storage provisioning. It does not cover Harbor installation, TLS configuration, or Ingress configuration.

---

## 1. Architecture and Prerequisites

### 1.1. Architecture

The NFS server provides shared storage accessible from the Kubernetes nodes. The NFS Subdirectory External Provisioner integrates this storage with Kubernetes and automatically creates subdirectories within the exported directory when PersistentVolumeClaims request storage through the configured StorageClass.

The general workflow is:

1. A user creates a PersistentVolumeClaim.
2. Kubernetes identifies the requested StorageClass.
3. The NFS provisioner creates a subdirectory on the NFS server.
4. Kubernetes creates a PersistentVolume associated with the claim.
5. A Pod mounts the PersistentVolume and accesses the shared storage.

### 1.2. Infrastructure configuration

| Component | Configuration |
|---|---|
| Kubernetes master | `192.168.1.10` |
| Kubernetes worker | `192.168.1.11` |
| NFS server | `192.168.1.20` |
| NFS export path | `/srv/nfs/harbor` |
| Provisioner namespace | `storage-system` |
| Helm release | `nfs-harbor` |
| StorageClass | `nfs-harbor` |
| Access mode | `ReadWriteMany` |
| Reclaim policy | `Retain` |

**Important:** Replace the example IP addresses with the actual addresses of your infrastructure if they differ.

### 1.3. Prerequisites

Before proceeding, ensure that:

- The NFS server is installed and configured.
- The directory `/srv/nfs/harbor` is exported by the NFS server.
- The Kubernetes master and worker can reach the NFS server.
- NFS mount and write tests have succeeded from the relevant Kubernetes nodes.
- `kubectl` is configured to access the cluster.
- Helm is installed on the master node.
- The cluster can pull the container images required by the provisioner and test Pod.

The NFS export must be accessible from every node that may run a Pod using these volumes, not only from the node hosting the provisioner.

---

## 2. Add and Verify the Helm Repository

Run the following commands on the Kubernetes master.

### 2.1. Add the official Helm repository

```bash
helm repo add nfs-subdir-external-provisioner \
  https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
```

![Screenshot: Adding the NFS provisioner Helm repository](../images/kubernetes-storage/01-helm-repo-add.png)

### 2.2. Update the repository index

```bash
helm repo update
```

![Screenshot: Updating Helm repositories](../images/kubernetes-storage/02-helm-repo-update.png)

### 2.3. Verify chart availability

```bash
helm search repo nfs-subdir-external-provisioner
```

![Screenshot: Verifying chart availability](../images/kubernetes-storage/03-helm-search-repo.png)

### 2.4. List available chart versions

```bash
helm search repo nfs-subdir-external-provisioner --versions
```

![Screenshot: Listing available chart versions](../images/kubernetes-storage/04-helm-chart-versions.png)

Select a chart version that is compatible with your Kubernetes environment. Record the exact version returned by Helm.

For reproducibility, use the same version throughout this guide rather than relying on the latest chart version.

---

## 3. Inspect the Helm Chart Configuration

Inspect the chart defaults before installing the provisioner. This helps confirm which configuration options are supported by the selected chart version.

### 3.1. Define the chart version

Replace `VERSION` with the version selected in Section 2.

```bash
CHART_VERSION="VERSION"
```

For example, if Helm returns version `4.0.18` and that is the version you selected, use:

```bash
CHART_VERSION="4.0.18"
```

The example is illustrative: use the actual version available in your repository and verified in your environment.

### 3.2. Download the default values

```bash
helm show values \
  nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --version "$CHART_VERSION" \
  > nfs-values-default.yaml
```

![Screenshot: Downloading the default Helm chart values](../images/kubernetes-storage/05-helm-show-values.png)

### 3.3. Inspect the NFS configuration

```bash
grep -n -A25 '^nfs:' nfs-values-default.yaml
```

![Screenshot: Inspecting NFS configuration options](../images/kubernetes-storage/06-inspect-nfs-values.png)

### 3.4. Inspect the StorageClass configuration

```bash
grep -n -A35 '^storageClass:' nfs-values-default.yaml
```

![Screenshot: Inspecting StorageClass configuration options](../images/kubernetes-storage/07-inspect-storageclass-values.png)

### 3.5. Verify the chart templates

Render the chart locally to check the Kubernetes resources that would be generated with the intended configuration.

```bash
helm template nfs-harbor \
  nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --namespace storage-system \
  --version "$CHART_VERSION" \
  --set nfs.server=192.168.1.20 \
  --set nfs.path=/srv/nfs/harbor \
  --set storageClass.name=nfs-harbor \
  --set storageClass.defaultClass=false \
  --set storageClass.reclaimPolicy=Retain \
  --set 'storageClass.accessModes[0]=ReadWriteMany' \
  > nfs-rendered.yaml
```

![Screenshot: Rendering the NFS provisioner chart](../images/kubernetes-storage/08-helm-template.png)

Inspect the generated manifest:

```bash
less nfs-rendered.yaml
```

This step is optional but recommended. Confirm that the generated resources contain the intended NFS server, export path, StorageClass name, access mode, and reclaim policy.

---

## 4. Create the Provisioner Namespace

### 4.1. Create the namespace

```bash
kubectl create namespace storage-system
```

![Screenshot: Creating the storage-system namespace](../images/kubernetes-storage/09-create-namespace.png)

If the namespace already exists, Kubernetes will report that it already exists. In that case, proceed to the verification step.

### 4.2. Verify the namespace

```bash
kubectl get namespace storage-system
```

![Screenshot: Verifying the storage-system namespace](../images/kubernetes-storage/10-verify-namespace.png)

The namespace should have the `Active` status.

---

## 5. Install the NFS Subdirectory External Provisioner

### 5.1. Check for an existing Helm release

Before installing the provisioner, verify whether a release with the same name already exists.

```bash
helm list -n storage-system
```

![Screenshot: Checking existing Helm releases](../images/kubernetes-storage/11-check-helm-release.png)

If `nfs-harbor` already exists, inspect its configuration before deciding whether to upgrade it. Do not blindly run `helm install` again.

### 5.2. Install the provisioner

If the release does not exist, run:

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

![Screenshot: Installing the NFS provisioner with Helm](../images/kubernetes-storage/12-install-provisioner.png)

The command configures:

- **NFS server:** `192.168.1.20`
- **NFS export path:** `/srv/nfs/harbor`
- **StorageClass:** `nfs-harbor`
- **Default StorageClass:** Disabled
- **Reclaim policy:** `Retain`
- **Access mode:** `ReadWriteMany`

The StorageClass is not marked as the cluster default. Applications must explicitly request `storageClassName: nfs-harbor` to use it.

> **Reclaim policy:** `Retain` helps prevent automatic reclamation of the underlying PersistentVolume when its PVC is deleted. It does not provide backups or guarantee data integrity.

**Chart compatibility:** The command assumes that the selected chart version supports the specified values. If you select another version, verify the default values and rendered manifests before installation.

---

## 6. Verify the Provisioner Installation

### 6.1. Verify the Helm release

```bash
helm list -n storage-system
```

![Screenshot: Verifying the installed Helm release](../images/kubernetes-storage/13-verify-helm-release.png)

Confirm that the `nfs-harbor` release is present and deployed.

### 6.2. Check the provisioner Pods

```bash
kubectl get pods -n storage-system -o wide
```

![Screenshot: Checking provisioner Pod status](../images/kubernetes-storage/14-check-provisioner-pods.png)

The provisioner Pod should eventually reach the `Running` state and report all containers as ready.

### 6.3. Check the Deployment

```bash
kubectl get deployments -n storage-system
```

![Screenshot: Checking the provisioner Deployment](../images/kubernetes-storage/15-check-provisioner-deployment.png)

Verify that the Deployment has the expected number of available replicas.

### 6.4. Verify the StorageClass

```bash
kubectl get storageclass
```

![Screenshot: Listing Kubernetes StorageClasses](../images/kubernetes-storage/16-check-storageclasses.png)

The `nfs-harbor` StorageClass should appear in the output.

### 6.5. Inspect StorageClass details

```bash
kubectl describe storageclass nfs-harbor
```

![Screenshot: Inspecting the nfs-harbor StorageClass](../images/kubernetes-storage/17-describe-storageclass.png)

Verify the provisioner name, reclaim policy, volume binding mode, and access-related configuration where applicable.

### 6.6. Inspect recent events

```bash
kubectl get events -n storage-system \
  --sort-by=.lastTimestamp
```

![Screenshot: Inspecting storage-system events](../images/kubernetes-storage/18-check-storage-events.png)

Events can help identify image-pulling problems, scheduling errors, missing permissions, or other deployment issues.

### 6.7. Retrieve provisioner logs

First, retrieve the actual Deployment name:

```bash
kubectl get deployments -n storage-system
```

Then retrieve its logs:

```bash
kubectl logs -n storage-system deployment/DEPLOYMENT_NAME
```

Replace `DEPLOYMENT_NAME` with the name returned by the previous command.

![Screenshot: Inspecting provisioner logs](../images/kubernetes-storage/19-provisioner-logs.png)

Do not continue to the PVC test until the provisioner is operational and the StorageClass exists.

---

## 7. Test Dynamic PersistentVolume Provisioning

A running provisioner does not necessarily mean that it can successfully provision and mount volumes. This section validates the complete workflow, from PVC creation to actual data access.

### 7.1. Create the test manifest

On the Kubernetes master, create a file named `nfs-test.yaml`:

```bash
nano nfs-test.yaml
```

Add the following configuration:

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

![Screenshot: Creating the NFS test manifest](../images/kubernetes-storage/20-create-test-manifest.png)

Save the file and exit the editor.

The manifest creates two resources:

1. A PVC requesting `1Gi` from the `nfs-harbor` StorageClass.
2. A Pod that mounts the PVC at `/data` and writes a test file.

The PVC requests storage but does not create a strict quota on the NFS filesystem.

### 7.2. Apply the manifest

```bash
kubectl apply -f nfs-test.yaml
```

![Screenshot: Applying the NFS test manifest](../images/kubernetes-storage/21-apply-test-manifest.png)

### 7.3. Check the PVC status

```bash
kubectl get pvc nfs-test-pvc
```

![Screenshot: Checking the test PVC status](../images/kubernetes-storage/22-check-test-pvc.png)

The PVC should reach the `Bound` state.

If it remains `Pending`, inspect the PVC events and provisioner logs before proceeding.

### 7.4. Check the PersistentVolume

```bash
kubectl get pv
```

![Screenshot: Checking dynamically provisioned PersistentVolumes](../images/kubernetes-storage/23-check-persistent-volumes.png)

Identify the PV associated with `nfs-test-pvc` and verify that it is bound to the claim.

### 7.5. Check the test Pod

```bash
kubectl get pod nfs-test-pod -o wide
```

![Screenshot: Checking the NFS test Pod status](../images/kubernetes-storage/24-check-test-pod.png)

The Pod should reach the `Running` state.

If the Pod is not running, inspect its events:

```bash
kubectl describe pod nfs-test-pod
```

### 7.6. Inspect the test Pod logs

```bash
kubectl logs nfs-test-pod
```

![Screenshot: Inspecting the NFS test Pod logs](../images/kubernetes-storage/25-check-test-logs.png)

The command may return no output because the test script redirects its message to `/data/test.txt` instead of standard output.

### 7.7. Read the test file

```bash
kubectl exec nfs-test-pod -- cat /data/test.txt
```

![Screenshot: Reading the file from the mounted NFS volume](../images/kubernetes-storage/26-verify-test-file.png)

Expected output:

```text
NFS fonctionne
```

This confirms that the Pod can write to and read from the mounted volume.

### 7.8. Verify the file on the NFS server

Run this command directly on the NFS server, not on the Kubernetes master unless both roles are hosted on the same machine:

```bash
sudo find /srv/nfs/harbor -type f -name test.txt
```

![Screenshot: Verifying the test file on the NFS server](../images/kubernetes-storage/27-verify-file-on-nfs-server.png)

The provisioner typically creates a dedicated subdirectory for the PVC. Therefore, the file may be located several levels below the exported directory.

### 7.9. Verify data persistence

To confirm that the data is stored on the persistent volume rather than only inside the container, delete the test Pod while keeping the PVC.

```bash
kubectl delete pod nfs-test-pod
```

Wait until the Pod has been deleted, then recreate it using the existing manifest:

```bash
kubectl apply -f nfs-test.yaml
```

Wait for the Pod to become ready:

```bash
kubectl wait \
  --for=condition=Ready \
  pod/nfs-test-pod \
  --timeout=120s
```

Read the existing file again:

```bash
kubectl exec nfs-test-pod -- cat /data/test.txt
```

Expected output:

```text
NFS fonctionne
```

The file should still exist because the PVC and its underlying storage were preserved when the original Pod was deleted.

This test validates data persistence across Pod replacement.

### 7.10. Understand the storage capacity limitation

The `1Gi` storage request specifies the amount of storage requested by the PVC. It does not automatically enforce a `1Gi` physical quota on the NFS directory.

Monitor the available disk space on the NFS server and configure filesystem quotas separately if strict capacity enforcement is required.

---

## 8. Troubleshooting

### 8.1. PVC remains Pending

Inspect the claim:

```bash
kubectl describe pvc nfs-test-pvc
```

Check the StorageClass:

```bash
kubectl describe storageclass nfs-harbor
```

Check recent events:

```bash
kubectl get events --sort-by=.lastTimestamp
```

Possible causes include:

- The StorageClass does not exist.
- The PVC references an incorrect StorageClass name.
- The provisioner is not running.
- The provisioner cannot access the NFS server or export path.
- The NFS directory permissions prevent subdirectory creation.

### 8.2. Provisioner Pod fails to start

Inspect the Pod status:

```bash
kubectl get pods -n storage-system -o wide
```

Describe the Pod:

```bash
kubectl describe pod -n storage-system POD_NAME
```

Retrieve the provisioner logs:

```bash
kubectl logs -n storage-system deployment/DEPLOYMENT_NAME
```

Check the events for image-pulling errors, scheduling failures, configuration errors, or permission problems.

Replace `POD_NAME` and `DEPLOYMENT_NAME` with the actual resource names.

### 8.3. Pod cannot mount the NFS volume

Inspect the test Pod:

```bash
kubectl describe pod nfs-test-pod
```

Check the following:

- The NFS server address is correct.
- The exported directory exists.
- The NFS server allows connections from the Kubernetes nodes.
- The required NFS client utilities are installed on the nodes where the workload runs.
- The export permissions allow the required operations.
- Network routing and firewall rules permit NFS traffic.

For NFSv4, TCP port `2049` is commonly used. Other NFS configurations may require additional ports.

### 8.4. Pod cannot write to the mounted directory

Inspect the permissions on the NFS server:

```bash
sudo ls -ld /srv/nfs/harbor
```

Inspect the export configuration:

```bash
sudo exportfs -v
```

Review the ownership, directory permissions, and NFS export rules.

Do not assume that `chmod 777` is an appropriate permanent solution. Configure permissions according to the required security model and the identities used by the workloads.

### 8.5. Check connectivity from a Kubernetes node

Run the following command on the node where the workload is scheduled:

```bash
showmount -e 192.168.1.20
```

This can help verify the NFS export configuration when the server supports the relevant mount protocol discovery.

Note that `showmount` may not work with every NFSv4-only configuration. A failed `showmount` command alone does not prove that the NFS server is unreachable.

---

## 9. Clean Up the Test Resources

Perform this cleanup only after confirming that the test data is no longer required.

### 9.1. Delete the test Pod

```bash
kubectl delete pod nfs-test-pod
```

![Screenshot: Deleting the NFS test Pod](../images/kubernetes-storage/32-delete-test-pod.png)

### 9.2. Delete the test PVC

```bash
kubectl delete pvc nfs-test-pvc
```

![Screenshot: Deleting the test PVC](../images/kubernetes-storage/33-delete-test-pvc.png)

### 9.3. Verify the PersistentVolume status

```bash
kubectl get pv
```

Because the StorageClass uses the `Retain` reclaim policy, the PersistentVolume may transition to the `Released` state after the claim is deleted.

This is expected behavior. The retained volume may still reference the underlying NFS directory and its data.

**Warning:** Do not manually delete the PV or remove its NFS directory until you have confirmed that the data is disposable and that no application depends on it.

The `Retain` policy helps preserve data, but it also requires deliberate cleanup and lifecycle management.

---

## 10. Prepare Storage for Harbor

After the PVC test succeeds, the NFS provisioner is ready to provide persistent volumes for applications that use the `nfs-harbor` StorageClass.

Before proceeding with Harbor installation, verify that:

- The provisioner is running and healthy.
- The `nfs-harbor` StorageClass exists.
- The PVC provisioning and mounting tests succeeded.
- Data remains available after Pod replacement.
- The NFS export has sufficient free disk space.
- The NFS permissions are appropriate for the intended workloads.
- The storage configuration is documented and reproducible.

During Harbor deployment, the relevant persistent storage settings must explicitly reference the configured StorageClass where appropriate.

The exact PVC requirements depend on the Harbor chart version and the persistence configuration selected for its components. Review the Harbor chart values before configuring those claims.

> **Next step:** Configure Harbor persistence and deploy Harbor using Helm. TLS, Ingress, DNS, and registry access will be documented separately.

---

## References

- [NFS Subdirectory External Provisioner — GitHub Repository](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)
- [NFS Subdirectory External Provisioner — Helm Chart Repository](https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/)
- [Kubernetes PersistentVolumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Kubernetes StorageClasses](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [Kubernetes PersistentVolumeClaims](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)
- [Harbor Documentation](https://goharbor.io/docs/)

---

## Summary

At the end of this procedure, the Kubernetes cluster should have a functional NFS-backed StorageClass capable of dynamically provisioning PersistentVolumes.

The validation confirms that a PVC can be bound, a Pod can mount the volume, data can be written and read, and the data remains available after Pod replacement.

This establishes the storage foundation required for the next stage: deploying Harbor with persistent storage.
