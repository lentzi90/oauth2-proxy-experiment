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
minikube start \
  --extra-config=apiserver.oidc-issuer-url=https://dex.192.168.99.100.nip.io \
  --extra-config=apiserver.oidc-username-claim=email \
  --extra-config=apiserver.oidc-client-id="example-app"
# Wait for it to start, deploy dex and then restart once the certificate has
# been generated.
helm init --wait
helm upgrade --install dex dex -f dex-values.yml

minikube start \
  --extra-config=apiserver.oidc-issuer-url=https://dex.192.168.99.100.nip.io \
  --extra-config=apiserver.oidc-username-claim=email \
  --extra-config=apiserver.oidc-client-id="example-app" \
  --extra-config=apiserver.oidc-ca-file=/var/lib/minikube/certs/oidc.pem

helm upgrade --install nginx-ingress stable/nginx-ingress --set controller.service.externalIPs[0]=$(minikube ip)
helm upgrade --install oauth2 stable/oauth2-proxy -f oauth2-proxy-values.yml
kubectl apply -f dashboard.yaml
kubectl apply -f rbac.yaml
```

Now try to login with `admin@example.com:password` at [dashboard.192.168.99.100.nip.io](http://dashboard.192.168.99.100.nip.io).
