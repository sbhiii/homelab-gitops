[← Back to README](../README.md) · [Architecture](architecture.md) · [Getting started](getting-started.md) · [Operations](operations.md)

# Security model

## The NetworkPolicy layer

Every app in this repo (`argocd`, `cert-manager`, `traefik`) carries an identical `networkpolicy.yml` denying egress from its namespace to `169.254.169.254` — the Hetzner metadata service, which serves the cluster's ServiceAccount token-signing key unauthenticated. The full reasoning for *why* that address matters lives in `sre-homelab`'s [security model](https://github.com/sbhiii/sre-homelab/blob/main/docs/security.md); this repo is where the mitigation is actually declared.

This is **defense in depth, not the primary control.** The primary mitigation is a host-level `iptables` rule installed by `sre-homelab`'s cloud-init script, which covers every namespace uniformly because it operates below Kubernetes entirely. These `NetworkPolicy` objects are the secondary layer, and they have a real limitation the host rule doesn't: **`NetworkPolicy` is namespaced.** `default`, `kube-system`, and any namespace added to this repo without its own copy of `networkpolicy.yml` are not covered. Each policy file says as much in its own comment — read one directly if you're touching this.

## RBAC: letting `cert-manager` mint its own token

`apps/cert-manager/rbac.yml` grants exactly one thing: `create` on `serviceaccounts/token`, scoped via `resourceNames` to the `cert-manager` ServiceAccount alone, in the `cert-manager` namespace alone. Without it, `cert-manager` can't request the projected token it exchanges with AWS STS — the whole OIDC mechanism in the other repo depends on this one narrow grant existing here. It's a `Role` + `RoleBinding` pair, not a `ClusterRole`; nothing here reaches outside the `cert-manager` namespace.

## What secret management currently doesn't exist here

There is no [sealed-secrets](https://github.com/bitnami-labs/sealed-secrets), no External Secrets Operator, and no other in-cluster secret manager in this repo. That's not an oversight — sealed-secrets was removed after its Helm chart repository started returning 404, and by the time it was removed it had no consumers left anyway: `cert-manager`'s AWS credential was eliminated entirely by the OIDC migration (nothing to encrypt when there's no static credential), and the Traefik dashboard's basic-auth secret was removed along with the dashboard's public exposure (see below).

**If you add something that genuinely needs a secret** — a database password, an API token for some third-party integration — there is currently nowhere in this repo to put it safely. The planned path is [SSM Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html), most likely fronted by External Secrets Operator, authenticating through the same OIDC trust chain `cert-manager` already uses — one additional IAM role in `sre-homelab`, no new credential mechanism. Until that lands, don't commit a manifest that assumes a secret exists without first deciding where it actually comes from.

## Why the Traefik dashboard isn't exposed

`apps/traefik/kustomization.yml` sets `dashboard: true` (the dashboard runs) but disables its `IngressRoute` and carries no `Ingress` of its own. It's reachable only via:

```bash
kubectl port-forward -n traefik deploy/traefik 9000:9000
```

Its authentication used to be HTTP Basic Auth backed by a `SealedSecret`. Once that secret manager was removed, the honest options were: publish the dashboard with no authentication at all, or don't publish it. The dashboard reveals every route, service, and TLS configuration on the cluster — for a single-operator homelab, port-forward-only access costs nothing and closes that off entirely.

## Known limitations

- **`NetworkPolicy` coverage is per-namespace, and incomplete.** `default` and `kube-system` are not protected by anything in this repo. See [The NetworkPolicy layer](#the-networkpolicy-layer) above.
- **Cross-repo values are copied by hand.** `hostedZoneID`, `role`, and the repo URL itself are literal strings here, sourced from `sre-homelab`'s Terraform outputs with nothing gluing the two together. See [Getting started](getting-started.md#forking-this-repo-for-your-own-cluster).
- **No secret management exists yet.** See above.
- **No CI.** Nothing runs `kubectl kustomize --enable-helm` against every app on a pull request; it's done by hand before merging.

---

[← Back to README](../README.md)
