
# Istio Masterclass: From Zero to Production Service Mesh

## Table of Contents

1. What is Istio
2. Service Mesh Architecture
3. Istio Components
4. Installing Istio
5. Sidecar Injection
6. Traffic Management
7. Security with mTLS
8. Authorization Policies
9. Namespace Isolation Example
10. Observability (Kiali, Grafana, Prometheus)
11. Debugging Istio
12. Production Best Practices

---

# 1. What is Istio

Istio is a **service mesh** that manages communication between microservices.

It provides:

- Traffic management
- Security (mTLS)
- Observability
- Policy enforcement

Without modifying application code.

---

# 2. Service Mesh Architecture

A service mesh consists of:

### Data Plane

Envoy proxies running as **sidecars** inside pods.

Responsibilities:

- Routing
- Encryption
- Telemetry

### Control Plane

Managed by **Istiod**.

Responsibilities:

- Configuration distribution
- Certificate management
- Policy enforcement

---

# 3. Core Istio Components

| Component | Purpose |
|-----------|--------|
| Istiod | Control plane |
| Envoy Proxy | Sidecar proxy |
| Kiali | Visualization |
| Grafana | Metrics dashboards |
| Prometheus | Metrics collection |
| Jaeger | Distributed tracing |

---

# 4. Installing Istio

## Download

```bash
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH
```

## Install Default Profile

```bash
istioctl install --set profile=default -y
```

Verify:

```bash
kubectl get pods -n istio-system
```

---

# 5. Sidecar Injection

Enable automatic sidecar injection:

```bash
kubectl label namespace default istio-injection=enabled
```

Deploy any application and you will see:

```
app-container
istio-proxy
```

---

# 6. Traffic Management

Istio allows advanced routing using:

### VirtualService

Controls routing rules.

### DestinationRule

Defines subsets and policies.

Example:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
```

---

# 7. Security with mTLS

Mutual TLS ensures:

- Service identity verification
- Encrypted communication
- Zero trust networking

Enable strict mTLS:

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
spec:
  mtls:
    mode: STRICT
```

---

# 8. Authorization Policies

AuthorizationPolicy controls **who can talk to whom**.

Example:

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-frontend
spec:
  rules:
  - from:
    - source:
        namespaces: ["frontend"]
```

---

# 9. Namespace Isolation Example

Goal:

- Frontend services communicate only with frontend
- Backend services communicate only with backend

Steps:

1. Enable sidecar injection
2. Enable strict mTLS
3. Apply deny-all policy
4. Allow only same namespace communication

This enforces **service mesh level segmentation**.

---

# 10. Observability

Install monitoring tools:

```bash
kubectl apply -f samples/addons/prometheus.yaml
kubectl apply -f samples/addons/grafana.yaml
kubectl apply -f samples/addons/kiali.yaml
```

Access Kiali:

```bash
kubectl port-forward svc/kiali -n istio-system 20001:20001
```

---

# 11. Debugging Istio

Useful commands:

Check proxy configuration:

```bash
istioctl proxy-status
```

Describe proxy:

```bash
istioctl proxy-config cluster POD_NAME
```

Check logs:

```bash
kubectl logs POD_NAME -c istio-proxy
```

---

# 12. Production Best Practices

- Enable strict mTLS
- Use AuthorizationPolicies
- Monitor using Kiali and Prometheus
- Use resource limits for sidecars
- Use canary deployments
- Implement circuit breaking

---

# Conclusion

Istio enables:

- Secure microservice communication
- Advanced traffic control
- Observability and monitoring

With proper configuration, it becomes a **powerful platform for operating microservices at scale**.
