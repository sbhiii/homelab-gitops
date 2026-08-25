[← Back to README](../README.md) · [Architecture](architecture.md) · [Getting started](getting-started.md) · [Operations](operations.md)

# Security model

## The NetworkPolicy layer

Every app in this repo (`argocd`, `cert-manager`, `traefik`) carries an identical `networkpolicy.yml` denying egress from its namespace to `169.254.169.254` — the Hetzner metadata service, which serves the cluster's ServiceAccount token-signing key unauthenticated. The full reasoning for *why* that address matters lives in `sre-homelab`'s [security model](https://github.com/sbhiii/sre-homelab/blob/main/docs/security.md); this repo is where the mitigation is actually declared.

This is **defense in depth, not the primary control.** The primary mitigation is a host-level `iptables` rule installed by `sre-homelab`'s cloud-init script, which covers every namespace uniformly because it operates below Kubernetes entirely. These `NetworkPolicy` objects are the secondary layer, and they have a real limitation the host rule doesn't: **`NetworkPolicy` is namespaced.** `default`, `kube-system`, and any namespace added to this repo without its own copy of `networkpolicy.yml` are not covered. Each policy file says as much in its own comment — read one directly if you're touching this.

## How `cert-manager` gets its AWS credentials, and what it is not allowed to do

The kubelet projects a ServiceAccount token into the controller pod with `audience: sts.amazonaws.com`, and the AWS SDK inside `cert-manager` exchanges it for temporary credentials via `AssumeRoleWithWebIdentity`. `AWS_ROLE_ARN` and `AWS_WEB_IDENTITY_TOKEN_FILE` tell it where to look. This is the same mechanism EKS calls IRSA, and unlike cert-manager's own `serviceAccountRef` it works for any AWS SDK workload, not just this solver.

The role ARN is committed here, account ID and all. That is deliberate: AWS does not treat account IDs as secret, and the role is assumable only by a token signed by this cluster's key that satisfies both trust policy conditions. The protection is the conditions, not the obscurity of the ARN.

`cert-manager` holds **no Kubernetes permission** for any of it. An earlier version used `auth.kubernetes.serviceAccountRef`, where `cert-manager` minted its own token through the TokenRequest API, and that required a `Role` granting `create` on `serviceaccounts/token`. Projecting the token through the kubelet removes the need for that grant entirely, so it was deleted rather than left in place unused.

The audience on the projected token is load-bearing. It must equal the `aud` condition on the IAM role's trust policy, and dropping that condition is the most common IRSA misconfiguration there is: it lets a token minted for any audience assume the role.

## What secret management currently doesn't exist here

There is no [sealed-secrets](https://github.com/bitnami-labs/sealed-secrets), no External Secrets Operator, and no other in-cluster secret manager in this repo. That's not an oversight — sealed-secrets was removed after its Helm chart repository started returning 404, and by the time it was removed it had no consumers left anyway: `cert-manager`'s AWS credential was eliminated entirely by the OIDC migration (nothing to encrypt when there's no static credential), and the Traefik dashboard's basic-auth secret was removed along with the dashboard's public exposure (see below).

**If you add something that genuinely needs a secret** — a database password, an API token for some third-party integration — there is currently nowhere in this repo to put it safely. The planned path is [SSM Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html), most likely fronted by External Secrets Operator, authenticating through the same OIDC trust chain `cert-manager` already uses — one additional IAM role in `sre-homelab`, no new credential mechanism. Until that lands, don't commit a manifest that assumes a secret exists without first deciding where it actually comes from.

## The Traefik dashboard is not exposed externally, but is reachable in-cluster

`apps/traefik/kustomization.yml` sets `dashboard: true`, disables the dashboard's `IngressRoute`, and carries no `Ingress` of its own, so nothing reaches it from outside the cluster.

It is not private inside the cluster. The same file sets `api.insecure: true` and `ports.traefik.expose.default: true`, which puts the unauthenticated API on the `traefik` `Service`:

```bash
kubectl port-forward -n traefik deploy/traefik 8080:8080   # from a laptop
curl http://traefik.traefik.svc:8080/dashboard/            # from any pod, no credential
```

The `NetworkPolicy` layer in this repo is egress-only and imposes no ingress restriction, so it does not close this off either.

Authentication used to be HTTP Basic Auth backed by a `SealedSecret`. When that secret manager was removed the choice was to publish the dashboard unauthenticated or stop publishing it, and external exposure was dropped. What remains is the in-cluster surface: the dashboard reveals every route, service and TLS configuration, and any workload here can read it without credentials.

**Known gap, not a decision.** Closing it means setting `api.insecure: false` or `ports.traefik.expose.default: false`. Neither is set today.

## Known limitations

- **`NetworkPolicy` coverage is per-namespace, and incomplete.** `default` and `kube-system` are not protected by anything in this repo. See [The NetworkPolicy layer](#the-networkpolicy-layer) above.
- **Cross-repo values are copied by hand.** The repo URL, the role ARN and the hosted zone ID are literal strings here, sourced from `sre-homelab`'s Terraform outputs with nothing gluing the two together. See [Getting started](getting-started.md#forking-this-repo-for-your-own-cluster).
- **No secret management exists yet.** See above.
- **No CI.** Nothing runs `kubectl kustomize --enable-helm` against every app on a pull request; it's done by hand before merging.

---

[← Back to README](../README.md)
