# Infrastructure

Cluster-scoped resources and cluster-level configuration that every app in
this repo depends on.

## cert-manager

Installed in the `cert-manager` namespace (not managed by this repo). Two
ClusterIssuers, defined in [`issuers/`](issuers/), both solving ACME HTTP-01
challenges through Traefik:

- `letsencrypt-staging` — untrusted certs, generous rate limits. Referenced by
  every app's base manifests, so it's the default anyone applying an app
  lands on.
- `letsencrypt-prod` — trusted certs, strict rate limits. Patched in by each
  app's `overlay-prod`. Promoting to a trusted cert is an explicit step.

