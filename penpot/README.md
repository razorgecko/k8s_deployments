# Penpot

Design and prototyping platform. Deployed from the upstream Helm chart
(`penpot/penpot`), which ships no databases: PostgreSQL and Valkey are provided
here. Penpot supports OIDC natively, so oauth2-proxy gates network access and
Penpot separately signs the user in against Keycloak — two Keycloak clients.

- Namespace: `penpot`, served at `penpot.<domain>`.
- Postgres StatefulSet (`18.4-bookworm`, 10Gi PVC) and Valkey Deployment
  (`9.1.0-trixie`, 1Gi PVC) back the Helm release (frontend, backend, exporter;
  10Gi assets PVC on the `fs` objects-storage backend), storage on `local-path`.
- Fronted by oauth2-proxy (`v7.15.3`) in reverse-proxy mode behind a Traefik
  Ingress (`letsencrypt-prod` cert, HTTP→HTTPS redirect). The chart's own
  Ingress is disabled.
- Keycloak is the only allowed login method (`disable-login-with-password`);
  first login registers the account (`enable-oidc-registration`).

## Split of ownership

| kustomize (`base/`, `overlay-prod/`)                           | Helm (`values-prod.yaml`)                           |
| -------------------------------------------------------------- | --------------------------------------------------- |
| namespace, Postgres, Valkey, oauth2-proxy, Ingress, Middleware | frontend, backend, exporter, assets PVC, SA         |
| all Secrets                                                    | references those Secrets by name (`existingSecret`) |

No secret material is passed to Helm, so none is stored in Helm's release
Secret either.

## Apply

Copy the `*.env.example` files in `overlay-prod/` to `*.env`, and
`values-prod.yaml.example` to `values-prod.yaml`; fill in real values.

Kustomize first — the namespace and the Secrets the chart references must exist:

```bash
kubectl apply -k overlay-prod
helm install penpot penpot/penpot --version 1.6.1 -n penpot -f values-prod.yaml
```

oauth2-proxy proxies every non-`/oauth2` request to the frontend Service the
chart creates (`penpot.penpot.svc.cluster.local:8080`), so it serves errors
until the Helm release is up.

## Keycloak clients

- `oauth2-proxy` — used by oauth2-proxy. Redirect URI
  `https://penpot.<domain>/oauth2/callback`.
- `penpot` — used by Penpot itself. Redirect URI
  `https://penpot.<domain>/api/auth/oauth/oidc/callback`.

## Notes

**The OIDC `baseURI` needs a trailing slash.** In `values-prod.yaml`,
`config.providers.oidc.baseURI` must end in `/` — without it Penpot treats the
realm segment as a file name and replaces it.

**Long upstream timeout.** `OAUTH2_PROXY_UPSTREAM_TIMEOUT=300s` keeps requests
from being cut off while working with heavy design files.
