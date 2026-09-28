# Istio Gateway + VirtualService Example (Beginner Friendly)

This is a simple example of how traffic moves through Istio.

## Traffic flow

```text
client --> Gateway --> VirtualService --> Service --> Pod
```

In simple words:
- Gateway: the entry point for incoming traffic
- VirtualService: tells Istio where to send the request
- Service: routes inside Kubernetes
- Pod: the actual application running

---

## 1) Create a namespace and enable Istio injection

```bash
kubectl create ns hello
kubectl label ns hello istio-injection=enabled
```

What this does:
- creates a namespace named `hello`
- turns on automatic sidecar injection for applications in that namespace

This helps each app receive an Istio proxy automatically.

---

## 2) Deploy the application

Create a file called `application.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
spec:
  selector:
    matchLabels:
      app: hello
  replicas: 1
  template:
    metadata:
      labels:
        app: hello
      annotations:
        sidecar.istio.io/inject: "true"
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
  labels:
    app: hello
  name: hello
spec:
  ports:
    - name: http-8080
      port: 8080
      targetPort: 8080
  selector:
    app: hello
```

Apply it:

```bash
kubectl create -f application.yaml -n hello
```

This creates:
- a Deployment named `hello`
- a Service named `hello`
- one pod running an HTTP server

The app writes `Hello World` to the webpage and serves it on port `8080`.

---

## 3) Create an Istio Gateway

Create a file called `gateway.yaml`:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: hello-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "foo.example.com"
```

Apply it:

```bash
kubectl create -f gateway.yaml -n hello
```

What this means:
- this Gateway is connected to the Istio ingress gateway
- it listens on port `80`
- requests for `foo.example.com` are allowed in

Think of it as the front door to your application.

---

## 4) Create a VirtualService

Create a file called `virtual-service.yaml`:

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: hello-vs
spec:
  hosts:
  - "foo.example.com"
  gateways:
  - hello-gateway
  http:
  - route:
    - destination:
        host: hello.hello.svc.cluster.local
        port:
          number: 8080
```

Apply it:

```bash
kubectl create -f virtual-service.yaml -n hello
```

What this does:
- if a request comes for `foo.example.com`
- route it to the `hello` service in the `hello` namespace
- service then forwards it to the pod on port `8080`

This is the traffic routing rule.

---

## 5) Check Istio ingress gateway service

```bash
kubectl get svc -n istio-system
```

Example output:

```bash
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                                      AGE
istio-egressgateway    ClusterIP   10.111.86.97    <none>        80/TCP,443/TCP
istio-ingressgateway   NodePort    10.97.132.26    <none>        15021:30650/TCP,80:32091/TCP,443:30984/TCP,31400:32759/TCP,15443:30454/TCP   10m
istiod                 ClusterIP   10.96.235.136   <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP
```

Important point:
- `istio-ingressgateway` is the main entry point for traffic
- it exposes port `80` via NodePort

---

## 6) Add a local DNS entry

Edit `/etc/hosts`:

```bash
vim /etc/hosts
```

Add:

```bash
10.97.132.26   foo.example.com
```

This maps the domain name to the Istio ingress gateway IP.

---

## 7) Test the application

Run:

```bash
curl foo.example.com
```

Expected output:

```bash
Hello World
```

This confirms the full flow works:

```text
client -> ingress gateway -> Gateway -> VirtualService -> Service -> Pod
```

---

## 8) Verify the running resources

Check pods and services:

```bash
kubectl get po,svc -n hello
```

Example output:

```bash
NAME                         READY   STATUS    RESTARTS   AGE
pod/hello-5b8c95c97c-zp2tq   2/2     Running   0          8m26s

NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/hello   ClusterIP   10.108.50.161   <none>        8080/TCP   11m
```

Check gateway and virtual service:

```bash
kubectl get gw,vs -n hello
```

Example output:

```bash
NAME                                        AGE
gateway.networking.istio.io/hello-gateway   8m

NAME                                          GATEWAYS            HOSTS                 AGE
virtualservice.networking.istio.io/hello-vs   ["hello-gateway"]   ["foo.example.com"]   7m29s
```

---

## Easy understanding

Think of the setup like this:

```text
Client
  |
  v
Gateway (hello-gateway)
  |
  v
VirtualService (hello-vs)
  |
  v
Service (hello)
  |
  v
Pod (hello app)
```

- Gateway accepts traffic from outside
- VirtualService decides where to send it
- Service connects to the pod
- Pod returns the webpage

---

## Final summary

This example demonstrates a basic Istio ingress configuration:

- external request comes to the ingress gateway
- Gateway listens for `foo.example.com`
- VirtualService routes it to the internal service
- Service forwards the request to the pod
- pod responds with `Hello World`

This is the simplest form of traffic routing in Istio.

---

## Lab commands together

```bash
kubectl create ns hello
kubectl label ns hello istio-injection=enabled
kubectl create -f application.yaml -n hello
kubectl create -f gateway.yaml -n hello
kubectl create -f virtual-service.yaml -n hello
kubectl get svc -n istio-system
vim /etc/hosts
curl foo.example.com
kubectl get po,svc -n hello
kubectl get gw,vs -n hello
```

If you want, I can also create:
- a more detailed lab guide
- a diagram version with labels
- or a shorter cheat sheet version.
