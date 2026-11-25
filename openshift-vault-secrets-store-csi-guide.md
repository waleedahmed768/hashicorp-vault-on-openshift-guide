# OpenShift Secrets Store CSI Driver with HashiCorp Vault

## 1. Overview

This guide summarizes the procedure for integrating HashiCorp Vault with the OpenShift Secrets Store CSI Driver and mounting Vault secrets into application pods.

| Component | Role |
|---|---|
| Secrets Store CSI Driver Operator | Installs and manages the CSI driver on OpenShift. |
| `ClusterCSIDriver` | Requests management of the `secrets-store.csi.k8s.io` driver. |
| HashiCorp Vault | Stores and manages application secrets. |
| Vault CSI provider | Connects the CSI driver to Vault and retrieves secrets. |
| `SecretProviderClass` | Defines which Vault secret and key to mount. |
| Application Deployment | Mounts the secret into the application container filesystem. |

The CSI driver communicates with the provider over gRPC. The source documentation states that the driver polls for secret updates every 2 minutes; applications consuming mounted secrets must watch for updates.

## 2. Prerequisites

- Access to the OpenShift web console.
- Cluster-admin access to OpenShift.
- Helm installed.
- A Vault namespace.
- The Secrets Store CSI Driver Operator installed.

## 3. Install the Secrets Store CSI Driver Operator

In the OpenShift web console:

1. Go to **Operators → OperatorHub**.
2. Search for **Secrets Store CSI Driver Operator**.
3. Select the operator and click **Install**.
4. Select **All namespaces on the cluster (default)**.
5. Set the installed namespace to `openshift-cluster-csi-drivers`.
6. Click **Install**.

### Create the ClusterCSIDriver instance

Save as `secrets-store-csi-driver.yaml`:

```yaml
apiVersion: operator.openshift.io/v1
kind: ClusterCSIDriver
metadata:
    name: secrets-store.csi.k8s.io
spec:
  managementState: Managed
```

Apply it:

```bash
oc apply -f secrets-store-csi-driver.yaml
```

## 4. Install the HashiCorp Vault provider

Add the HashiCorp Helm repository:

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
```

Update repositories:

```bash
helm repo update
```

Create the Vault project:

```bash
oc new-project vault
```

Label the Vault namespace for pod security admission:

```bash
oc label ns vault security.openshift.io/scc.podSecurityLabelSync=false pod-security.kubernetes.io/enforce=privileged pod-security.kubernetes.io/audit=privileged pod-security.kubernetes.io/warn=privileged --overwrite
```

Grant privileged access to the Vault service account:

```bash
oc adm policy add-scc-to-user privileged -z vault -n vault
```

Grant privileged access to the Vault CSI provider service account:

```bash
oc adm policy add-scc-to-user privileged -z vault-csi-provider -n vault
```

Install Vault using Helm:

```bash
helm install vault hashicorp/vault --namespace=vault \
  --set "server.dev.enabled=true" \
  --set "injector.enabled=false" \
  --set "csi.enabled=true" \
  --set "global.openshift=true" \
  --set "injector.agentImage.repository=docker.io/hashicorp/vault" \
  --set "server.image.repository=docker.io/hashicorp/vault" \
  --set "csi.image.repository=docker.io/hashicorp/vault-csi-provider" \
  --set "csi.agent.image.repository=docker.io/hashicorp/vault" \
  --set "csi.daemonSet.providersDir=/var/run/secrets-store-csi-providers"
```

Patch the Vault CSI provider DaemonSet to set the first container's security context to privileged:

```bash
oc patch daemonset -n vault vault-csi-provider --type='json' -p='[{"op": "add", "path": "/spec/template/spec/containers/0/securityContext", "value": {"privileged": true} }]'
```

Check the Vault pods:

```bash
oc get pods -n vault
```

## 5. Create and verify a secret in Vault

Create the example secret:

```bash
oc exec vault-0 --namespace=vault -- vault kv put secret/example1 testSecret1=my-secret-value
```

Read it back:

```bash
oc exec vault-0 --namespace=vault -- vault kv get secret/example1
```

The example uses the KV v2 data path `secret/data/example1` and key `testSecret1`.

## 6. Configure Kubernetes authentication in Vault

Enable Kubernetes authentication:

Why do we enable Kubernetes authentication in HashiCorp Vault?
In this integration, OpenShift (Kubernetes) authentication allows Vault to verify which pod is requesting a secret.

Vault and OpenShift are two separate systems:
- OpenShift manages pods, namespaces and ServiceAccounts.
- Vault stores sensitive information such as passwords, API keys and certificates.
- Secrets Store CSI Driver connects the pod to Vault and mounts the requested secret as a file.
Vault needs a secure way to identify and authenticate the pod before allowing it to read a secret. That is why we enable the Kubernetes authentication method.
How the authentication flow works
1. Application pod

Uses a Kubernetes ServiceAccount

ServiceAccount JWT token

2. Secrets Store CSI Driver

Requests the secret from the Vault provider

Authentication request

3. HashiCorp Vault

Validates the ServiceAccount identity with the Kubernetes API

If authorized, Vault issues a token

4. Secret mounted in the pod

The application reads the secret as a file

For example, suppose your application runs in namespace my-namespace using ServiceAccount default.

You configure Vault to allow that ServiceAccount to read secrets under secret/data/*. A pod using a different ServiceAccount or an unauthorized namespace will not be allowed by that role.

The important point is that the application does not need a hardcoded Vault username and password. Vault uses the pod's Kubernetes identity to authenticate it.



```bash
oc exec vault-0 --namespace=vault -- vault auth enable kubernetes

```



Set the token reviewer JWT:
Why do we need this token?

Vault needs permission to ask the Kubernetes API whether a ServiceAccount token presented by an application is valid.

This is called the TokenReview process.

```bash
TOKEN_REVIEWER_JWT="$(oc exec vault-0 --namespace=vault -- cat /var/run/secrets/kubernetes.io/serviceaccount/token)"
```

Get the Kubernetes service IP:

Why do we need this?

Vault must communicate with the Kubernetes API to validate application ServiceAccount tokens.

This command retrieves the internal ClusterIP of the Kubernetes API service.

For example, it might return:


```bash
KUBERNETES_SERVICE_IP="$(oc get svc kubernetes --namespace=default -o go-template="{{ .spec.clusterIP }}")"
```

Configure the Kubernetes authentication method:

This is one of the most important commands.

It tells Vault how to communicate with and validate identities against Kubernetes.
Parameter	Explanation
auth/kubernetes/config	Vault's Kubernetes authentication configuration endpoint.
issuer	Expected issuer of the Kubernetes ServiceAccount JWT.
token_reviewer_jwt	Token Vault uses to call Kubernetes TokenReview API.
kubernetes_host	Kubernetes API server address.
kubernetes_ca_cert	CA certificate used to verify the Kubernetes API server's TLS certificate.


```bash
oc exec -i vault-0 --namespace=vault -- vault write auth/kubernetes/config \
  issuer="https://kubernetes.default.svc.cluster.local" \
  token_reviewer_jwt="${TOKEN_REVIEWER_JWT}" \
  kubernetes_host="https://${KUBERNETES_SERVICE_IP}:443" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

## 7. Create a Vault policy

Create the `csi` policy:

```bash
oc exec -i vault-0 --namespace=vault -- vault policy write csi -<<EOF
  path "secret/data/*" {
  capabilities = ["read"]
  }
  EOF
```

This policy grants read access to secrets under `secret/data/*`.

## 8. Create a Kubernetes authentication role

Create the `csi` role:

```bash
oc exec -i vault-0 --namespace=vault -- vault write auth/kubernetes/role/csi \
  bound_service_account_names=default \
  bound_service_account_namespaces=default,test-ns,negative-test-ns,my-namespace \
  policies=csi \
  ttl=20m
```

The role allows the Kubernetes ServiceAccount named `default` in the listed namespaces to authenticate and receive the `csi` policy, with a token TTL of 20 minutes.

### Add another namespace to the existing role

Rewrite the role with the complete namespace list, including all existing namespaces and the new namespace. The list is replaced by the new value; it is not appended automatically.

For example, to add `sap-namespace`, use:

```bash
oc exec -i vault-0 --namespace=vault -- vault write auth/kubernetes/role/csi \
  bound_service_account_names=default \
  bound_service_account_namespaces=default,test-ns,negative-test-ns,my-namespace,sap-namespace \
  policies=csi \
  ttl=20m
```

Verify the role:

```bash
oc exec vault-0 --namespace=vault -- vault read auth/kubernetes/role/csi
```

## 9. Create the SecretProviderClass

Save as `secret-provider-class-vault.yaml`:

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: my-vault-provider
  namespace: my-namespace
spec:
  provider: vault
  parameters:
    roleName: "csi"
    vaultAddress: "http://vault.vault:8200"
    objects: |
      - secretPath: "secret/data/example1"
        objectName: "testSecret1"
        secretKey: "testSecret1"
```

Key fields:

- `metadata.name`: Name of the SecretProviderClass.
- `metadata.namespace`: Namespace containing the SecretProviderClass; it must match the application pod namespace.
- `provider`: Specifies the Vault provider.
- `parameters`: Contains the Vault role, Vault address, secret path, mounted object name, and secret key.

Create it:

```bash
oc create -f secret-provider-class-vault.yaml
```

## 10. Create an application Deployment

Save as `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: busybox-deployment
  namespace: my-namespace
  labels:
    app: busybox
spec:
  replicas: 1
  selector:
    matchLabels:
      app: busybox
  template:
    metadata:
      labels:
        app: busybox
    spec:
      terminationGracePeriodSeconds: 0
      containers:
      - image: registry.k8s.io/e2e-test-images/busybox:1.29-4
        name: busybox
        imagePullPolicy: IfNotPresent
        command:
        - "/bin/sleep"
        - "10000"
        volumeMounts:
        - name: secrets-store-inline
          mountPath: "/mnt/secrets-store"
          readOnly: true
      volumes:
        - name: secrets-store-inline
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: "my-vault-provider"
```

The Deployment namespace must match the SecretProviderClass namespace. The `secretProviderClass` value must match its metadata name.

Create the Deployment:

```bash
oc create -f deployment.yaml
```

## 11. Verify the installation and mounted secret

Check Vault pods:

```bash
oc get pods -n vault
```

Check Secrets Store CSI Driver pods:

```bash
oc get pods -n openshift-cluster-csi-drivers | grep -E "secrets"
```

Check the mounted file (replace `<hash>` with the actual pod suffix):

```bash
oc exec busybox-deployment-<hash> -n my-namespace -- ls /mnt/secrets-store/
```

Example output:

```text
testSecret1
```

Read the secret:

```bash
oc exec busybox-deployment-<hash> -n my-namespace -- cat /mnt/secrets-store/testSecret1
```

Example output:

```text
my-secret-value
```

## 12. Troubleshooting: CSI driver not registered

If a pod reports an error similar to:

```text
MountVolume.SetUp failed for volume "secrets-store-inline":
kubernetes.io/csi: mounter.SetUpAt failed to get CSI client:
driver name secrets-store.csi.k8s.io not found in the list of registered CSI drivers
```

Check registered CSI drivers:

```bash
oc get csidrivers
```

Check the driver and operator pods:

```bash
oc get pods -n openshift-cluster-csi-drivers | grep -E "secrets"
```

Check the ClusterCSIDriver resource:

```bash
oc get clustercsidriver
```

Confirm that the `ClusterCSIDriver` named `secrets-store.csi.k8s.io` exists with `managementState: Managed`, and that CSI driver node pods are running on the nodes where application pods are scheduled.

## 13. Important notes

- The documented Helm command sets `server.dev.enabled=true`. Vault development mode is for testing, not production.
- The policy uses the KV v2 path `secret/data/*` and grants read access.
- The role permits the `default` ServiceAccount in the namespaces listed in the role.
- When adding a namespace, include every existing namespace you want to retain in the updated role command.
- The Vault CSI provider and the Secrets Store CSI Driver are separate components. The provider does not replace the node CSI driver.
- The source documentation states that mounted secret content is polled for updates every 2 minutes. Applications must watch for changes if they need to react to rotated content.
- The source documentation says Vault provider integration was tested with AWS and Azure cloud providers; other cloud providers might work but were not stated as tested there.
