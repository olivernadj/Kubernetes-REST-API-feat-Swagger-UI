# Kubernetes-REST-API-feat-Swagger-UI
How to unlock Kubernetes REST API and link it to Swagger UI

**WARNING!!** Do not apply this changes on publicly available cluster unless, 
you hold your keys under the mat.


In case view only is sufficient, I found better guide for you: 
https://jonnylangefeld.com/blog/kubernetes-how-to-view-swagger-ui

---

Interacting with the Kubernetes REST API via Swagger UI is not that 
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

The yaml manifests will:
  1. Create `api-proxy` namespace
  2. Bind default `ServiceAccount` to `cluster-admin` roles.
  3. Create a proxy pass nginx config with with actual Bearer token and certificates.
  4. Deploy and expose nginx as a proxy for kube api.
  5. Create Ingress with `kube-api.loc` host and CORS enabled. 

Add `kube-api.loc` to `/etc/hosts` and make it point to `127.0.0.1`.

```bash
kubectl apply -f https://raw.githubusercontent.com/olivernadj/Kubernetes-REST-API-feat-Swagger-UI/main/api-proxy.yaml
```

At this point you should be able to access Kube API on http://kube-api.loc/api



## Install Swagger UI

The yaml manifests will:
 1. Create `swagger` namespace
 2. Deploy and expose Swagger UI.
 3. Create Ingress with `swagger.loc` host. 

Add `swagger.loc` to `/etc/hosts` and make it point to `127.0.0.1`.

```bash
kubectl apply -f https://raw.githubusercontent.com/olivernadj/Kubernetes-REST-API-feat-Swagger-UI/main/swagger.yaml
```

At this point you should be able to access Swagger UI on http://swagger.loc/


## Yayy!! \o/

I hope you succeed and enjoy carefree hand-on experience with Kubernetes REST API

![](https://raw.githubusercontent.com/olivernadj/Kubernetes-REST-API-feat-Swagger-UI/main/get-pods.png)