[← Back to README](../README.md) · [Architecture](architecture.md) · [Operations](operations.md) · [Security model](security.md)

# Getting started

## Prerequisites

### Tools

| Tool | Used for |
|---|---|
| `kubectl` | applying nothing directly (ArgoCD does that), but essential for `kubectl kustomize --enable-helm` to validate changes locally and for reading cluster state |
| `helm` | required on `PATH` for `kubectl kustomize --enable-helm` to resolve the `helmCharts:` blocks — ArgoCD's repo-server needs the server-side equivalent, already configured if you're using this alongside `sre-homelab` |
| `git` | this is a plain GitOps repo; there's no CLI of its own |

### Accounts and existing infrastructure

This repo is not self-sufficient — it's the second half of a pair. To actually use it, you need:

- **A running k3s/Kubernetes cluster with ArgoCD installed**, and an `Application` seeded to point at this repo's `bootstrap/` directory. [`sre-homelab`](https://github.com/sbhiii/sre-homelab) is exactly that — its Terraform installs ArgoCD and seeds `root-app` at first boot.
- **An IAM role and hosted zone that `cert-manager` can use for Route53 DNS-01**, if you keep the OIDC-federated `ClusterIssuer` as-is. `sre-homelab`'s `iac/aws` module produces both — see its [architecture doc](https://github.com/sbhiii/sre-homelab/blob/main/docs/architecture.md#the-oidc-trust-chain). If you don't want that mechanism, `cert-manager`'s Route53 solver also supports a plain access-key `Secret`; that means editing `apps/cert-manager/cluster-issuer.yml` yourself, since this repo assumes the federated approach throughout.
- **A domain you control**, for the ingress hostnames in `apps/argocd/ingress.yml` and anything you add later.

### Background knowledge

- **ArgoCD**: the app-of-apps pattern, `Application` CRDs, sync waves, and what `automated: {prune: true, selfHeal: true}` actually does to a manual `kubectl edit` (see [Operations](operations.md)).
- **Kustomize**: `resources:`, and the `helmCharts:` field specifically — it's a Kustomize builtin, not a separate tool, and it's how every chart in this repo is pulled in.
- **Kubernetes fundamentals**: `Ingress`, `NetworkPolicy`, `Role`/`RoleBinding` — enough to read a manifest and know what it grants or exposes.
- **ACME DNS-01, at a conceptual level**: what a `ClusterIssuer` solver does and why it needs a hosted zone ID and a way to write records. The *mechanics* of how it authenticates without a static credential are `sre-homelab`'s concern, not this repo's — see that repo's [Architecture](https://github.com/sbhiii/sre-homelab/blob/main/docs/architecture.md) if you want them.

## Forking this repo for your own cluster

If you're standing up your own instance of this pair of repos rather than just reading, here's everything that has to change together. Missing one of these is the most common way to end up with an ArgoCD that syncs against someone else's fork, or a `ClusterIssuer` that can't authenticate.

**1. The repo URL is hardcoded in three places, not one.** Every file in `bootstrap/` carries the same literal `repoURL`:

```bash
grep -rn 'repoURL' bootstrap/
```

Point all three at your fork. (A fourth copy lives in `sre-homelab`'s `terraform.tfvars` as `github_repo_url` — that's what seeds `root-app` in the first place, so it has to match too.)

**2. `apps/cert-manager/cluster-issuer.yml` needs three values from your own AWS setup:**

```yaml
hostedZoneID: <from sre-homelab's `terraform output hosted_zone_id`>
role: <from sre-homelab's `terraform output cert_manager_role_arn`>
email: <a real address you control>
```

These are Terraform outputs in the other repo but literal strings here — there's no automation copying them across. See [Known limitations](security.md#known-limitations) for why, and [`sre-homelab`'s getting-started guide](https://github.com/sbhiii/sre-homelab/blob/main/docs/getting-started.md) for where these values come from.

**3. Ingress hostnames.** `apps/argocd/ingress.yml` hardcodes `argocd.homelab.sbhi.io` in two places (the `tls.hosts` entry and the `rules.host`). Change both to your own domain, and make sure a DNS record actually points there — `sre-homelab`'s `iac/aws/apps-dns.tf` creates a wildcard for this, but only for its own zone.

**4. Push, and watch `root-app` pick it up:**

```bash
kubectl -n argocd get applications
```

Give it a few minutes — `cert-manager` needs its wave to finish and issue a real certificate before Traefik's wave completes and the ingress starts serving.

Next: [Operations](operations.md) for adding a new app, or [Security model](security.md) for what's actually protecting the cluster.

---

[← Back to README](../README.md) · [Architecture →](architecture.md)
