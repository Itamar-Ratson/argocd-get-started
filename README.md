# ArgoCD Getting Started Guide (Gateway API)

Local development setup using KinD, Traefik, Helm, and Kubernetes Gateway API.

## Prerequisites

- Docker
- kubectl
- Helm
- KinD

## 1. Create KinD Cluster

Create `kind-config.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: argocd-dev
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

## 2. Install Gateway API CRDs

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
```

## 3. Create Traefik Chart

Create `charts/traefik/Chart.yaml`:

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

Create `charts/traefik/values.yaml`:

```yaml
traefik:
  ports:
    web:
      hostPort: 80
    websecure:
      hostPort: 443

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

Create `charts/traefik/templates/gateway-class.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: traefik
spec:
  controllerName: traefik.io/gateway-controller
```

Create `charts/traefik/templates/gateway.yaml`:

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
      port: 8443
      tls:
        mode: Terminate
        certificateRefs:
          - name: argocd-tls
      allowedRoutes:
        namespaces:
          from: All
```

Build and install:

```bash
helm dependency build ./charts/traefik
helm upgrade --install traefik ./charts/traefik -n traefik --create-namespace
```

## 4. Create ArgoCD Chart

Create `charts/argocd/Chart.yaml`:

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

Create `charts/argocd/values.yaml`:

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

httproute:
  enabled: true
  gateway:
    name: traefik-gateway
    namespace: traefik
  hostname: argocd.localhost
```

Create `charts/argocd/templates/httproute.yaml`:

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

Build and install:

```bash
helm dependency build ./charts/argocd
helm upgrade --install argocd ./charts/argocd -n argocd --create-namespace
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=300s
```

## 5. Access ArgoCD

Get admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

Access UI at <http://argocd.localhost>

CLI login:

```bash
argocd login argocd.localhost --grpc-web --insecure
```

## Cleanup

```bash
kind delete cluster --name argocd-dev
```
