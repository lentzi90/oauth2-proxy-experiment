# Oauth2 proxy for k8s

```
minikube start
helm init
helm upgrade --install nginx-ingress stable/nginx-ingress --set controller.service.externalIPs[0]=$(minikube ip)
helm upgrade --install dex stable/dex -f dex-values.yml
helm upgrade --install oauth2 stable/oauth2-proxy -f oauth2-proxy-values.yml
```
