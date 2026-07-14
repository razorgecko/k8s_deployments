# K8s deployments

A self-hosted single-sign-on platform: Kubernetes manifests for a small set of
apps that all authenticate through one Keycloak realm, sitting behind Traefik
with automated TLS. One directory per app, deployed with Kustomize.
Tested on a single-node K3s cluster; multi-node HA is out of scope by design —
this targets a personal/homelab footprint.

## Architecture

Traefik (bundled with K3s) terminates TLS and routes by host/path.
cert-manager requests and renews certs automatically via ACME HTTP-01.
Keycloak is the sole identity provider. oauth2-proxy sits in front of apps in
reverse-proxy mode and requires a Keycloak login before any request reaches 
either app — but what that login gets depends on the SSO support of the apps.
See each app's README for the specifics.

## Tech stack

- **Identity**: Keycloak (OIDC), oauth2-proxy
- **Apps**: Open WebUI, Overleaf (Community Edition), Penpot
- **Data stores**: Postgres, MongoDB, Redis, Valkey
- **Ingress / TLS**: Traefik, cert-manager
- **Orchestration**: K3s, Kustomize, Helm (Penpot)

Pinned versions live in each app's README.

## Prerequisites

- A K3s cluster (Traefik ingress controller ships with it by default).
- [cert-manager](https://cert-manager.io/) installed, with ports 80/443
  reachable from the internet for ACME HTTP-01 challenges.
- A domain with DNS A records for each app's host pointed at the
  cluster.
- `kubectl` with Kustomize support (`kubectl apply -k`), and `helm` for Penpot.

## Bootstrap order

1. `infrastructure/issuers` — ClusterIssuers must exist before any app can
   request a cert.
2. `keycloak` — every other app's oauth2-proxy depends on a reachable realm.
3. Regular apps — independent of each other, in any order.

For each app, copy the `*.env.example` files in `overlay-prod/` to `*.env`,
fill in real values, then:

```bash
kubectl apply -k <app>/overlay-prod
```

Base manifests point at the `letsencrypt-staging` issuer; `overlay-prod`
patches the annotation to `letsencrypt-prod`. Test first apply
against staging while sorting out DNS/ingress to avoid burning through
production ACME limits. See [`infrastructure/`](infrastructure/) for how
the issuers and cluster-level settings are set up.

## Layout

Each app directory follows the same structure:

- `base/` — namespace plus one subdirectory per component (database, the app
  itself), aggregated in a root `kustomization.yaml` that also pulls in
  `common/oauth2-proxy`.
- `overlay-prod/` — sets the namespace, pins image tags, generates
  ConfigMaps/Secrets from `.env` files, and carries env-specific patches and
  replacements.

[`common/`](common/) holds manifests shared by every app: `oauth2-proxy`,
configured per app by the ConfigMap and Secret its overlay generates.

## Infrastructure

[`infrastructure/`](infrastructure/) holds cluster-scoped resources and
cluster-level config that every app depends on — ClusterIssuers.

## Apps

- [Keycloak](keycloak/) — OpenID provider with a Postgres backend (ns: `iam`)
- [Open WebUI](openwebui/) — AI interface with a Postgres backend, gated by
  oauth2-proxy via trusted headers (ns: `open-webui`). Native OIDC login is
  configured but disabled by default.
- [Overleaf](overleaf/) — LaTeX editor with MongoDB and Redis backends, gated
  by oauth2-proxy (ns: `overleaf`)
- [Penpot](penpot/) — design platform with Postgres and Valkey backends,
  deployed from the upstream Helm chart, gated by oauth2-proxy and signing in
  natively against Keycloak (ns: `penpot`)

## Secrets

Real config and secrets live in `*.env` files consumed by the Kustomize
generators and are gitignored.
A committed `<name>.env.example` sits next to each as a template.
