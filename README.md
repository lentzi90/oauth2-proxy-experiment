# Oauth2 proxy for k8s

```
minikube start
helm init
helm upgrade --install nginx-ingress stable/nginx-ingress --set controller.service.externalIPs[0]=$(minikube ip)
helm upgrade --install dex stable/dex -f dex-values.yml
helm upgrade --install oauth2 stable/oauth2-proxy -f oauth2-proxy-values.yml
```

Some resources:

- https://blog.heptio.com/on-securing-the-kubernetes-dashboard-16b09b1b7aca
- https://itnext.io/protect-kubernetes-dashboard-with-openid-connect-104b9e75e39c
- https://thenewstack.io/single-sign-on-for-kubernetes-dashboard-experience/
