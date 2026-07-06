# K8s deployments

Kubernetes manifests: one directory per app, deployed with Kustomize.
Tested on a single-node K3s cluster.

## Layout

Each app directory follows the same structure:

- `base/` — namespace plus one subdirectory per component (database,
  oauth2-proxy, the app itself), aggregated in a root `kustomization.yaml`.
- `overlay-prod/` — sets the namespace, pins image tags, generates
  ConfigMaps/Secrets from `.env` files, and carries env-specific patches and
  replacements.

Apply an app with:

```bash
kubectl apply -k <app>/overlay-prod
```

## Infrastructure

`infrastructure/` holds cluster-scoped resources that apps depend on, grouped by
kind:
- `issuers/` — Let's Encrypt ClusterIssuers (cert-manager).

## Apps

- Keycloak — OpenID provider with a Postgres backend (ns: `iam`)
- Open WebUI — AI interface with a Postgres backend, 
  gated by oauth2-proxy (ns: `open-webui`).

## Secrets

Real config and secrets live in `*.env` files consumed by the Kustomize
generators and are gitignored.
A committed `<name>.env.example` sits next to each as a template.
