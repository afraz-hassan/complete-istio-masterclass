
# Namespace Isolation Through Istio

## References
- https://istio.io/latest/docs/ops/diagnostic-tools/istioctl/
- https://istio.io/latest/docs/ops/configuration/security/security-policy-examples/#namespace-isolation
- https://funkypenguin.co.nz/blog/istio-namespace-isolation-tricks/

## Overview

Goal:

- Frontend pods can communicate **only with frontend pods**
- Backend pods can communicate **only with backend pods**
- **No communication allowed across namespaces**
- But the traffic from the Ingress should be allowed inside the namespace.

Isolation is enforced using **Istio AuthorizationPolicy** at the **service mesh layer**, not only Kubernetes NetworkPolicies.

This provides **identity‑aware L4/L7 traffic control**.

---

## Prerequisites

- Kubernetes Cluster (v1.24+ recommended)
- kubectl configured
- Cluster admin permissions

---

# Step 1: Install Istio

## Download Istio

```bash
curl -L https://istio.io/downloadIstio | sh -
cd istio-*

export PATH=$PWD/bin:$PATH
```

Alternative if already installed:

```bash
export PATH=$PATH:/path/to/installed/istio
```

## Install Istio

```bash
istioctl install --set profile=default -y
```

## Verify Installation

```bash
kubectl get pods -n istio-system
```

---

# Step 2: Check Namespaces

```bash
kubectl get namespace frontend
kubectl get namespace backend
```

If they do not exist:

```bash
kubectl create namespace frontend
kubectl create namespace backend
```

---

# Step 3: Enable Istio Sidecar Injection

Sidecar injection adds the **Envoy proxy** to every pod.

```bash
kubectl label namespace frontend istio-injection=enabled
kubectl label namespace backend istio-injection=enabled
```

Verify:

```bash
kubectl get namespace frontend --show-labels
kubectl get namespace backend --show-labels
```

---

# Step 4: Install Monitoring Addons (Optional)

Navigate to:

```
istio-*/samples/addons/
```

Available integrations:

- Grafana
- Prometheus
- Kiali

Example installing **Kiali**:

```bash
kubectl apply -f samples/addons/kiali.yaml
```

Port forward to access:

```bash
kubectl port-forward svc/kiali -n istio-system 20001:20001
```

Access in browser:

```
http://localhost:20001
```

---

# Step 5: Enforce Strict mTLS

This ensures:

- Only **Istio authenticated identities** communicate
- Non‑mesh traffic is denied

### strict-mtls-frontend.yaml

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: strict-mtls
  namespace: frontend
spec:
  mtls:
    mode: STRICT
```

### strict-mtls-backend.yaml

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: strict-mtls
  namespace: backend
spec:
  mtls:
    mode: STRICT
```

Apply:

```bash
kubectl apply -f strict-mtls-frontend.yaml
kubectl apply -f strict-mtls-backend.yaml
```

---

# Step 6: Default Deny All Traffic

This establishes a **Zero Trust baseline**.

### frontend-deny-all.yaml

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: frontend
spec: {}
```

### backend-deny-all.yaml

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: backend
spec: {}
```

Apply:

```bash
kubectl apply -f frontend-deny-all.yaml
kubectl apply -f backend-deny-all.yaml
```

At this point:

❌ No pod can talk to any other pod

---

# Step 7: Allow Intra‑Namespace Communication

## Allow Frontend → Frontend

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-frontend-self
  namespace: frontend
spec:
  rules:
  - from:
    - source:
        namespaces: ["frontend"]
  - from:
    - source:
        namespaces: ["ingress-nginx"]
```

## Allow Backend → Backend

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-backend-self
  namespace: backend
spec:
  rules:
  - from:
    - source:
        namespaces: ["backend"]
  - from:
    - source:
        namespaces: ["ingress-nginx"]
```

Apply:

```bash
kubectl apply -f frontend-allow-self.yaml
kubectl apply -f backend-allow-self.yaml
```

---

# Step 8: Verify Isolation

## Frontend → Frontend (Should Work)

```bash
kubectl exec -n frontend -it $(kubectl get pod -n frontend -l app=frontend -o jsonpath='{.items[0].metadata.name}') -- curl http://frontend-service
```

Expected:

```
200 OK
```

---

## Frontend → Backend (Should Fail)

```bash
kubectl exec -n frontend -it $(kubectl get pod -n frontend -l app=frontend -o jsonpath='{.items[0].metadata.name}') -- curl http://backend-service.backend
```

Expected:

```
RBAC: access denied
```

---

## Backend → Frontend (Should Fail)

```bash
kubectl exec -n backend -it $(kubectl get pod -n backend -l app=backend -o jsonpath='{.items[0].metadata.name}') -- curl http://frontend-service.frontend
```

Expected:

```
Access denied
```

---

## Backend → Backend (Should Work)

```bash
kubectl exec -n backend -it $(kubectl get pod -n backend -l app=backend -o jsonpath='{.items[0].metadata.name}') -- curl http://backend-service
```

Expected:

```
Success
```
