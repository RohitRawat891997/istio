# Mutual TLS (mTLS) in Istio

This guide demonstrates implementing mTLS (Mutual TLS) for secure service-to-service communication in Kubernetes using Istio. It covers two primary scenarios: enforcing mTLS between services using PeerAuthentication policies and configuring TLS at the ingress gateway.

---

## Table of Contents

1. [Lab 1: mTLS Service-to-Service Communication](#lab-1-mtls-service-to-service-communication-peer-authentication)
2. [Lab 2: TLS-Based Authentication at Ingress Gateway](#lab-2-tls-based-authentication-at-ingress-gateway)

---

## Lab 1: mTLS Service-to-Service Communication (Peer Authentication)

### Objective
Enable STRICT mTLS mode between services to enforce encrypted communication at the network layer. When configured, services reject unencrypted traffic.

### Prerequisites

- Two services running in the cluster:
  - `hello` service (port 8080)
  - `mypod` service (port 8080)
- Both services injected with Istio sidecar proxies

### Step 1: Verify Current Environment

Check the running services and pods:

```bash
$ kubectl get all

NAME                         READY   STATUS    RESTARTS   AGE
pod/hello-5b8c95c97c-b2tmf   2/2     Running   0          48m
pod/mypod-bc6b495-7njhj      0/1     ContainerCreating   0          6s

NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/hello        ClusterIP   10.106.77.77     <none>        8080/TCP   48m
service/kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP    9d
service/mypod        ClusterIP   10.105.188.225   <none>        8080/TCP   6s

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hello   1/1     1            1           48m
deployment.apps/mypod   0/1     1            0           6s
```

### Step 2: Test Communication Without mTLS

Access the `mypod` pod and verify connectivity to the `hello` service over plaintext HTTP:

```bash
$ kubectl exec -it pod/mypod-bc6b495-7njhj -- bash
root@mypod-bc6b495-7njhj:/# curl http://hello:8080
Hello World
root@mypod-bc6b495-7njhj:/# exit
```

**Result:** Communication succeeds without encryption.

### Step 3: Create PeerAuthentication Policy

Create a `peer.yaml` file to enforce STRICT mTLS mode:

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
spec:
  mtls:
    mode: STRICT
```

Apply the policy:

```bash
$ kubectl create -f peer.yaml
peerauthentication.security.istio.io/default created
```

Verify the policy is created:

```bash
$ kubectl get peerauthentications
NAME      MODE     AGE
default   STRICT   33s
```

### Step 4: Test Communication With STRICT mTLS

Attempt to access the `hello` service again. Since `mypod` is not sending encrypted traffic by default, the request will fail:

```bash
$ kubectl exec -it pod/mypod-bc6b495-7njhj -- bash
root@mypod-bc6b495-7njhj:/# curl http://hello:8080
curl: (56) Recv failure: Connection reset by peer
root@mypod-bc6b495-7njhj:/# exit
command terminated with exit code 56
```

**Result:** The connection is rejected because unencrypted traffic violates the STRICT mTLS policy.

### Step 5: Clean Up

Delete the PeerAuthentication policy:

```bash
$ kubectl delete -f peer.yaml
peerauthentication.security.istio.io "default" deleted from default namespace
```

Verify communication is restored:

```bash
$ kubectl exec -it pod/mypod-bc6b495-7njhj -- bash
root@mypod-bc6b495-7njhj:/# curl http://hello:8080
Hello World
root@mypod-bc6b495-7njhj:/# exit
```

### Key Takeaways

- **STRICT mode** requires all traffic to be encrypted with mTLS.
- Istio sidecars automatically handle certificate issuance and rotation.
- Without explicit workload policies (like `DestinationRule`), Istio sidecars will attempt to upgrade plaintext connections to mTLS if the receiving service supports it.
- When STRICT mode is enforced without client-side configuration, unencrypted traffic is blocked.

---

## Lab 2: TLS-Based Authentication at Ingress Gateway

### Objective
Configure Istio Ingress Gateway to handle HTTPS traffic using TLS certificates. This enables secure communication between external clients and the cluster.

### Prerequisites

- Istio Ingress Gateway deployed in `istio-system` namespace
- At least one backend service (e.g., `hello`) running

---

### Step 1: Generate Self-Signed Certificates

Create a self-signed certificate and private key pair for testing:

```bash
$ openssl req -x509 -newkey rsa:2048 -nodes \
  -keyout client.key \
  -out client.crt \
  -days 365
```

This command generates:
- `client.key` - Private key
- `client.crt` - Self-signed certificate (valid for 365 days)

---

### Step 2: Create a TLS Secret

Create a Kubernetes TLS secret in the `istio-system` namespace:

```bash
$ kubectl -n istio-system create secret tls istio-ingressgateway-customer-certs \
  --key customer.key \
  --cert customer.crt
```

Verify the secret is created:

```bash
$ kubectl get secrets -n istio-system

NAME                                  TYPE                  DATA   AGE
customer                              kubernetes.io/tls    2      19m
istio-ca-secret                       istio.io/ca-root      5      27m
```

---

### Step 3: Mount Certificates into Ingress Gateway Pod

Create a JSON patch file (`gateway-patch.json`) to mount the TLS secret as a volume in the Ingress Gateway pod:

```json
[
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/volumeMounts/0",
    "value": {
      "mountPath": "/etc/istio/customer-certs",
      "name": "customer-certs",
      "readOnly": true
    }
  },
  {
    "op": "add",
    "path": "/spec/template/spec/volumes/0",
    "value": {
      "name": "customer-certs",
      "secret": {
        "secretName": "istio-ingressgateway-customer-certs",
        "optional": true
      }
    }
  }
]
```

Apply the patch to the `istio-ingressgateway` deployment:

```bash
$ kubectl -n istio-system patch --type=json deploy istio-ingressgateway \
  -p "$(cat gateway-patch.json)"
```

---

### Step 4: Create Gateway and VirtualService Manifests

#### Option 1: Direct Certificate Path Reference

Create `gateway.yaml` with explicit certificate and key paths:

```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: hello-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - "foo.example.com"
    tls:
      mode: SIMPLE
      serverCertificate: /etc/istio/customer-certs/tls.crt
      privateKey: /etc/istio/customer-certs/tls.key
```

#### Option 2: Secret-Based Reference (Recommended)

Alternatively, use the secret credential name:

```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: hello-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - "foo.example.com"
    tls:
      mode: SIMPLE
      credentialName: customer
```

**Recommendation:** Option 2 is preferred for better secret management and automatic certificate rotation.

#### VirtualService Manifest

Create a `virtualservice.yaml` to route traffic to the backend service:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: hello-vs
spec:
  hosts:
  - "foo.example.com"
  gateways:
  - hello-gateway
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: hello
        port:
          number: 8080
```

---

### Step 5: Deploy and Verify

Apply the Gateway and VirtualService:

```bash
$ kubectl create -f gateway.yaml
$ kubectl create -f virtualservice.yaml
```

Verify resources are created:

```bash
$ kubectl get gw,vs

NAME                                        AGE
gateway.networking.istio.io/hello-gateway   8m3s

NAME                                          GATEWAYS            HOSTS                 AGE
virtualservice.networking.istio.io/hello-vs   ["hello-gateway"]   ["foo.example.com"]   24m
```

Inspect the Gateway configuration:

```bash
$ kubectl get gw hello-gateway -o yaml

apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: hello-gateway
  namespace: default
  creationTimestamp: "2026-09-28T22:15:35Z"
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - foo.example.com
    port:
      name: https
      number: 443
      protocol: HTTPS
    tls:
      mode: SIMPLE
      privateKey: /etc/istio/customer-certs/tls.key
      serverCertificate: /etc/istio/customer-certs/tls.crt
```

---

### Step 6: Add DNS Entry

Update `/etc/hosts` to resolve `foo.example.com` to the Ingress Gateway's external IP:

```bash
$ cat /etc/hosts

10.109.148.215   foo.example.com
```

---

### Step 7: Test HTTPS Access

Use `curl` with the `-k` flag to skip certificate validation (since it's self-signed):

```bash
$ curl -k https://foo.example.com
Hello World
```

**Result:** Secure HTTPS communication is successful!

#### Additional Testing

To verify the certificate details:

```bash
$ curl -kv https://foo.example.com

* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
...
* Server certificate:
*  subject: C=XX,ST=StateName,L=City,O=Organization,CN=foo.example.com
```

---

## Troubleshooting Guide

| Issue | Cause | Solution |
|-------|-------|----------|
| `curl: (56) Recv failure: Connection reset by peer` | STRICT mTLS enforced but client not using mTLS | Ensure client sidecar is injected or disable STRICT mode |
| `curl: (60) SSL: certificate problem` | Certificate validation failed | Use `-k` flag for self-signed certs or add to trusted CA |
| Gateway not receiving traffic | Certificate path incorrect in pod | Verify secret mounting via `kubectl exec` and check paths |
| `Connection refused` on port 443 | Ingress Gateway pod not ready | Check Ingress Gateway pod status: `kubectl get pods -n istio-system` |

---

## Best Practices

1. **Use Secret-Based Credentials:** Prefer `credentialName` over direct certificate paths for better secret management.
2. **Automate Certificate Rotation:** Use cert-manager with Istio for automatic certificate renewal.
3. **Enforce mTLS Gradually:** Start with PERMISSIVE mode to monitor traffic before switching to STRICT.
4. **Namespace-Level Policies:** Apply PeerAuthentication at namespace scope for targeted enforcement.
5. **Monitor Certificate Expiry:** Set up alerts for certificate expiration to prevent service disruption.

---

## References

- [Istio Security - Mutual TLS](https://istio.io/latest/docs/concepts/security/)
- [Istio PeerAuthentication Documentation](https://istio.io/latest/docs/reference/config/security/peer_authentication/)
- [Istio Gateway Configuration](https://istio.io/latest/docs/reference/config/networking/gateway/)
- [Kubernetes TLS Secrets](https://kubernetes.io/docs/concepts/configuration/secret/#tls-secrets)

---

**Last Updated:** September 28, 2026
