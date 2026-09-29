# Istio Authorization Policy Hands-On Lab

This guide demonstrates how to deploy a containerized application behind an **Istio Ingress Gateway** and secure it using **Istio Authorization Policies (RBAC)**.

---

## 🏗️ Architecture Overview

1. **Deployment**: A single replica `hello` application running Apache (`httpd`) on port `8080`.
2. **Service**: An internal ClusterIP service routing traffic to port `8080`.
3. **Gateway**: Exposes the application externally on port `80` for the host `foo.example.com`.
4. **VirtualService**: Connects the `hello-gateway` traffic to the internal `hello` service.
5. **AuthorizationPolicy**: Restricts access based on source principals, namespaces, and target ports.

---

## 🛠️ Step 1: Deploy the Core Application Stack

Create a single file named `application.yaml` containing the deployment, service, gateway, and virtual service configurations.

### `application.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello
      version: v1
  template:
    metadata:
      labels:
        app: hello
        version: v1
      annotations:
        sidecar.istio.io/inject: "true" # Enables automatic Istio sidecar injection
    spec:
      containers:
      - name: hello
        image: docker.io/rohitrawat891997/httpd:latest
        command: ['sh', '-c', 'echo "Hello World" > /usr/local/apache2/htdocs/index.html && httpd-foreground']
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: hello
  namespace: default
  labels:
    app: hello
spec:
  ports:
  - name: http-8080
    port: 8080
    targetPort: 8080
  selector:
    app: hello
---
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: hello-gateway
  namespace: default
spec:
  selector:
    istio: ingressgateway # Targets the standard Istio Ingress Controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "foo.example.com"
---
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: hello-vs
  namespace: default
spec:
  hosts:
  - "foo.example.com"
  gateways:
  - hello-gateway
  http:
  - route:
    - destination:
        host: hello
        port:
          number: 8080
```

### Apply Configurations & Verify Routing
1. **Apply the manifest:**
   ```bash
   kubectl create -f application.yaml
   ```

2. **Verify Pods, Services, and Gateways are active:**
   ```bash
   kubectl get all
   kubectl get gw,vs
   ```

3. **Map the Ingress Gateway IP to your local hosts file:**
   Find your `istio-ingressgateway` IP via `kubectl get svc -n istio-system` and add it to `/etc/hosts`:
   ```text
   # Example entry inside /etc/hosts
   10.109.247.149   foo.example.com
   ```

4. **Test baseline connectivity (Should return 200 OK):**
   ```bash
   curl http://example.com
   # Output: Hello World
   ```

---

## 🔒 Step 2: Introduce Istio Authorization Policy (With a Bug)

Next, we enforce an explicit `ALLOW` rule. However, intentionally defining the wrong target port simulates a real-world debugging scenario.

### `auth.yaml` (Misconfigured)
```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: httpbin
  namespace: default
spec:
  action: ALLOW
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account"]
        - source:
            namespaces: ["default"]
      to:
        - operation:
            methods: ["GET"]
            ports: ["8088"] # ❌ BUG: The application actually listens on 8080!
```

### Apply and Test the Policy
1. **Apply the security policy:**
   ```bash
   kubectl create -f auth.yaml
   ```

2. **Verify the traffic behavior:**
   Because an explicit `ALLOW` policy is active but evaluates to a non-matching port (`8088`), Istio falls back to a default **deny-all** stance for unhandled traffic patterns:
   ```bash
   curl http://example.com
   # Output: RBAC: access denied
   ```

---

## 🛠️ Step 3: Hot-Fixing the Policy (The Pro Fix)

To restore legitimate traffic access, we patch the target port parameter from `8088` to match the target runtime port `8080`.

1. **Edit the live policy object directly:**
   ```bash
   kubectl edit ap httpbin
   ```

2. **Update the ports section under the operations block:**
   ```yaml
   # Change this:
   ports: ["8088"]
   
   # To this:
   ports: ["8080"]
   ```
   Save and exit (`:wq!`).

3. **Final Validation:**
   Once the updated config syncs down to Envoy proxies across the mesh, verify access is securely restored to authorized principals:
   ```bash
   curl http://example.com
   # Output: Hello World
   ```
