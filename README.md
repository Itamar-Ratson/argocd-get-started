# ArgoCD Getting Started Guide (Gateway API)

Local development setup using KinD, Cilium (with WireGuard encryption), Traefik, Helm, cert-manager, Sealed Secrets, and Kubernetes Gateway API.

## Prerequisites

- Docker
- kubectl
- Helm
- KinD
- htpasswd (from apache2-utils)
- kubeseal

## 1. Create KinD Cluster

Create `kind-config.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: argocd-dev
networking:
  disableDefaultCNI: true
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
```

```bash
kind create cluster --config kind-config.yaml
```

The cluster won't be Ready until a CNI is installed (next step).

## 2. Install Cilium (CNI with WireGuard Encryption)

Cilium provides encrypted pod-to-pod communication using WireGuard.

Create `helm/cilium/Chart.yaml`:

```yaml
apiVersion: v2
name: cilium
version: 1.0.0
description: Cilium CNI with WireGuard encryption
dependencies:
  - name: cilium
    version: "1.16.5"
    repository: "https://helm.cilium.io"
```

Create `helm/cilium/values.yaml`:

```yaml
cilium:
  # Enable WireGuard encryption for pod-to-pod traffic
  encryption:
    enabled: true
    type: wireguard

  # Required for hostPort support when replacing default CNI
  hostPort:
    enabled: true

  # Required for KinD single-node clusters
  socketLB:
    hostNamespaceOnly: true

  # Single node cluster only needs one operator replica
  operator:
    replicas: 1
```

Build and install:

```bash
helm repo add cilium https://helm.cilium.io
helm repo update
helm dependency build ./helm/cilium
helm install cilium ./helm/cilium -n kube-system -f ./helm/cilium/values.yaml
kubectl wait --for=condition=Ready nodes --all --timeout=300s
```

Verify encryption is active:

```bash
kubectl -n kube-system exec ds/cilium -- cilium-dbg encrypt status
```

## 3. Install Gateway API CRDs

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
```

## 4. Install cert-manager

Create `helm/cert-manager/Chart.yaml`:

```yaml
apiVersion: v2
name: cert-manager
version: 1.0.0
description: cert-manager with self-signed ClusterIssuer
dependencies:
  - name: cert-manager
    version: "1.17.1"
    repository: "https://charts.jetstack.io"
```

Create `helm/cert-manager/values.yaml`:

```yaml
cert-manager:
  crds:
    enabled: true
```

Create `helm/cert-manager/templates/cluster-issuer.yaml`:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned
  annotations:
    "helm.sh/hook": post-install,post-upgrade
    "helm.sh/hook-weight": "1"
spec:
  selfSigned: {}
```

Build and install:

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm dependency build ./helm/cert-manager
helm upgrade --install cert-manager ./helm/cert-manager -n cert-manager --create-namespace
kubectl wait --for=condition=Available deployment/cert-manager -n cert-manager --timeout=120s
```

## 5. Install Sealed Secrets

Create `helm/sealed-secrets/Chart.yaml`:

```yaml
apiVersion: v2
name: sealed-secrets
version: 1.0.0
description: Sealed Secrets controller for encrypting secrets in Git
dependencies:
  - name: sealed-secrets
    version: "2.17.1"
    repository: "https://bitnami-labs.github.io/sealed-secrets"
```

Create `helm/sealed-secrets/values.yaml`:

```yaml
sealed-secrets: {}
```

Build and install:

```bash
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo update
helm dependency build ./helm/sealed-secrets
helm upgrade --install sealed-secrets ./helm/sealed-secrets -n sealed-secrets --create-namespace
kubectl wait --for=condition=Available deployment/sealed-secrets -n sealed-secrets --timeout=120s
```

Install kubeseal CLI:

```bash
KUBESEAL_VERSION=$(curl -s https://api.github.com/repos/bitnami-labs/sealed-secrets/releases/latest | grep -Po '"tag_name": "v\K[^"]*')
curl -OL "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION}/kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz"
tar -xvzf kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
rm kubeseal kubeseal-*.tar.gz
```

Verify:

```bash
kubeseal --controller-name sealed-secrets --controller-namespace sealed-secrets --fetch-cert
```

## 6. Install Traefik

Create `helm/traefik/Chart.yaml`:

```yaml
apiVersion: v2
name: traefik
version: 1.0.0
description: Traefik ingress controller with Gateway API support
dependencies:
  - name: traefik
    version: "34.3.0"
    repository: "https://traefik.github.io/charts"
```

Create `helm/traefik/values.yaml`:

```yaml
traefik:
  # Run in host network namespace for direct port binding
  # This avoids Cilium hostPort eBPF complexity in KinD
  hostNetwork: true

  deployment:
    # Recreate strategy required for hostNetwork deployments
    # RollingUpdate would fail as old pod holds the port
    strategy:
      type: Recreate

  # With hostNetwork, ports are bound directly on the host
  ports:
    web:
      port: 80
    websecure:
      port: 443

  nodeSelector:
    ingress-ready: "true"

  tolerations:
    - key: node-role.kubernetes.io/control-plane
      operator: Equal
      effect: NoSchedule

  providers:
    kubernetesGateway:
      enabled: true

  gateway:
    enabled: false
```

Create `helm/traefik/templates/gateway-class.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: traefik
spec:
  controllerName: traefik.io/gateway-controller
```

Create `helm/traefik/templates/gateway.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: traefik-gateway
  namespace: {{ .Release.Namespace }}
spec:
  gatewayClassName: traefik
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        mode: Terminate
        certificateRefs:
          - name: argocd-tls
      allowedRoutes:
        namespaces:
          from: All
```

Create `helm/traefik/templates/certificate.yaml`:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: argocd-tls
  namespace: {{ .Release.Namespace }}
spec:
  secretName: argocd-tls
  issuerRef:
    name: selfsigned
    kind: ClusterIssuer
  dnsNames:
    - argocd.localhost
```

Build and install:

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
helm dependency build ./helm/traefik
helm upgrade --install traefik ./helm/traefik -n traefik --create-namespace
kubectl wait --for=condition=Available deployment/traefik -n traefik --timeout=120s
```

## 7. Install ArgoCD

Create `helm/argocd/Chart.yaml`:

```yaml
apiVersion: v2
name: argocd
version: 1.0.0
description: ArgoCD with Gateway API HTTPRoute
dependencies:
  - name: argo-cd
    version: "7.7.16"
    repository: "https://argoproj.github.io/argo-helm"
```

Create `helm/argocd/values.yaml`:

```yaml
argo-cd:
  dex:
    enabled: false

  notifications:
    enabled: false

  applicationSet:
    enabled: false

  configs:
    params:
      server.insecure: true
    secret:
      createSecret: false

httproute:
  enabled: true
  gateway:
    name: traefik-gateway
    namespace: traefik
  hostname: argocd.localhost
```

Create `helm/argocd/templates/httproute.yaml`:

```yaml
{{- if .Values.httproute.enabled }}
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: argocd-server
  namespace: {{ .Release.Namespace }}
spec:
  parentRefs:
    - name: {{ .Values.httproute.gateway.name }}
      namespace: {{ .Values.httproute.gateway.namespace }}
  hostnames:
    - {{ .Values.httproute.hostname }}
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: argocd-server
          port: 80
{{- end }}
```

### Set Custom Admin Password

Create `example.argocd-admin-secret.yaml`:

```yaml
# Example ArgoCD admin password secret
#
# Usage:
#   1. Copy this file: cp example.argocd-admin-secret.yaml argocd-admin-secret.yaml
#
#   2. Generate a bcrypt hash for your password:
#      htpasswd -nbBC 10 "" "your-password" | tr -d ':\n' | sed 's/$2y/$2a/'
#
#   3. Generate a secret key for JWT signing:
#      openssl rand -base64 32
#
#   4. Replace the values below with your generated hash and secret key
#
#   5. Seal it:
#      kubeseal --controller-name sealed-secrets --controller-namespace sealed-secrets \
#        -o yaml < argocd-admin-secret.yaml > helm/argocd/templates/sealed-argocd-secret.yaml
#
#   6. Delete the plain file: rm argocd-admin-secret.yaml
#
# Quick setup with default password "admin":
#   HASH=$(htpasswd -nbBC 10 "" "admin" | tr -d ':\n' | sed 's/$2y/$2a/')
#   SECRET_KEY=$(openssl rand -base64 32)
#   sed -e "s|REPLACE_WITH_BCRYPT_HASH|$HASH|" \
#       -e "s|REPLACE_WITH_SECRET_KEY|$SECRET_KEY|" \
#       example.argocd-admin-secret.yaml > argocd-admin-secret.yaml
#
apiVersion: v1
kind: Secret
metadata:
  name: argocd-secret
  namespace: argocd
type: Opaque
stringData:
  admin.password: "REPLACE_WITH_BCRYPT_HASH"
  admin.passwordMtime: "2024-01-01T00:00:00Z"
  server.secretkey: "REPLACE_WITH_SECRET_KEY"
```

Create the ArgoCD admin secret using Sealed Secrets:

```bash
# Generate values and create secret file
HASH=$(htpasswd -nbBC 10 "" "admin" | tr -d ':\n' | sed 's/$2y/$2a/')
SECRET_KEY=$(openssl rand -base64 32)
sed -e "s|REPLACE_WITH_BCRYPT_HASH|$HASH|" \
    -e "s|REPLACE_WITH_SECRET_KEY|$SECRET_KEY|" \
    example.argocd-admin-secret.yaml > argocd-admin-secret.yaml

# Seal the secret
kubeseal --controller-name sealed-secrets --controller-namespace sealed-secrets \
  -o yaml < argocd-admin-secret.yaml > helm/argocd/templates/sealed-argocd-secret.yaml

# Delete the plain secret
rm argocd-admin-secret.yaml
```

Build and install:

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm dependency build ./helm/argocd
helm upgrade --install argocd ./helm/argocd -n argocd --create-namespace
kubectl wait --for=condition=Available deployment/argocd-server -n argocd --timeout=300s
```

## 8. Access ArgoCD

Access UI at <https://argocd.localhost> (accept the self-signed certificate warning).

Login with username `admin` and password `admin` (or whatever password you set).

CLI login:

```bash
argocd login argocd.localhost --grpc-web
```

## Troubleshooting

### Verify Cilium encryption

```bash
kubectl -n kube-system exec ds/cilium -- cilium-dbg encrypt status
```

### Check Gateway status

```bash
kubectl get gateway -n traefik
```

The Gateway should show `PROGRAMMED: True`.

### Check HTTPRoute status

```bash
kubectl get httproute -n argocd
```

### Check certificate

```bash
kubectl get certificate -n traefik
```

### Traefik logs

```bash
kubectl logs -n traefik -l app.kubernetes.io/name=traefik
```

### If password doesn't work

Regenerate and patch the secret directly:

```bash
HASH=$(htpasswd -nbBC 10 "" "admin" | tr -d ':\n' | sed 's/$2y/$2a/')
kubectl patch secret argocd-secret -n argocd \
  -p "{\"stringData\": {\"admin.password\": \"$HASH\", \"admin.passwordMtime\": \"$(date -Iseconds)\"}}"
kubectl rollout restart deployment/argocd-server -n argocd
```

## Cleanup

```bash
kind delete cluster --name argocd-dev
```
