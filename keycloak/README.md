# Keycloak

OpenID Connect provider for every gated app in this repo.

- Namespace: `iam`, served at `auth.<domain>`.
- Postgres StatefulSet (`18.4-bookworm`, 2Gi PVC on `local-path`) holds
  realm/user data.
- Keycloak StatefulSet (`26.6.4`, running `start` — not `--optimized`),
  fronted by a Traefik Ingress with a `letsencrypt-prod` cert and
  HTTP→HTTPS redirect.

## Apply

```bash
kubectl apply -k overlay-prod
```
