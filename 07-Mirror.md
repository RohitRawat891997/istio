# Mirroring Traffic with Istio

This task demonstrates Istio traffic mirroring, also known as shadowing. Instead of sending user requests to a new version of a service, you duplicate a portion of the live traffic and send it to the new version for testing, while the original request still goes to the stable version.

This is useful when you want to validate a new release under real production traffic without affecting user-facing behavior.

In this lab, we will:

1. Deploy two versions of the `httpbin` service: `v1` and `v2`
2. Force all traffic to `v1`
3. Mirror traffic from `v1` to `v2`
4. Verify the logs for both versions
5. Clean up the environment

## Prerequisites

- A Kubernetes cluster
- `kubectl` configured to the cluster
- Istio installed and working

If you have not installed Istio yet, follow the Istio installation guide for your platform.

## 1) Deploy the application

Create the two `httpbin` deployments and the `httpbin` service.

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin-v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpbin
      version: v1
  template:
    metadata:
      labels:
        app: httpbin
        version: v1
    spec:
      containers:
      - name: httpbin
        image: docker.io/kennethreitz/httpbin
        imagePullPolicy: IfNotPresent
        command: ["gunicorn", "--access-logfile", "-", "-b", "[::]:80", "httpbin:app"]
        ports:
        - containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpbin
      version: v2
  template:
    metadata:
      labels:
        app: httpbin
        version: v2
    spec:
      containers:
      - name: httpbin
        image: docker.io/kennethreitz/httpbin
        imagePullPolicy: IfNotPresent
        command: ["gunicorn", "--access-logfile", "-", "-b", "[::]:80", "httpbin:app"]
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: httpbin
  labels:
    app: httpbin
spec:
  selector:
    app: httpbin
  ports:
  - name: http
    port: 8000
    targetPort: 80
EOF
```

Next, deploy a `curl` workload to generate traffic:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: curl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: curl
  template:
    metadata:
      labels:
        app: curl
    spec:
      containers:
      - name: curl
        image: curlimages/curl
        command: ["/bin/sleep", "3650d"]
        imagePullPolicy: IfNotPresent
EOF
```

## 2) Set a default route to v1

By default, Kubernetes does not know about application versions. If both pods are selected by the `httpbin` service, traffic is distributed across both versions. To control this, we define a `VirtualService` and a `DestinationRule`.

```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  - httpbin
  http:
  - route:
    - destination:
        host: httpbin
        subset: v1
      weight: 100
---
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: httpbin
spec:
  host: httpbin
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
EOF
```

This tells Istio:

- all traffic for `httpbin` should go to the backend subset `v1`
- `v1` is selected by the label `version: v1`
- `v2` is defined for later mirroring

## 3) Send a request and validate v1 receives it

Now send a request through the service:

```bash
kubectl exec deploy/curl -c curl -- curl -sS http://httpbin:8000/headers
```

Example response:

```json
{
  "headers": {
    "Accept": "*/*",
    "Content-Length": "0",
    "Host": "httpbin:8000",
    "User-Agent": "curl/7.35.0",
    "X-B3-Parentspanid": "57784f8bff90ae0b",
    "X-B3-Sampled": "1",
    "X-B3-Spanid": "3289ae7257c3f159",
    "X-B3-Traceid": "b56eebd279a76f0b57784f8bff90ae0b"
  }
}
```

Check the access logs on both versions:

```bash
kubectl logs deploy/httpbin-v1 -c httpbin
kubectl logs deploy/httpbin-v2 -c httpbin
```

Expected result:

- `httpbin-v1` shows the request log
- `httpbin-v2` shows no traffic yet

Example log from `v1`:

```text
127.0.0.1 - - [07/Mar/2018:19:02:43 +0000] "GET /headers HTTP/1.1" 200 321 "-" "curl/7.35.0"
```

`v2` should remain empty:

```text
<none>
```

## 4) Mirror production traffic to v2

Now add the mirror rule. The original request continues to `v1`, but a copy is sent to `v2`.

```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  - httpbin
  http:
  - route:
    - destination:
        host: httpbin
        subset: v1
      weight: 100
    mirror:
      host: httpbin
      subset: v2
    mirrorPercentage:
      value: 100.0
EOF
```

Important clarification:

- `route` controls the actual user-facing traffic
- `mirror` creates a shadow copy of the request
- the result returned to the client still comes from `v1`
- `v2` receives a duplicated request for observation, testing, or telemetry

## 5) Send traffic and validate mirroring

Send another request:

```bash
kubectl exec deploy/curl -c curl -- curl -sS http://httpbin:8000/headers
```

Now inspect logs:

```bash
kubectl logs deploy/httpbin-v1 -c httpbin
kubectl logs deploy/httpbin-v2 -c httpbin
```

Expected behavior:

- `httpbin-v1` logs the original requests
- `httpbin-v2` logs mirrored requests as well

Example output:

```text
$ kubectl logs deploy/httpbin-v1 -c httpbin
127.0.0.1 - - [07/Mar/2018:19:02:43 +0000] "GET /headers HTTP/1.1" 200 321 "-" "curl/7.35.0"
127.0.0.1 - - [07/Mar/2018:19:26:44 +0000] "GET /headers HTTP/1.1" 200 321 "-" "curl/7.35.0"

$ kubectl logs deploy/httpbin-v2 -c httpbin
127.0.0.1 - - [07/Mar/2018:19:26:44 +0000] "GET /headers HTTP/1.1" 200 361 "-" "curl/7.35.0"
127.0.0.1 - - [07/Mar/2018:19:26:44 +0000] "GET /headers HTTP/1.1" 200 361 "-" "curl/7.35.0"
```

Notice the behavior:

- the user still receives the response from `v1`
- the mirrored traffic is only copied to `v2`
- the mirrored requests can help you verify the new version under real traffic conditions without impacting the production experience

## Why mirroring is valuable

Traffic mirroring is ideal for:

- canary validation with real traffic
- testing new versions against production-like requests
- collecting logs, metrics, or traces from a candidate deployment
- reducing release-risk without user-facing impact

For production, you usually do not mirror 100% of requests. Instead, you may use a lower percentage such as 5%, 10%, or 20%, depending on your risk appetite.

## Clean up

Delete the VirtualService and DestinationRule:

```bash
kubectl delete virtualservice httpbin
kubectl delete destinationrule httpbin
```

Delete the workloads and service:

```bash
kubectl delete deploy httpbin-v1 httpbin-v2 curl
kubectl delete svc httpbin
```

## Summary

This lab showed how Istio can:

- route all traffic to a stable version
- duplicate or mirror traffic to a test version
- let you validate a release using real traffic without risking the user experience

Traffic mirroring is one of the most practical techniques for safe progressive delivery in service mesh environments.
