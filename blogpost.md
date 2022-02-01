# How to use Istio and OAuth2-Proxy as a layer in front of your application to authenticate through OIDC in Kubernetes

This post will show how [Istio](https://istio.io/) can be used to force users to authenticate before accessing applications.
Picture a use case were you are working on an application with a microservice architecture. The end user have
multiple endpoints to connect to and you want all of them to be protected behind some access control. Instead
of having each service provide its own authentication code you want some separate service to handle this for all
your applications. 

My solution for this is to use Istio. There are many posts and guides on different benefits and
use cases for Istio but this is a rarer use case I could not find any detailed examples about. The solution
comes down to using Istio and its authorization policies to route all requests to specific hostnames through
an [Oauth2-Proxy](https://oauth2-proxy.github.io/oauth2-proxy/) to any Identity provider (IDP)
supporting [OIDC](https://openid.net/connect/).

## Prerequisites

* A running Kubernetes cluster
* Two dns records pointing into the cluster. The guide uses a LoadBalancer behind a public IP, but a NodePort can also be used.
* Kubectl and helm3 cli installed.
* Clone/fork the repository https://github.com/vanneback/istio-external-auth-demo

In my example I used a kubernetes cluster in AKS. With a public IP exposed behind a LoadBalancer. I also have two dns-records
which in the guide is replaced by `httpbin.example.com` and `dex.example.com`

## Installing the sample application

To demonstrate this an example application called httpbin is used. This application frequently occurs in the Istio
guides which makes it a perfect app for this example. Start by installing namespaces and the application.

```
kubectl apply -f namespaces.yaml
kubectl apply -f httpbin-deploy.yaml
```

Now the application should be installed and accessible only through the cluster. Check installation with.
```
kubectl get pods -n demo
kubectl port-forward -n demo svc/httpbin 8000:8000
```
There should be one pod deployed in demo with only 1/1 containers ready. When installing Istio there will be a
sidecar added here. Access the application on localhost:8000

## Installing Istio

In this guide I was using Azures AKS which has the option to use the LoadBalancer service type with a static IP.
If you also use Azure replace the IPs in `istio-controlplane.yaml` with your public IP. If you are not using a
provider with support for LoadBalancers you can replace this with NodePorts. An example of this is commented in
the `istio-controlplane.yaml` file. After the config is ready install Istio with:

```
kubectl apply -f istio-1.12.1/manifests/charts/base/crds/crd-all.gen.yaml
./istio-1.12.1/bin/istioctl operator init
kubectl apply -f istio-controlplane.yaml
./istio-1.12.1/bin/istioctl verify-install
```
The most interesting thing about this config is the meshConfig parameters:
```yaml
  meshConfig:
    extensionProviders:
    - name: "oauth2-proxy"
      envoyExtAuthzHttp:
        service: "oauth2-proxy.demo.svc.cluster.local"
        port: "80"
        headersToDownstreamOnDeny:
          - content-type
          - set-cookie
        headersToUpstreamOnAllow:
          - authorization
          - cookie
          - path
          - x-auth-request-access-token
          - x-forwarded-access-token
        includeHeadersInCheck:
          - "cookie"
          - "x-forwarded-access-token"
```
This config is creating a new `provider` with the name `oauth2-proxy`. This can later be used in the authorization-policy
to route requests through the oauth2-proxy. The rest of this config is basically what headers to include to make sure the
correct tokens are passed along.

If you delete the httpbin-pod now it should restart with a sidecar. Check that it creates a pod with 2/2 containers ready:
```
kubectl delete pod -n demo --all
kubectl get pods -n demo
```

## Create certificates and gateways

To make the example represent a correct installation we will use cert-manager to create some valid certificates for us.
Install cert-manager with helm:

```
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.4.0 \
  --set installCRDs=true \
  --set startupapicheck.enabled=false
```

Replace `email` in `cluster-issuer.yaml` then replace the occurrences of `httpbin.example.com` with the url
you want to use for your httpbin-app. Do this in both `httpbin-istio-gw.yaml` and `httpbin-tls-cert.yaml` then apply with:
```
kubectl apply -f cluster-issuer.yaml
kubectl apply -f httpbin-istio-gw.yaml
kubectl apply -f httpbin-tls-cert.yaml
```

One thing to note here is how the certificates are created in the istio-system namespace. This is a limitation when using [Istio
with cert-manager](https://istio.io/latest/docs/ops/integrations/certmanager/). For more information about Istio gateways and services
go to the [Istio docs](https://istio.io/latest/docs/reference/config/networking/gateway/)

### Validate

You can validate the installation by checking pods and certificates with:
```
kubectl get pods -n cert-manager
kubectl get pods -n demo
kubectl get certificate -n istio-system
```

This should show ready pods an a ready certificate. Then visit your url `httpbin.example.com`. You can also
go to `httpbin.example.com/headers` to see the included headers. At the moment there should not be any header called
`Authorization` which should be added when the authentication flow is complete.

## Installing Dex

For authentication any IDP which supports OIDC can be used. In this example [Dex](https://dexidp.io/) is installed
which can in turn be connected to other AD sources see our blog post on 
[how to connect dex to google](https://elastisys.com/elastisys-engineering-how-to-use-dex-with-google-accounts-to-manage-access-in-kubernetes/) 
for more information. In this post we will only use Dex static user as an example.

Install dex by replacing the occurrences of `dex.example.com` with the url you want to use for dex. Do this in `dex-values.yaml`,
`dex-istio-gw.yaml` and `dex-tls-cert.yaml`.
Then run:
```
helm repo add dex https://charts.dexidp.io
helm repo update
helm install \
  --namespace demo \
  --values dex-values.yaml \
  --version 0.6.5 \
  dex dex/dex

kubectl apply -f dex-istio-gw.yaml
kubectl apply -f dex-tls-cert.yaml
```

## Installing oauth2-proxy
The final tool needed is [OAuth2-Proxy](https://oauth2-proxy.github.io/oauth2-proxy/). Oauth2-proxy is an open source software
handling the authentication flow needed for OAuth2 or in this case OIDC. This will handle the Authentication flow and pass the 
needed token back to the application. 

Install by replacing `oidc_issuer_url` and `cookie_domains` from `oauth2-proxy-values.yaml` with your domain name then apply with:

```
helm repo add oauth2-proxy https://oauth2-proxy.github.io/manifests
helm repo update
helm install \
  --namespace demo \
  --values oauth2-proxy-values.yaml \
  --version 5.0.6 \
  oauth2-proxy oauth2-proxy/oauth2-proxy
```

## Apply authorization policy
Finally apply the authorization-policy to tell Istio what requests should be routed through the oauth2-proxy.
```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: oauth-policy
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  action: CUSTOM
  provider:
    name: "oauth2-proxy"
  rules:
  - to:
    - operation:
        hosts:
        - "httpbin.example.com"
```

The most important part of this config is the provider name which has to match the provider created in the meshConfig part of 
the `istio-controlplane.yaml` and the `hosts` list which is the host names that the policy applies to.

Apply by replacing `httpbin.example.com` with you app url in `authorization-policy.yaml` then run:
```
kubectl apply -f authorization-policy.yaml
```

The authorization policy will trigger when trying to access the hostname configured. 
When the policy is triggered it will use the extensionProvider from the `istio-controlplane.yaml` config.
This will cause a redirect to the oauth2-proxy which in turn will go to dex for authentication. 

### Validation

You should now be able to access httpbin on your url for `httpbin.example.com`. This should take you directly to the Dex login
page were you can authenticate with:
```
Username: admin
Password: password
```

To check the authorization token visit `httpbin.example.com/headers` and it should be a header named `Authorization` with a jwt token.

## Authentication and Authorization

If you have made it this far you have probably heard of authentication vs authorization before. I just want to make it clear
that this use case of Istio is only used for Authentication. Istio will route to the authentication service and if a valid token is presented it will pass you
on to the application. This provides some security for all the applications but the application will still have to be
responsible for Authorization and any form of RBAC.

## ID Token vs Access Token

To be brief `ID Tokens` are passed by the Authentication server as a proof that you have been authenticated. It can be used
to get information about the user such as name, email etc. It should however not be the token used by the Applications to
decide what you are allowed to do. For this an `Access Token` should be used. For a more detailed explanation see this 
[blog post](https://auth0.com/blog/id-token-access-token-what-is-the-difference/)

An issue with the OAut2 Proxy is that it sends the `ID Token` under the `Authorization` header. And its difficult to 
even access the `Access Token`. A solution for this is first to enable the options under `config.configFile` in the oauth2-proxy helm chart:
``` yaml
    set_xauthrequest = true
    set_authorization_header = true
    pass_authorization_header = true 
    pass_host_header = true
    pass_access_token = true
```

The entire config is in `oauth2-proxy-values.yaml`. The second part is to enable Istio to pass this header on to the end applications
by adding this line to the meshConfig in `istio-controlplane.yaml`:
```yaml
  includeAdditionalHeadersInCheck:
    authorization: '%REQ(x-auth-request-access-token)%'
```
The final part which is not displayed in these examples is by having istio replacing the `Authorization` header in the virtualService
resource. A full example of this is displayed below.
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  namespace: demo
  name: httpbin-vsvc
spec:
  hosts:
  - "httpbin.example.com"
  gateways:
  - httpbin-gateway
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        port:
          number: 8000
        host: httpbin.demo.svc.cluster.local
      headers:
        request:
          set:
            Authorization: 'Bearer %REQ(x-auth-request-access-token)%'
```

Even if this was not the default I included in the examples it can be important to know and was not trivial to find how
to make this replacement. The optimal solution would be if this is included as an option or even the default in the Oauth2-proxy.
But as of writing this post this is not the case.

## Conclusion

Hopefully this blog gives an insight on how Istio together with OAuth2 Proxy can be used as layer in front of applications
were authentication is needed. At least I hope it provides some clarity how to configure Istio to do this, and perhaps it can help 
make your decision on how to handle authentication in microservices easier.