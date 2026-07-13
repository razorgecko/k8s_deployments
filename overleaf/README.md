# Overleaf

LaTeX editor. Community Edition has no SSO support, so oauth2-proxy only
gates network access — it does not carry identity into the app. Overleaf
keeps its own separate login/credentials behind the proxy.

- Namespace: `overleaf`, served at `overleaf.<domain>`.
- MongoDB StatefulSet (`8.0`, single-node replica set `rs0`, 5Gi PVC) and
  Redis Deployment (`7.4`, 1Gi PVC) back the Overleaf Deployment
  (`sharelatex 6.2.1`, 10Gi PVC for projects/compiles), storage on
  `local-path`.
- Fronted by oauth2-proxy in reverse-proxy mode behind a Traefik Ingress
  (`letsencrypt-prod` cert, HTTP→HTTPS redirect). oauth2-proxy requires a
  Keycloak login before any request reaches Overleaf; Overleaf then presents
  its own login on top of that.

## Apply

```bash
kubectl apply -k overlay-prod
```
