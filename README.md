# sre-homelab-gitops

Everything [ArgoCD](https://argo-cd.readthedocs.io/) applies to the [`sre-homelab`](https://github.com/sbhiii/sre-homelab) cluster: the app-of-apps bootstrap, `cert-manager`'s `ClusterIssuer`, Traefik, and the `NetworkPolicy` layer that's half of a metadata-service mitigation whose other half lives in the infrastructure repo.

Once the cluster exists, **this repo is the only input to it.** Every `Application` here syncs with `automated: {prune: true, selfHeal: true}` — a manual `kubectl edit` against anything ArgoCD owns gets reverted on the next reconcile. If you want to change what runs on the cluster, this is the repo to open a PR against, not `kubectl`.

## Documentation

| | |
|---|---|
| **[Architecture](docs/architecture.md)** | The app-of-apps pattern and sync waves, kustomize+helm, ingress conventions, and the one asymmetry in bootstrap order worth knowing about |
| **[Getting started](docs/getting-started.md)** | Prerequisites and how to fork this repo and point your own cluster at it — including the values that have to be copied in by hand from `sre-homelab`'s Terraform outputs |
| **[Operations](docs/operations.md)** | Adding an app, validating locally before pushing, reading ArgoCD sync failures |
| **[Security model](docs/security.md)** | The `NetworkPolicy` layer, the RBAC that lets `cert-manager` mint its own token, and what secret management currently doesn't exist here |

## Related repository

[`sre-homelab`](https://github.com/sbhiii/sre-homelab) builds the cluster and the AWS identity `cert-manager` uses to reach Route53 — Terraform, three modules, OIDC federation instead of a static access key. Read that repo for how the cluster and its AWS identity get built; read this one for what actually runs on it.
