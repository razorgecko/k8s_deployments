# Common

Manifests shared by every app. Not applied on its own — each app's
`base/kustomization.yaml` pulls a component in as a resource
(`../../common/oauth2-proxy`), so it lands in that app's namespace.

## oauth2-proxy

ServiceAccount, Service (port 4180) and Deployment for the auth gate that fronts
each app in reverse-proxy mode: it requires a Keycloak login and proxies every
non-`/oauth2` request upstream.

The manifests carry no app-specific values. The Deployment takes its whole
configuration from two objects the app's `overlay-prod` generates, and its image
tag from the overlay's `images:` block:

- `oauth2-proxy-config` ConfigMap — from `oauth2-proxy.cm.env`: OIDC issuer,
  client ID, cookie and upstream settings.
- `oauth2-proxy-secret` Secret — from `oauth2-proxy.secret.env`: client secret
  and cookie secret.

The Ingress and Traefik middlewares that route to the Service stay with the app,
since hosts and per-app routing differ.
