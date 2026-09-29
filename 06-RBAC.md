
 Autherization Policy 
=====================================================================


root@controlplane:~$ 
root@controlplane:~$ cat    application.yaml 
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
  name: hello-vs
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

root@controlplane:~$ 
root@controlplane:~$ 
root@controlplane:~$ 

root@controlplane:~$ 
root@controlplane:~$ 
root@controlplane:~$ kubectl  create   -f    application.yaml 
deployment.apps/hello created
service/hello created
gateway.networking.istio.io/hello-gateway created
virtualservice.networking.istio.io/hello-vs created
root@controlplane:~$ 
root@controlplane:~$ kubectl  get   all
NAME                        READY   STATUS    RESTARTS   AGE
pod/hello-78c6844f6-sctfl   2/2     Running   0          25s

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/hello        ClusterIP   10.101.65.131   <none>        8080/TCP   25s
service/kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP    9d

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hello   1/1     1            1           26s

NAME                              DESIRED   CURRENT   READY   AGE
replicaset.apps/hello-78c6844f6   1         1         1       25s
root@controlplane:~$ 
root@controlplane:~$ 
root@controlplane:~$ kubectl  get   gw,vs
NAME                                        AGE
gateway.networking.istio.io/hello-gateway   34s

NAME                                          GATEWAYS            HOSTS                 AGE
virtualservice.networking.istio.io/hello-vs   ["hello-gateway"]   ["foo.example.com"]   34s
root@controlplane:~$ 
root@controlplane:~$ 
root@controlplane:~$ kubectl  get   gw  hello-gateway   -o   yaml 
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  creationTimestamp: "2026-09-29T14:07:26Z"
  generation: 1
  name: hello-gateway
  namespace: default
  resourceVersion: "9176"
  uid: cf163c7f-7b7f-4402-a92c-8d2958e31cb9
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - foo.example.com
    port:
      name: http
      number: 80
      protocol: HTTP
root@controlplane:~$ 
root@controlplane:~$ 
root@controlplane:~$ kubectl  get   vs  hello-vs -o   yaml 
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  creationTimestamp: "2026-09-29T14:07:26Z"
  generation: 1
  name: hello-vs
  namespace: default
  resourceVersion: "9177"
  uid: ea877d57-793d-4051-8d59-79cf3548c94e
spec:
  gateways:
  - hello-gateway
  hosts:
  - foo.example.com
  http:
  - route:
    - destination:
        host: hello
        port:
          number: 8080
root@controlplane:~$ 
root@controlplane:~$ 
root@controlplane:~$ 

root@controlplane:~$ 
root@controlplane:~$ kubectl  get   svc  -n   istio-system 
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                      AGE
istio-egressgateway    ClusterIP   10.111.40.173    <none>        80/TCP,443/TCP                                                               15m
istio-ingressgateway   NodePort    10.109.247.149   <none>        15021:31888/TCP,80:31793/TCP,443:31021/TCP,31400:30492/TCP,15443:31272/TCP   15m
istiod                 ClusterIP   10.101.233.136   <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP                                        15m
root@controlplane:~$ 
root@controlplane:~$ 
root@controlplane:~$ cat    /etc/hosts

10.109.247.149   foo.example.com

root@controlplane:~$ 
root@controlplane:~$ 

root@controlplane:~$ 
root@controlplane:~$ curl  http://foo.example.com
Hello World
root@controlplane:~$ curl  http://foo.example.com
Hello World
root@controlplane:~$ curl  http://foo.example.com
Hello World
root@controlplane:~$ 

root@controlplane:~$ 
root@controlplane:~$ 
root@controlplane:~$ cat   auth.yaml 
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: httpbin
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
        ports: ["8088"]
root@controlplane:~$ 
root@controlplane:~$ 
root@controlplane:~$ 
root@controlplane:~$ 
root@controlplane:~$ kubectl  create  -f   auth.yaml 
authorizationpolicy.security.istio.io/httpbin created
root@controlplane:~$ 
root@controlplane:~$ 
root@controlplane:~$ 
root@controlplane:~$ 
root@controlplane:~$ 
root@controlplane:~$ kubectl  get   authorizationpolicies
NAME      ACTION   AGE
httpbin   ALLOW    91s
root@controlplane:~$ 
root@controlplane:~$ 
root@controlplane:~$ kubectl  get   ap   httpbin   -o    yaml 
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  creationTimestamp: "2026-09-29T14:10:37Z"
  generation: 1
  name: httpbin
  namespace: default
  resourceVersion: "9718"
  uid: 7fa8f80e-f760-4d6c-8137-3cad09b458d6
spec:
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account
    - source:
        namespaces:
        - default
    to:
    - operation:
        methods:
        - GET
        ports:
        - "8088"
root@controlplane:~$ 
root@controlplane:~$ 

root@controlplane:~$ 
root@controlplane:~$ curl  http://foo.example.com
RBAC: access deniedroot@controlplane:~$ 
root@controlplane:~$ curl  http://foo.example.com
RBAC: access deniedroot@controlplane:~$ 
root@controlplane:~$ 
root@controlplane:~$ curl  http://foo.example.com
RBAC: access deniedroot@controlplane:~$ 
root@controlplane:~$ 
root@controlplane:~$ kubectl  edit   ap   httpbin 

# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  creationTimestamp: "2026-09-29T14:10:37Z"
  generation: 1
  name: httpbin
  namespace: default
  resourceVersion: "9718"
  uid: 7fa8f80e-f760-4d6c-8137-3cad09b458d6
spec:
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account
    - source:
        namespaces:
        - default
    to:
    - operation:
        methods:
        - GET
        ports:
        - "8080"   # convert from 8088 to 8080

:wq!

root@controlplane:~$ kubectl  edit   ap   httpbin 
authorizationpolicy.security.istio.io/httpbin edited
root@controlplane:~$ 
root@controlplane:~$ 
root@controlplane:~$ curl  http://foo.example.com
Hello World
root@controlplane:~$ curl  http://foo.example.com
Hello World
root@controlplane:~$ curl  http://foo.example.com
Hello World
root@controlplane:~$ 








