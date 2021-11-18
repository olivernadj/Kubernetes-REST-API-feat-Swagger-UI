# Kubernetes-REST-API-feat-Swagger-UI
How to unlock Kubernetes REST API and link it to Swagger UI

WARNING!! Do not apply this changes on publicly available cluster unless, 
you hold your keys under the mat.


In case view only is sufficient, I found better guide for you: 
https://jonnylangefeld.com/blog/kubernetes-how-to-view-swagger-ui


Unfortunatelly interacting with the Kubernetes REST API via Swagger UI is not that 
straight forward and I did not find yet any simple as possible guide to do so.


This guide crafted on Ubuntu and show how to:
 - create a kind cluster with ingress and expose 80 and 443 port to local machine.
 - expose Kubernetes REST API, using nginx proxy
 - install Swagger UI and point to proxied api.



## Create a kind cluster

```bash
cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: swaggerman
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
EOF
```

### Install ingress-nginx

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```
Make sure `ingress-nginx-controller` is already running, before you proceed further.


## Expose Kubernetes REST API

The yaml manifets will:
 1.) Create `api-proxy` namespac
 2.) Bind default `ServiceAccount` to `cluster-admin` roles.
 3.) Create a proxy pass nginx config with with actual Bearer token and certificates.
 4.) Deploy and expose nginx as a proxy for kube api.
 5.) Create Ingress with `kube-api.loc` host and CORS enabled. 

By the way add `kube-api.loc` to `/etc/hosts` and make it point to `127.0.0.1`.



```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: api-proxy

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: api-proxy-cluster-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: default
  namespace: api-proxy

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-proxy-config
  namespace: api-proxy
data:
  default.conf: |2-

    server {
        listen              80;
        server_name         localhost;
        location / {
            proxy_pass https://kubernetes.default.svc;
            proxy_set_header Authorization "Bearer $token_placeholder";
            proxy_ssl_trusted_certificate /var/run/secrets/kubernetes.io/serviceaccount/ca.crt;
        }
    }


---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-proxy
  namespace: api-proxy
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api-proxy
  template:
    metadata:
      labels:
        app: api-proxy
    spec:
      initContainers:
      - name: init-token
        image: busybox:1.28
        command: ['sh', '-c', "cp /default.conf /shared-data/default.conf && TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token) && sed -i 's/$token_placeholder/'$TOKEN'/' /shared-data/default.conf"]
        volumeMounts:
          - name: "shared-data"
            mountPath: "/shared-data"
          - name: "nginx-config"
            mountPath: "/default.conf"
            subPath: "default.conf"
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
        volumeMounts:
          - name: "shared-data"
            mountPath: "/etc/nginx/conf.d/default.conf"
            subPath: "default.conf"
      volumes:
        - name: "shared-data"
          emptyDir: {}
        - name: "nginx-config"
          configMap:
            name: "api-proxy-config"

---
kind: Service
apiVersion: v1
metadata:
  name: api-proxy
  namespace: api-proxy
spec:
  selector:
    app: api-proxy
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kube-api-ingress
  namespace: api-proxy
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/enable-cors: "true"
spec:
  rules:
  - host: kube-api.loc
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: 
            name: api-proxy
            port:
              number: 80

```

At this point you should be able to access Kube API on http://kube-api.loc/api



## Install Swagger UI

The yaml manifets will:
 1.) Create `swagger` namespac
 4.) Deploy and expose Swagger UI.
 5.) Create Ingress with `swagger.loc` host. 

By the way add `swagger.loc` to `/etc/hosts` and make it point to `127.0.0.1`.


```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: swagger

---
kind: Service
apiVersion: v1
metadata:
  name: swagger-ui
  namespace: swagger
spec:
  selector:
    app: swagger-ui
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: swagger-ui
  namespace: swagger
  labels:
    app: swagger-ui
spec:
  replicas: 1
  selector:
    matchLabels:
      app: swagger-ui
  template:
    metadata:
      labels:
        app: swagger-ui
    spec:
      containers:
      - name: swagger-ui
        image: swaggerapi/swagger-ui #build new image for adding local swaggerfile
        ports:
        - containerPort: 8080
        env:
        - name: BASE_URL
          value: /
        - name: API_URLS
          value: >-
            [
            {url:'http://kube-api.loc/openapi/v2',name:'kubernetes'}
            ]

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: swagger-ui-ingress
  namespace: swagger
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: swagger.loc
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: 
            name: swagger-ui
            port:
              number: 80
```

At this point you should be able to access Swagger UI on http://swagger.loc/


## Yayy

I hope you succed and enjoy carfree hand-on experience with Kubernetes REST API
