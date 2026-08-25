[← Back to README](../README.md) · [Architecture](architecture.md) · [Getting started](getting-started.md) · [Operations](operations.md)

# Security model

## The NetworkPolicy layer

Every app in this repo (`argocd`, `cert-manager`, `traefik`) carries an identical `networkpolicy.yml` denying egress from its namespace to `169.254.169.254` — the Hetzner metadata service, which serves the cluster's ServiceAccount token-signing key unauthenticated. The full reasoning for *why* that address matters lives in `homelab`'s [security model](https://github.com/sbhiii/homelab/blob/main/docs/security.md); this repo is where the mitigation is actually declared.

This is **defense in depth, not the primary control.** The primary mitigation is a host-level `iptables` rule installed by `homelab`'s cloud-init script, which covers every namespace uniformly because it operates below Kubernetes entirely. These `NetworkPolicy` objects are the secondary layer, and they have a real limitation the host rule doesn't: **`NetworkPolicy` is namespaced.** `default`, `kube-system`, and any namespace added to this repo without its own copy of `networkpolicy.yml` are not covered. Each policy file says as much in its own comment — read one directly if you're touching this.

## How `cert-manager` gets its AWS credentials, and what it is not allowed to do

The kubelet projects a ServiceAccount token into the controller pod with `audience: sts.amazonaws.com`, and the AWS SDK inside `cert-manager` exchanges it for temporary credentials via `AssumeRoleWithWebIdentity`. `AWS_ROLE_ARN` and `AWS_WEB_IDENTITY_TOKEN_FILE` tell it where to look. This is the same mechanism EKS calls IRSA, and unlike cert-manager's own `serviceAccountRef` it works for any AWS SDK workload, not just this solver.

The role ARN is committed here, account ID and all. That is deliberate: AWS does not treat account IDs as secret, and the role is assumable only by a token signed by this cluster's key that satisfies both trust policy conditions. The protection is the conditions, not the obscurity of the ARN.

`cert-manager` holds **no Kubernetes permission** for any of it. An earlier version used `auth.kubernetes.serviceAccountRef`, where `cert-manager` minted its own token through the TokenRequest API, and that required a `Role` granting `create` on `serviceaccounts/token`. Projecting the token through the kubelet removes the need for that grant entirely, so it was deleted rather than left in place unused.

The audience on the projected token is load-bearing. It must equal the `aud` condition on the IAM role's trust policy, and dropping that condition is the most common IRSA misconfiguration there is: it lets a token minted for any audience assume the role.

## What secret management currently doesn't exist here

There is no [sealed-secrets](https://github.com/bitnami-labs/sealed-secrets), no External Secrets Operator, and no other in-cluster secret manager in this repo. That's not an oversight — sealed-secrets was removed after its Helm chart repository started returning 404, and by the time it was removed it had no consumers left anyway: `cert-manager`'s AWS credential was eliminated entirely by the OIDC migration (nothing to encrypt when there's no static credential), and the Traefik dashboard's basic-auth secret was removed along with the dashboard's public exposure (see below).

**If you add something that genuinely needs a secret** — a database password, an API token for some third-party integration — there is currently nowhere in this repo to put it safely. The planned path is [SSM Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html), most likely fronted by External Secrets Operator, authenticating through the same OIDC trust chain `cert-manager` already uses — one additional IAM role in `homelab`, no new credential mechanism. Until that lands, don't commit a manifest that assumes a secret exists without first deciding where it actually comes from.

## The Traefik dashboard is not served at all

`apps/traefik/kustomization.yml` sets `api.insecure: false`, so the API and dashboard are not served on the `traefik` entrypoint. The dashboard's `IngressRoute` is disabled and there is no `Ingress`, so nothing reaches it from outside either. It is unreachable, including by `kubectl port-forward`.

That is a deliberate loss of a diagnostic. Reading Traefik's live routing table is genuinely useful, and it is given up because the alternative is worse.

**Why the obvious fix is not the fix.** For a while this was `api.insecure: true` with `ports.traefik.expose.default: true`, which put the unauthenticated API on the `traefik` `Service` where any pod could read every route, service and TLS configuration without a credential. Removing the port from the `Service` looks like it would close that, and does not: Kubernetes pod networking is flat, so a pod can reach the Traefik pod's IP on 8080 whether or not a `Service` points at it. The `NetworkPolicy` layer here is egress-only and imposes no ingress restriction either. The only thing that actually closes it is not serving the API on that port.

**The entrypoint stays up.** Both probes hit `/ping` on 8080, so port 8080 keeps listening; `--ping=true` survives while `--api.insecure=false` removes the dashboard from it.

**Getting the dashboard back** means a secured `IngressRoute`, which needs authentication, which needs somewhere to keep a credential. That is the gap described above under secret management. Until it exists, the honest options were an unauthenticated dashboard readable by every workload in the cluster, or no dashboard, and this repo picks no dashboard for the same reason it stopped publishing it externally.

## Known limitations

- **`NetworkPolicy` coverage is per-namespace, and incomplete.** `default` and `kube-system` are not protected by anything in this repo. See [The NetworkPolicy layer](#the-networkpolicy-layer) above.
- **Cross-repo values are copied by hand.** The repo URL, the role ARN and the hosted zone ID are literal strings here, sourced from `homelab`'s Terraform outputs with nothing gluing the two together. See [Getting started](getting-started.md#forking-this-repo-for-your-own-cluster).
- **No secret management exists yet.** See above.
- **No CI.** Nothing runs `kubectl kustomize --enable-helm` against every app on a pull request; it's done by hand before merging.

---

[← Back to README](../README.md)
