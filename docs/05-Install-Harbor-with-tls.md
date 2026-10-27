# 05 — Install Harbor on Kubernetes with Helm, TLS, Traefik and NFS

## Overview

This guide explains how to generate a local Certificate Authority (CA), create a TLS certificate for Harbor, configure Kubernetes Secrets, and deploy Harbor using Helm in a Kubernetes cluster.

Harbor will be exposed through Traefik using the hostname `harbor.k8s.local`. MetalLB provides the external IP address for Traefik, BIND9 resolves the Harbor hostname, and persistent data is stored using the existing NFS-backed Kubernetes StorageClass `nfs-harbor`.

### Architecture

- **Kubernetes:** Runs the Harbor components.
- **Traefik:** Routes incoming HTTPS requests to Harbor through an Ingress.
- **MetalLB:** Provides the external IP address assigned to Traefik.
- **BIND9:** Resolves `harbor.k8s.local` to the Traefik external IP.
- **NFS:** Provides persistent storage through the `nfs-harbor` StorageClass.
- **TLS:** Protects client connections to Harbor using a locally generated certificate signed by a local CA.

**Scope:** This guide covers TLS certificate generation, Harbor installation, persistent storage configuration, and deployment verification. It assumes that Kubernetes, Traefik, MetalLB, BIND9, and the `nfs-harbor` StorageClass are already configured.

## 1. Environment

The following table describes the example environment used in this guide.

| Component | Example value |
|---|---|
| Harbor hostname | `harbor.k8s.local` |
| DNS/NFS server | `192.168.1.20` |
| Traefik external IP | `192.168.1.240` |
| Ingress controller | Traefik |
| Kubernetes namespace | `harbor` |
| NFS StorageClass | `nfs-harbor` |
| Helm release | `harbor` |
| TLS Secret | `harbor-tls` |
| Local CA Secret | `harbor-ca` |
| Administrator password Secret | `harbor-admin-password` |

These values are examples. Verify the actual Traefik IP address, IngressClass, and StorageClass before installing Harbor.

The certificate, DNS record, and Harbor Ingress must all use the same hostname: `harbor.k8s.local`.

## 2. Verify the Prerequisites

Before proceeding, verify that the cluster networking, DNS resolution, and storage configuration are ready.

### 2.1. Check the Kubernetes IngressClass

Run:

```bash
kubectl get ingressclass
```

![Screenshot: Kubernetes IngressClass](../images/harbor-tls/04-ingressclass.png)

Identify the exact IngressClass name associated with Traefik. The configuration below assumes that its name is `traefik`.

If the output shows a different name, use the actual name in the Helm values file.

### 2.2. Check Helm and OpenSSL

Verify that Helm and OpenSSL are installed:

```bash
helm version
openssl version
```

![Screenshot: Helm and OpenSSL Versions](../images/harbor-tls/05-helm-openssl-versions.png)

These tools are required to deploy Harbor and generate the local TLS certificates.

### 2.3. Verify the NFS StorageClass

Check that the expected StorageClass exists:

```bash
kubectl get storageclass nfs-harbor
```

The `nfs-harbor` StorageClass must be available before Harbor's PersistentVolumeClaims can be provisioned.

If the StorageClass is missing, return to the NFS storage configuration and resolve the issue before continuing.

Also verify that `harbor.k8s.local` resolves to the current Traefik external IP address and that Traefik has an assigned external IP.

## 3. Generate a Local Certificate Authority and Harbor Certificate

The following steps create a local Certificate Authority and a server certificate for `harbor.k8s.local`.

Run these commands on the Kubernetes master node or another trusted administrative machine with OpenSSL installed.

**Security warning:** The CA private key can sign certificates for other services. Keep it protected and never commit it to GitHub.

### 3.1. Create the working directory

Create a dedicated directory for the certificate files:

```bash
mkdir -p "$HOME/harbor-tls"
chmod 700 "$HOME/harbor-tls"
cd "$HOME/harbor-tls"
```

The directory permissions restrict access to the current user.

### 3.2. Generate the local CA

Generate a self-signed CA certificate and its private key:

```bash
openssl req -x509 -newkey rsa:4096 -sha256 -days 3650 -nodes \
  -keyout ca.key \
  -out ca.crt \
  -subj "/CN=Harbor Local CA"
```

![Screenshot: Local CA Generation](../images/harbor-tls/09-ca-generation.png)

The generated files are:

- `ca.key`: the CA private key. Keep it secret and restrict access.
- `ca.crt`: the public CA certificate. This can be distributed to clients that need to trust Harbor's certificate.

The `-nodes` option creates an unencrypted private key. This simplifies automated use in the lab, but it makes file permissions and secure storage particularly important.

### 3.3. Generate the Harbor server key and certificate request

Generate the private key and Certificate Signing Request (CSR):

```bash
openssl req -new -newkey rsa:3072 -nodes \
  -keyout harbor.key \
  -out harbor.csr \
  -subj "/CN=harbor.k8s.local"
```

![Screenshot: Harbor Certificate Request](../images/harbor-tls/10-harbor-csr.png)

The files are:

- `harbor.key`: the private key used by the Harbor server certificate.
- `harbor.csr`: the certificate request that will be signed by the local CA.

### 3.4. Configure the certificate extensions

Create an extensions file defining the certificate's usage and Subject Alternative Name (SAN):

```bash
cat > harbor.ext <<'EOF'
basicConstraints=critical,CA:FALSE
keyUsage=critical,digitalSignature,keyEncipherment
extendedKeyUsage=serverAuth
subjectAltName=DNS:harbor.k8s.local
EOF
```

The SAN is essential because modern clients validate the requested hostname against the certificate's SAN entries.

The certificate must contain `DNS:harbor.k8s.local` for the hostname used to access Harbor.

### 3.5. Sign the Harbor certificate

Sign the server certificate using the local CA:

```bash
openssl x509 -req \
  -in harbor.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -out harbor.crt \
  -days 825 \
  -sha256 \
  -extfile harbor.ext
```

![Screenshot: Harbor Certificate Signing](../images/harbor-tls/12-harbor-certificate-signing.png)

Restrict access to the private keys:

```bash
chmod 600 ca.key harbor.key
```

**Security warning:** Never upload `ca.key` or `harbor.key` to GitHub. The CA private key is especially sensitive because anyone who obtains it can sign additional certificates trusted by clients that accept this CA.

The public certificates, `ca.crt` and `harbor.crt`, are not private keys. However, publish them only when appropriate for your environment.

## 4. Validate the TLS Certificate

Before creating the Kubernetes Secrets, verify that the server certificate is valid and matches the expected hostname.

### 4.1. Verify the certificate against the local CA

Run:

```bash
openssl verify -CAfile ca.crt harbor.crt
```

Expected output:

```text
harbor.crt: OK
```

This confirms that OpenSSL can validate the server certificate against the specified CA certificate.

### 4.2. Inspect the certificate details

Inspect the subject, issuer, and validity dates:

```bash
openssl x509 -in harbor.crt -noout -subject -issuer -dates
```

Verify the Subject Alternative Name:

```bash
openssl x509 -in harbor.crt -noout -ext subjectAltName
```

![Screenshot: Certificate Subject Alternative Name](../images/harbor-tls/16-certificate-san.png)

The output must contain:

```text
DNS:harbor.k8s.local
```

If the SAN is missing or incorrect, regenerate the certificate with the appropriate extensions before continuing.

### 4.3. Verify that the private key matches the certificate

Compare the public key derived from the certificate with the public key derived from the private key:

```bash
openssl x509 -in harbor.crt -pubkey -noout | openssl sha256
openssl pkey -in harbor.key -pubout | openssl sha256
```

![Screenshot: Certificate and Key Match](../images/harbor-tls/17-certificate-key-match.png)

Both commands should produce the same hash. If the hashes differ, the certificate and private key do not belong together.

## 5. Create the Harbor Namespace and Kubernetes Secrets

Harbor will be deployed in a dedicated Kubernetes namespace named `harbor`.

### 5.1. Create the namespace

Run:

```bash
kubectl create namespace harbor
```

If the namespace already exists, verify it instead of deleting or recreating it:

```bash
kubectl get namespace harbor
```

### 5.2. Create the TLS Secret

Create a Kubernetes TLS Secret from the server certificate and private key:

```bash
kubectl create secret tls harbor-tls \
  --namespace harbor \
  --cert="$HOME/harbor-tls/harbor.crt" \
  --key="$HOME/harbor-tls/harbor.key"
```

Inspect the Secret:

```bash
kubectl describe secret harbor-tls -n harbor
```

![Screenshot: Harbor TLS Secret Creation](../images/harbor-tls/19-harbor-tls-secret.png)

The Secret type should be `kubernetes.io/tls`, with the following data entries:

- `tls.crt`
- `tls.key`

The Helm chart will reference this Secret to configure TLS for the Harbor Ingress.

### 5.3. Create the CA certificate Secret

Create a Kubernetes Secret containing the public CA certificate:

```bash
kubectl create secret generic harbor-ca \
  --namespace harbor \
  --from-file=ca.crt="$HOME/harbor-tls/ca.crt"
```

![Screenshot: Harbor CA Secret Creation](../images/harbor-tls/21-harbor-ca-secret.png)

This Secret contains the public CA certificate, not the CA private key. The Harbor chart's `caSecretName` setting can use it to provide the CA download link in the portal.

### 5.4. Verify the Kubernetes Secrets

List the Secrets in the Harbor namespace:

```bash
kubectl get secrets -n harbor
```

![Screenshot: Harbor Kubernetes Secrets](../images/harbor-tls/22-harbor-secrets-list.png)

The list should include `harbor-tls` and `harbor-ca`.

If a Secret already exists, do not delete it blindly. Inspect its contents and update it deliberately if the certificate or key needs to be replaced.

## 6. Create the Harbor Administrator Password Secret

The initial Harbor administrator password should be stored in a Kubernetes Secret rather than written directly into the Helm values file.

### 6.1. Enter the administrator password

Prompt for a password without displaying it in the terminal:

```bash
read -rsp "Harbor administrator password: " HARBOR_ADMIN_PASSWORD
echo
```

![Screenshot: Harbor Administrator Password Prompt](../images/harbor-tls/23-harbor-admin-password-prompt.png)

Use a strong, unique password and store it securely.

### 6.2. Create the Kubernetes Secret

Create the Secret with the expected key:

```bash
kubectl create secret generic harbor-admin-password \
  --namespace harbor \
  --from-literal=HARBOR_ADMIN_PASSWORD="$HARBOR_ADMIN_PASSWORD"
```

Remove the password variable from the current shell:

```bash
unset HARBOR_ADMIN_PASSWORD
```

The Secret name and key must match the settings used in the Helm values file.

## 7. Add the Official Harbor Helm Repository

### 7.1. Add the repository

Add the official Harbor Helm chart repository:

```bash
helm repo add harbor https://helm.goharbor.io
```

### 7.2. Update the repository metadata

Refresh the Helm repository index:

```bash
helm repo update
```

### 7.3. Inspect the Harbor chart

Display the chart information:

```bash
helm show chart harbor/harbor
```

![Screenshot: Harbor Helm Chart Information](../images/harbor-tls/28-helm-chart-information.png)

Record the chart version. For reproducible installations, select and pin a specific version rather than relying indefinitely on whichever chart version is latest.

Inspect the default chart values:

```bash
helm show values harbor/harbor > "$HOME/harbor-values-default.yaml"
```

Review the persistence settings and confirm the configuration keys supported by the selected chart version before creating the custom values file.

The official Harbor chart uses settings such as `persistence.persistentVolumeClaim.registry.storageClass` and `persistence.persistentVolumeClaim.jobservice.jobLog.storageClass` to configure persistent storage.

## 8. Create the Harbor Helm Values File

This section configures the Harbor Ingress, TLS Secret, administrator password Secret, CA certificate Secret, and NFS-backed persistent volumes.

### 8.1. Generate the required secret values

Generate a random 16-character Harbor secret key and a random database password:

```bash
SECRET_KEY="$(openssl rand -hex 8)"
DB_PASSWORD="$(openssl rand -hex 16)"
```

The Harbor chart requires `secretKey` to be a 16-character string. The first command generates 16 hexadecimal characters.

### 8.2. Create the custom values file

Create the file using the following configuration:

```bash
cat > "$HOME/harbor-values.yaml" <<EOF
expose:
  type: ingress
  tls:
    enabled: true
    certSource: secret
    secret:
      secretName: harbor-tls
  ingress:
    className: traefik
    hosts:
      core: harbor.k8s.local
    annotations:
      traefik.ingress.kubernetes.io/router.entrypoints: websecure
      traefik.ingress.kubernetes.io/router.tls: "true"

externalURL: https://harbor.k8s.local

existingSecretAdminPassword: harbor-admin-password
existingSecretAdminPasswordKey: HARBOR_ADMIN_PASSWORD

caSecretName: harbor-ca

secretKey: "${SECRET_KEY}"

persistence:
  enabled: true
  resourcePolicy: keep
  persistentVolumeClaim:
    registry:
      storageClass: nfs-harbor
      accessMode: ReadWriteOnce
      size: 20Gi
    jobservice:
      jobLog:
        storageClass: nfs-harbor
        accessMode: ReadWriteOnce
        size: 1Gi
    database:
      storageClass: nfs-harbor
      accessMode: ReadWriteOnce
      size: 5Gi
    redis:
      storageClass: nfs-harbor
      accessMode: ReadWriteOnce
      size: 1Gi
    trivy:
      storageClass: nfs-harbor
      accessMode: ReadWriteOnce
      size: 5Gi

database:
  type: internal
  internal:
    password: "${DB_PASSWORD}"
EOF
```

Protect the values file because it contains sensitive configuration values:

```bash
chmod 600 "$HOME/harbor-values.yaml"
```

Remove the generated values from the current shell:

```bash
unset SECRET_KEY DB_PASSWORD
```

### 8.3. Understand the main configuration settings

- `expose.type: ingress` configures Harbor to be exposed through an Ingress.
- `expose.tls.secret.secretName: harbor-tls` specifies the TLS Secret created earlier.
- `expose.ingress.className: traefik` associates the Ingress with the Traefik IngressClass.
- `externalURL` sets Harbor's external URL, which is used in URLs returned to clients.
- `existingSecretAdminPassword` and `existingSecretAdminPasswordKey` specify the Secret containing the initial administrator password.
- `caSecretName` specifies the Secret containing the public CA certificate.
- `persistence.enabled: true` enables persistent storage.
- `persistence.resourcePolicy: keep` requests retention of supported persistent resources when the Helm release is deleted.
- The `storageClass` entries specify `nfs-harbor` for the configured Harbor components.
- `database.internal.password` sets the password for Harbor's internal database.

**Important configuration notes:**

1. The IngressClass name must match the actual name returned by `kubectl get ingressclass`.
2. The Traefik `websecure` entrypoint must exist and be configured to accept HTTPS traffic.
3. The TLS certificate must be valid for `harbor.k8s.local`.
4. The NFS provisioner must be able to provision the requested volumes.
5. The example storage sizes are for a lab environment. Adjust them according to the available storage capacity.
6. The values above configure TLS for client connections through the Ingress. Harbor's internal component TLS is a separate setting and is not enabled by this configuration.

The `ReadWriteOnce` access mode is used for these individual claims. This does not require the underlying NFS server to be limited to one client; it describes the Kubernetes volume access mode.

## 9. Validate the Helm Configuration Before Installation

Validate the StorageClass and render the Helm templates before deploying Harbor.

### 9.1. Check the StorageClass

Run:

```bash
kubectl get storageclass nfs-harbor
```

![Screenshot: NFS StorageClass Before Installation](../images/harbor-tls/34-nfs-storageclass-check.png)

Verify that the StorageClass exists and is configured as expected.

### 9.2. Render the Harbor manifests

Render the Kubernetes manifests without installing Harbor:

```bash
helm template harbor harbor/harbor \
  --namespace harbor \
  --values "$HOME/harbor-values.yaml" \
  > /tmp/harbor-rendered.yaml
```

If the command fails, correct the values file before proceeding.

Inspect the generated manifests for the hostname, IngressClass, and storage configuration:

```bash
grep -nE 'harbor.k8s.local|ingressClassName|storageClassName' \
  /tmp/harbor-rendered.yaml
```

![Screenshot: Harbor Helm Template Validation](../images/harbor-tls/35-harbor-helm-template.png)

Confirm that the rendered configuration contains the expected hostname, the intended IngressClass, and `nfs-harbor` storage references.

Template rendering validates chart rendering, but it does not prove that the PVCs will bind or that all Harbor components will start successfully.

## 10. Install Harbor with Helm

After the prerequisite checks and template validation succeed, install Harbor:

```bash
helm upgrade --install harbor harbor/harbor \
  --namespace harbor \
  --values "$HOME/harbor-values.yaml" \
  --wait \
  --timeout 20m
```

![Screenshot: Harbor Helm Installation](../images/harbor-tls/37-harbor-helm-installation.png)

The `helm upgrade --install` command installs the release if it does not exist or upgrades it if it already exists.

The `--wait` option makes Helm wait for relevant Kubernetes resources to become ready, subject to the timeout. If the installation times out, inspect the Pods, PVCs, events, and Helm release status before retrying.

## 11. Verify the Harbor Deployment

After installation, verify the status of Harbor's Kubernetes resources.

### 11.1. Check the Pods

Run:

```bash
kubectl get pods -n harbor
```

![Screenshot: Harbor Pods](../images/harbor-tls/38-harbor-pods.png)

Check that the Harbor components become ready. If a Pod remains in a non-ready state, inspect its events and logs.

### 11.2. Check the Services

List the Services:

```bash
kubectl get svc -n harbor
```

![Screenshot: Harbor Services](../images/harbor-tls/39-harbor-services.png)

Verify that the expected internal Services have been created.

### 11.3. Check the PersistentVolumeClaims

List the PVCs:

```bash
kubectl get pvc -n harbor
```

![Screenshot: Harbor PersistentVolumeClaims](../images/harbor-tls/40-harbor-pvcs.png)

The PVCs should eventually reach the `Bound` state.

If a claim remains `Pending`, inspect the PVC details and events. Verify the StorageClass, NFS provisioner status, server connectivity, export permissions, and available capacity.

### 11.4. Check the Ingress

List the Harbor Ingress resources:

```bash
kubectl get ingress -n harbor
```

![Screenshot: Harbor Ingress](../images/harbor-tls/41-harbor-ingress.png)

Verify that the hostname is `harbor.k8s.local` and that the Ingress uses the expected class.

### 11.5. Check the Helm release

Inspect the Helm release status:

```bash
helm status harbor -n harbor
```

![Screenshot: Harbor Helm Release Status](../images/harbor-tls/42-harbor-helm-status.png)

Use this output together with the Pod, Service, PVC, and Ingress status to assess the deployment.

## 12. Troubleshoot Pods and Storage

If a Harbor component fails to start, inspect the namespace events:

```bash
kubectl get events -n harbor --sort-by=.metadata.creationTimestamp
```

Describe a problematic Pod:

```bash
kubectl describe pod -n harbor <POD_NAME>
```

Replace `<POD_NAME>` with the actual Pod name returned by `kubectl get pods -n harbor`.

Retrieve its logs:

```bash
kubectl logs -n harbor <POD_NAME> --all-containers=true --tail=100
```

If the Pod has multiple containers and only one needs investigation, specify the appropriate container with `-c`.

For persistent storage issues, inspect the affected claim:

```bash
kubectl describe pvc <PVC_NAME> -n harbor
```

Replace `<PVC_NAME>` with the actual claim name.

Common causes include:

- The NFS provisioner is not ready.
- The `nfs-harbor` StorageClass is missing or misconfigured.
- The NFS server or export is unreachable.
- NFS directory permissions prevent access.
- The requested storage exceeds the available capacity.
- A Helm values key is unsupported by the selected chart version.

Resolve the underlying issue before attempting repeated Helm installations or upgrades.

## 13. Trust the Local CA on Client Machines

Because the Harbor certificate is signed by a local CA, client machines must trust `ca.crt` to validate the certificate without trust warnings.

Transfer **only the public CA certificate** to the client. Never distribute `ca.key` or the Harbor server's private key.

### 13.1. Configure a Linux client

On Debian or Ubuntu, copy the CA certificate to the system trust directory:

```bash
sudo cp ca.crt /usr/local/share/ca-certificates/harbor-local-ca.crt
```

Update the trusted certificates:

```bash
sudo update-ca-certificates
```

Ensure that the file copied to the client is the same CA certificate that signed `harbor.crt`.

### 13.2. Configure a Windows client

Transfer `ca.crt` securely to the Windows client.

Import the certificate into the **Trusted Root Certification Authorities** certificate store for the appropriate user or local computer, according to the trust requirements of your environment.

Only trust a CA certificate whose origin and fingerprint you have verified. Importing a CA certificate grants trust to certificates signed by that CA.

## 14. Test Harbor HTTPS Access

Test the HTTPS endpoint from a client that can resolve `harbor.k8s.local` to the current Traefik external IP.

### 14.1. Test with the CA certificate explicitly

If `ca.crt` is available at `$HOME/harbor-tls/ca.crt` on the test machine, run:

```bash
curl --cacert "$HOME/harbor-tls/ca.crt" -I https://harbor.k8s.local
```

This test explicitly tells `curl` which CA certificate to use when validating the server certificate.

### 14.2. Test using the system trust store

On a Linux client where the CA has been installed into the system trust store, run:

```bash
curl -I https://harbor.k8s.local
```

![Screenshot: Harbor HTTPS Test with System Trust](../images/harbor-tls/50-harbor-https-system-trust.png)

The exact HTTP status depends on the Harbor configuration. A successful TLS connection should not report a certificate trust or hostname mismatch error.

### 14.3. Open the Harbor web interface

Open the following URL in a browser:

`https://harbor.k8s.local`

![Screenshot: Harbor Web Interface](../images/harbor-tls/51-harbor-web-interface.png)

Sign in using the username `admin` and the administrator password configured in Step 6.

If the page is unreachable, verify DNS resolution, the Traefik external IP, the Ingress rule, the `websecure` entrypoint, and the relevant firewall rules.

If the browser reports a certificate warning, verify the certificate chain, SAN, validity dates, and client CA trust.

**Security note:** Do not use `curl -k` as a permanent solution to certificate errors. Validate the DNS record, certificate SAN, certificate chain, and client CA trust instead.

## References

- [Official Harbor Helm Chart](https://github.com/goharbor/harbor-helm)
- [Harbor Helm Chart Values](https://github.com/goharbor/harbor-helm/blob/main/values.yaml)
- [Kubernetes Ingress Documentation](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Kubernetes TLS Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [OpenSSL Documentation](https://docs.openssl.org/)
- [Traefik Kubernetes Ingress Provider](https://doc.traefik.io/traefik/providers/kubernetes-ingress/)
