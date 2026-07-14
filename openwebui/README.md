# Open WebUI

AI chat interface, gated by oauth2-proxy.

- Namespace: `open-webui`, served at `owui.<domain>`.
- Postgres StatefulSet (`17.10-bookworm`, 2Gi PVC) holds chats/users.
- Open WebUI Deployment (`0.10.2`, 4Gi PVC for cache/RAG/uploaded files),
  storage on `local-path`.
- Fronted by oauth2-proxy in reverse-proxy mode behind a Traefik Ingress
  (`letsencrypt-prod` cert, HTTP→HTTPS redirect). oauth2-proxy requires a
  Keycloak login before any request reaches the app.

Open WebUI is not exposed directly: requests pass through oauth2-proxy, and the
app then signs the user in natively via OIDC against the same Keycloak realm.
The app is configured to make the second sign-in silent by hiding the password
form (`ENABLE_LOGIN_FORM`) and redirecting the request (`OAUTH_AUTO_REDIRECT`).
Password auth is off (`ENABLE_PASSWORD_AUTH`), so Keycloak is the only method
to log in.

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

**`ui.*` config is persisted in the database.**
After first boot, `ui.*`/`webui.*` settings are read from the `config` table,
so changing the env var alone has no effect.
Use `ENABLE_PERSISTENT_CONFIG=false` for env vars in ConfigMap to have
effect. **WARNING:** The action discards settings configured through UI.

**SPA shell caching.**
The catch-all `/` path is served with `Cache-Control: no-store`; 
only `/_app/immutable` (content-hashed build assets) gets a long-lived
`immutable` cache. This prevents a logout failure when browser is serving a 
disk-cached copy of the SPA document instead of revalidating it which put the 
post-logout redirect into a reload loop. Forcing the document to always
revalidate — while letting content-hashed assets under /_app/immutable cache
long-term — breaks the loop.
