# Open WebUI

AI chat interface, gated by oauth2-proxy.

- Namespace: `open-webui`, served at `owui.<domain>`.
- Postgres StatefulSet (`17.10-bookworm`, 2Gi PVC) holds chats/users.
- Open WebUI Deployment (`0.10.2`, 4Gi PVC for cache/RAG/uploaded files),
  storage on `local-path`.
- Fronted by oauth2-proxy in reverse-proxy mode behind a Traefik Ingress
  (`letsencrypt-prod` cert, HTTP→HTTPS redirect). oauth2-proxy requires a
  Keycloak login before any request reaches the app.

Open WebUI is not exposed directly: it trusts the identity headers
oauth2-proxy sets (`WEBUI_AUTH_TRUSTED_EMAIL_HEADER` /
`WEBUI_AUTH_TRUSTED_NAME_HEADER`) to sign the user in — no separate Open
WebUI credentials. A native OIDC client against the same Keycloak realm is
also wired but disabled by default (ConfigMap generator is commented out in
`overlay-prod/kustomization.yaml`). 

## Apply

```bash
kubectl apply -k overlay-prod
```

## Notes

**API routes bypass the auth gate.**
`OAUTH2_PROXY_SKIP_AUTH_ROUTES` exempts these paths from oauth2-proxy:

```
GET=^/api/models$
POST=^/api/chat/completions$
POST=^/api/message$
POST=^/api/v1/messages$
```

These routes bypass Keycloak SSO but are still protected by Open WebUI's 
API-token auth, which is what lets non-interactive clients authenticate.

**SPA shell caching.**
The catch-all `/` path is served with `Cache-Control: no-store`; 
only `/_app/immutable` (content-hashed build assets) gets a long-lived
`immutable` cache. This prevents a logout failure when browser is serving a 
disk-cached copy of the SPA document instead of revalidating it which put the 
post-logout redirect into a reload loop. Forcing the document to always
revalidate — while letting content-hashed assets under /_app/immutable cache
long-term — breaks the loop.
