# Oauth2 proxy for k8s

This is a demonstration of how to use [oauth2-proxy](https://github.com/pusher/oauth2_proxy) in combination with [dex](https://github.com/dexidp/dex) to achieve a smooth login experience for the Kubernetes dashboard.

It is based on the following guides:

- https://blog.heptio.com/on-securing-the-kubernetes-dashboard-16b09b1b7aca
- https://itnext.io/protect-kubernetes-dashboard-with-openid-connect-104b9e75e39c
- https://thenewstack.io/single-sign-on-for-kubernetes-dashboard-experience/

The dashboard is installed as normal, except that it uses the insecure port without TLS since oauth2-proxy cannot handle the self-signed certificate.
Oauth2-proxy acts as an ingress for the dashboard and checks all incoming requests.
If a specific authentication cookie is present, the request is proxied straight to the dashboard service, otherwise it will be rerouted to dex for authentication.

The thing that really makes this setup useful is that oauth2-proxy also sets to authorization bearer token on each request to the dashboard.
This means that the user does not have to copy/paste any token or upload a kubeconfig file to get access to the dashboard.
You just log in through dex and immediately get access to the dashboard based on your personal access rights.

```
+------------------------+       +------------------+
|                        |       |                  |
|  Ingress               |       |  Ingress         |
|  dashboard.example.com |   +--->  dex.example.com |
|                        |   |   |                  |
+-------+----------------+   |   +-------+----------+
        |                    |           |
        |                    |           |
+-------v----------+         |    +------v--------+       +----------------+
|                  |         |    |               |       |                |
|  Service         |         |    |  Service      |       |   Service      |
|  Oauth2-proxy    |         |    |  dex          |   +--->   dashboard    |
|                  |         |    |               |   |   |                |
+-------+----------+         |    +------+--------+   |   +-------+--------+
        |                    |           |            |           |
+-------v----------+         |    +------v--------+   |   +-------v--------+
|                  |         |    |               |   |   |                |
|  Pod             +---------+    |  Pod          |   |   |  Pod           |
|  Oauth2-proxy    |              |  dex          |   |   |  dashboard     |
|                  |              |               |   |   |                |
+--------+---------+              +---------------+   |   +----------------+
         |                                            |
         |                                            |
         |                                            |
         +--------------------------------------------+
```

## Quick start

Dex generates a self-signed certificate when started the first time.
This certificate must be made available to the API server for it to trust dex as an identity provider.
To accomplish this, we have to restart minikube (with additional flags) after installing dex the first time.

```
minikube start
# Wait for it to start, deploy dex and then restart once the certificate has
# been generated.
MINIKUBE_IP=$(minikube ip)
helm init --wait
helm upgrade --install dex stable/dex --version 1.5.0 -f dex-values.yml \
  --set ingress.hosts[0]=dex.$MINIKUBE_IP.nip.io \
  --set ingress.tls[0].hosts[0]=dex.$MINIKUBE_IP.nip.io \
  --set certs.web.altNames[0]=dex.$MINIKUBE_IP.nip.io \
  --set config.issuer=https://dex.$MINIKUBE_IP.nip.io \
  --set config.staticClients[0].redirectURIs[0]=http://dashboard.$MINIKUBE_IP.nip.io/oauth2/callback

minikube start \
  --extra-config=apiserver.oidc-issuer-url=https://dex.$MINIKUBE_IP.nip.io \
  --extra-config=apiserver.oidc-username-claim=email \
  --extra-config=apiserver.oidc-client-id="example-app" \
  --extra-config=apiserver.oidc-ca-file=/var/lib/minikube/certs/oidc.pem

helm upgrade --install nginx-ingress stable/nginx-ingress --version 1.6.13 \
  --set controller.service.externalIPs[0]=$MINIKUBE_IP
helm upgrade --install oauth2 stable/oauth2-proxy --version 0.12.2 \
  -f oauth2-proxy-values.yml \
  --set extraArgs.redirect-url=http://dashboard.$MINIKUBE_IP.nip.io/oauth2/callback \
  --set extraArgs.oidc-issuer-url=https://dex.$MINIKUBE_IP.nip.io \
  --set ingress.hosts[0]=dashboard.$MINIKUBE_IP.nip.io

kubectl apply -f dashboard.yaml
kubectl apply -f rbac.yaml

echo "Log in to the k8s dashboard at http://dashboard.$MINIKUBE_IP.nip.io"
echo "Use admin@example.com:password for credentials"
```
