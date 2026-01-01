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

Verify installation:

```bash
kubectl get crd | grep gateway
```

## 3. Install Traefik

Create `traefik-values.yaml`:

```yaml
# Map host ports directly to Traefik's container ports
ports:
  web:
    hostPort: 80      # Host port 80 -> container port 8000 (Traefik's default)
  websecure:
    hostPort: 443     # Host port 443 -> container port 8443 (Traefik's default)

# Schedule on the node with KinD's extraPortMappings
nodeSelector:
  ingress-ready: "true"

# Allow scheduling on control-plane node
tolerations:
  - key: node-role.kubernetes.io/control-plane
    operator: Equal
    effect: NoSchedule

# Enable Gateway API provider
providers:
  kubernetesGateway:
    enabled: true

# We create our own Gateway resource
gateway:
  enabled: false
```

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
helm install traefik traefik/traefik -n traefik --create-namespace -f traefik-values.yaml
```

## 4. Install ArgoCD

Create `argocd-values.yaml`:

```yaml
dex:
  enabled: false

notifications:
  enabled: false

applicationSet:
  enabled: false

configs:
  params:
    server.insecure: true
```

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
kubectl create namespace argocd
helm install argocd argo/argo-cd -n argocd -f argocd-values.yaml
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=300s
```

## 5. Create GatewayClass, Gateway, and HTTPRoute

Create `gateway-class.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: traefik
spec:
  controllerName: traefik.io/gateway-controller
```

Create `gateway.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: traefik-gateway
  namespace: traefik
spec:
  gatewayClassName: traefik
  listeners:
    - name: http
      protocol: HTTP
      port: 8000
      allowedRoutes:
        namespaces:
          from: All
```

Create `argocd-httproute.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: argocd-server
  namespace: argocd
spec:
  parentRefs:
    - name: traefik-gateway
      namespace: traefik
  hostnames:
    - argocd.localhost
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: argocd-server
          port: 80
```

Apply resources:

```bash
kubectl apply -f gateway-class.yaml
kubectl apply -f gateway.yaml
kubectl apply -f argocd-httproute.yaml
```

Verify:

```bash
kubectl get gatewayclass
kubectl get gateway -n traefik
kubectl get httproute -n argocd
```

## 6. Access ArgoCD

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
