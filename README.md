# Lab 04: Chaos and fault injection in Istio

This lab focuses on failure injection and resilience testing. It is designed to help you understand how Istio can deliberately inject errors and how to observe the system response.

## Goal

Learn how to:
- deploy the Bookinfo application
- enable automatic sidecar injection
- apply a VirtualService fault injection rule
- observe 500 errors and delay behavior
- understand how traffic routing interacts with fault injection

## Lab 1: Bookinfo fault injection exercise

### Step 1: Label the namespace and deploy Bookinfo

```bash
kubectl label namespace default istio-injection=enabled
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-test-delay.yaml
```

What this command does:
- enables sidecar injection in the default namespace
- deploys the Bookinfo microservice demo application
- applies a traffic rule that introduces a delay for the `ratings` service

Expected result:
- Bookinfo services and pods are created
- the mesh has traffic rules controlling the `ratings` service

Troubleshooting:
- If services do not appear, confirm the Bookinfo manifests exist in your Istio installation
- Ensure the correct namespace is set when applying the YAML

### Step 2: Check the running services

```bash
kubectl get all
kubectl get gw bookinfo-gateway
kubectl get virtualservices
kubectl get virtualservices ratings -o yaml
```

What this command does:
- shows all Bookinfo resources and their status
- confirms the ingress Gateway exists
- reveals the VirtualService configuration for `ratings`
- displays the injected fault configuration

Key part of the YAML to notice:

```yaml
spec:
  hosts:
  - ratings
  http:
  - fault:
      abort:
        httpStatus: 500
        percentage:
          value: 50
    route:
    - destination:
        host: ratings
        subset: v1
```

Interpretation:
- 50% of requests to the `ratings` service are aborted with `HTTP 500`
- the rest continue to the destination

Expected output summary:
- `bookinfo-gateway` exists
- `ratings` VirtualService is present
- `fault.abort.httpStatus: 500` is configured

## Lab 2: Fault injection with a hello application

This exercise creates a simple app and intentionally aborts 50% of traffic.

### Step 1: Deploy the app

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
spec:
  selector:
    matchLabels:
      app: hello
      version: v1
  replicas: 1
  template:
    metadata:
      labels:
        app: hello
        version: v1
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
---
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
---
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: hello
spec:
  hosts:
  - foo.example.com
  gateways:
  - hello-gateway
  http:
  - fault:
      abort:
        percentage:
          value: 50
        httpStatus: 500
    route:
    - destination:
        host: hello
        port:
          number: 8080
```

Apply it:

```bash
kubectl create -f application.yaml
```

What this command does:
- deploys a simple app named `hello`
- exposes it through a Gateway and VirtualService
- injects a fault rule that aborts 50% of requests with HTTP 500

### Step 2: Add host entry and test

```bash
kubectl get svc -n istio-system
vim /etc/hosts
10.109.230.115  foo.example.com
for i in {1..10}; do curl -I http://foo.example.com; done
```

Expected output pattern:

```text
HTTP/1.1 500 Internal Server Error
...
HTTP/1.1 200 OK
...
```

What this command does:
- sends multiple requests through the ingress gateway
- demonstrates that approximately half of the requests fail intentionally
- proves the fault injection is working

Expected result summary:
- some requests return `500 Internal Server Error`
- some return `200 OK`

## Why this matters

Fault injection is used to test resilience. It helps answer questions like:
- What happens when a service fails?
- Does the client recover correctly?
- Are retries and timeouts configured properly?
- Is the system resilient under partial outages?

## Troubleshooting tips

- If all responses are 200 OK, the fault rule may not be applied locally or the route may not match the host
- If you see no response, verify the ingress gateway service is running
- If the app is not available, check pod readiness and container logs
- If the Gateway is not selecting ingress, verify `istio: ingressgateway` matches the deployment labels

## Summary

This lab demonstrates a core service mesh behavior: traffic can be intentionally manipulated to simulate failure conditions. This is an essential practice for understanding debugging, resilience, and resiliency testing in a distributed system.

---

Sample command output from the lab:

```bash
root@controlplane:~$ for i in {1..10}; do curl -I http://foo.example.com; done
HTTP/1.1 500 Internal Server Error
...
HTTP/1.1 200 OK
...
```

The variation between `500` and `200` confirms that fault injection is active.

"""

}]}{