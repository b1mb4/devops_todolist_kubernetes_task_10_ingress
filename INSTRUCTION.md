# Validation Instructions

This document provides step-by-step instructions for deploying and validating the Kubernetes Ingress configuration for the Django ToDo application.

---

## 1. Create the Kubernetes Cluster

Use `kind` to spin up a multi-node cluster configured with extra port mappings for HTTP (port 80) and HTTPS (port 443) on the control plane node:

```bash
kind create cluster --config cluster.yml
```

Verify that the cluster nodes are running:

```bash
kubectl get nodes
```

---

## 2. Deploy Infrastructure & Ingress Controller

Run `bootstrap.sh` to deploy the MySQL database, ToDo application resources, and install the NGINX Ingress Controller:

```bash
chmod +x bootstrap.sh
./bootstrap.sh
```

Wait for the NGINX Ingress Controller pod to be ready in the `ingress-nginx` namespace:

```bash
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

Verify that all pods in the `mysql` and `todoapp` namespaces are running:

```bash
kubectl get pods -n mysql
kubectl get pods -n todoapp
```

---

## 3. Apply and Verify Ingress Resource

Apply the Ingress manifest located in `.infrastructure/ingress/ingress.yml`:

```bash
kubectl apply -f .infrastructure/ingress/ingress.yml
```

Verify that the Ingress resource was created successfully:

```bash
kubectl get ingress -n todoapp
```

Inspect the Ingress details:

```bash
kubectl describe ingress todoapp-ingress -n todoapp
```

Expected configuration:
- **Namespace**: `todoapp`
- **Annotations**:
  - `nginx.ingress.kubernetes.io/use-regex: "true"`
  - `nginx.ingress.kubernetes.io/rewrite-target: /$1`
- **Rules**:
  - Host: `*`
  - Path: `/(.*)` (ImplementationSpecific) -> Backend: `todoapp-service:80`

---

## 4. Validate Application Access via Ingress

Since the Kind cluster maps port `80` on the host to the ingress controller, traffic to `http://localhost` will be routed through the ingress to `todoapp-service`.

### 4.1 CLI Validation

Test the root endpoint:

```bash
curl -I http://localhost/
```

Expected output:
HTTP response status code `200 OK` (or redirect `302 Found`).

Test the API endpoint:

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost/api/
```

Expected output:
`200`

### 4.2 Browser & Console Validation

1. Open `http://localhost` in your web browser.
2. Verify that the Django ToDo list landing page loads correctly.
3. Open the browser Developer Tools (**F12** or **Inspect**) and navigate to the **Console** and **Network** tabs.
4. Verify that:
   - All static assets (Skeleton CSS, jQuery, images, fonts) load successfully (`200 OK`).
   - There are **no 404 Not Found** errors for any static files or API requests.
5. Create, complete, or delete a ToDo item to confirm API and database interactions are working.

---

## 5. Cleanup (Optional)

To delete the Kind cluster after validation:

```bash
kind delete cluster
```
