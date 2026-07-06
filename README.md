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

## Secrets

Real config and secrets live in `*.env` files consumed by the Kustomize
generators and are gitignored.
A committed `<name>.env.example` sits next to each as a template.
