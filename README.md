## Installation

### Installing base application
```
# install namespaces
kubectl apply -f namespaces.yaml

# Install application
kubectl apply -f httpbin-deploy.yaml
```
Now the application should be installed and accessable only through the cluster. Check installation with.
```
kubectl get pods -n demo
kubectl port-forward -n demo svc/httpbin 8000:8000
```
There should be one pod deployed in demo with only 1/1 containers ready. When installing istio there will be a
sidecar added here. Access the application on localhost:8000

### Installing istio


```
kubectl apply -f istio-1.12.1/manifests/charts/base/crds/crd-all.gen.yaml
./istio-1.12.1/bin/istioctl operator init
kubectl apply -f istio-controlplane.yaml
./istio-1.12.1/bin/istioctl verify-install
```


### Installing cert-manager

```
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.6.1 \
  --set installCRDs=true \
  --set startupapicheck.enabled=false
```

### Create gateways and certificates

```
kubectl apply -f cluster-issuer.yaml
kubectl apply -f httpbin-istio-gw.yaml
kubectl apply -f httpbin-tls-cert.yaml
```