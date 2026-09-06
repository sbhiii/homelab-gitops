[← Back to README](../README.md) · [Architecture](architecture.md) · [Getting started](getting-started.md) · [Operations](operations.md)

# Security model

## The NetworkPolicy layer

Every app in this repo (`argocd`, `cert-manager`, `traefik`, `podinfo`, `external-secrets`) carries an identical `networkpolicy.yml` denying egress from its namespace to `169.254.169.254` — the Hetzner metadata service, which serves the cluster's ServiceAccount token-signing key unauthenticated. The full reasoning for *why* that address matters lives in `homelab`'s [security model](https://github.com/sbhiii/homelab/blob/main/docs/security.md); this repo is where the mitigation is actually declared.

This is **defense in depth, not the primary control.** The primary mitigation is a host-level `iptables` rule installed by `homelab`'s cloud-init script, which covers every namespace uniformly because it operates below Kubernetes entirely. These `NetworkPolicy` objects are the secondary layer, and they have a real limitation the host rule doesn't: **`NetworkPolicy` is namespaced.** `default`, `kube-system`, and any namespace added to this repo without its own copy of `networkpolicy.yml` are not covered. Each policy file says as much in its own comment — read one directly if you're touching this.

## How `cert-manager` gets its AWS credentials, and what it is not allowed to do

The kubelet projects a ServiceAccount token into the controller pod with `audience: sts.amazonaws.com`, and the AWS SDK inside `cert-manager` exchanges it for temporary credentials via `AssumeRoleWithWebIdentity`. `AWS_ROLE_ARN` and `AWS_WEB_IDENTITY_TOKEN_FILE` tell it where to look. This is the same mechanism EKS calls IRSA, and unlike cert-manager's own `serviceAccountRef` it works for any AWS SDK workload, not just this solver.

The role ARN is committed here, account ID and all. That is deliberate: AWS does not treat account IDs as secret, and the role is assumable only by a token signed by this cluster's key that satisfies both trust policy conditions. The protection is the conditions, not the obscurity of the ARN.

`cert-manager` holds **no Kubernetes permission** for any of it. An earlier version used `auth.kubernetes.serviceAccountRef`, where `cert-manager` minted its own token through the TokenRequest API, and that required a `Role` granting `create` on `serviceaccounts/token`. Projecting the token through the kubelet removes the need for that grant entirely, so it was deleted rather than left in place unused.

The audience on the projected token is load-bearing. It must equal the `aud` condition on the IAM role's trust policy, and dropping that condition is the most common IRSA misconfiguration there is: it lets a token minted for any audience assume the role.

## Where secrets come from

Secrets live in AWS SSM Parameter Store, in the `sbhi-homelab` account, and reach the cluster through [External Secrets Operator](https://external-secrets.io/). Nothing in this repository holds a secret value.

Two things use it today: the basic-auth credential in front of every exposed app, and ArgoCD's admin password, which is read from Parameter Store so that it survives a cluster rebuild rather than being regenerated.

An `ExternalSecret` names a parameter path and the `Secret` it should produce. It is a pointer, not a secret, which is why it is safe to commit here while the value it names never leaves AWS. The operator fetches on `refreshInterval` and writes a real `Secret` into the namespace.

The operator authenticates through the same OIDC trust chain `cert-manager` uses, with no stored credential. Two details are worth knowing because neither is obvious:

- **The role ARN lives on the ServiceAccount**, in an `eks.amazonaws.com/role-arn` annotation, not in the `ClusterSecretStore`. The name is EKS-branded and this cluster is k3s, which invites deleting it; the operator reads the key itself, and the EKS pod identity webhook is not involved. `spec.provider.aws.role` is a *different* field, for chaining a second role after the initial exchange.
- **The token audience is not set explicitly.** The operator already requests `sts.amazonaws.com` and appends anything in `serviceAccountRef.audiences`, and the API server does not deduplicate. Naming it again produces a token with two audiences, which AWS rejects because OIDC requires an `azp` claim once `aud` holds more than one value.

`external-secrets-ssm` is a ServiceAccount that no pod runs as. It exists only as the identity in the IAM trust policy's `sub` condition, so one store maps to one role and a second store can later have its own without widening the first. This is scoping, not containment: the operator holds `serviceaccounts/token: create` cluster-wide and can mint a token for any ServiceAccount.

**Parameter values are created out of band**, with `aws ssm put-parameter`, and are not managed by Terraform. `aws_ssm_parameter.value` is a `computed` attribute, so it is read back into state on every refresh regardless of `ignore_changes` — managing values there would put them in the bucket that already holds the cluster's signing key.

**What still cannot come from here.** Anything ArgoCD needs before it can sync this repository, because the operator is installed by the repository. That is a real boundary, not a gap to close: a credential needed to reach the secret store cannot come from the secret store.

## The Traefik dashboard is not served at all

`apps/traefik/kustomization.yml` sets `api.insecure: false`, so the API and dashboard are not served on the `traefik` entrypoint. The dashboard's `IngressRoute` is disabled and there is no `Ingress`, so nothing reaches it from outside either. It is unreachable, including by `kubectl port-forward`.

That is a deliberate loss of a diagnostic. Reading Traefik's live routing table is genuinely useful, and it is given up because the alternative is worse.

**Why the obvious fix is not the fix.** For a while this was `api.insecure: true` with `ports.traefik.expose.default: true`, which put the unauthenticated API on the `traefik` `Service` where any pod could read every route, service and TLS configuration without a credential. Removing the port from the `Service` looks like it would close that, and does not: Kubernetes pod networking is flat, so a pod can reach the Traefik pod's IP on 8080 whether or not a `Service` points at it. The `NetworkPolicy` layer here is egress-only and imposes no ingress restriction either. The only thing that actually closes it is not serving the API on that port.

**The entrypoint stays up.** Both probes hit `/ping` on 8080, so port 8080 keeps listening; `--ping=true` survives while `--api.insecure=false` removes the dashboard from it.

**Getting the dashboard back** means a secured `IngressRoute`. The basic-auth middleware every other exposed app uses would serve, so this is now a choice rather than a constraint. It stays unexposed because `kubectl port-forward -n traefik deploy/traefik 9000:9000` reaches it and nothing justifies a public route to the cluster's live routing table.

## Everything internet-facing is behind basic auth

Ports 80 and 443 are open to `0.0.0.0/0` at the Hetzner firewall, because that is how any app here is reached. Every `Ingress` therefore carries a Traefik `basicAuth` middleware, including ones with nothing worth protecting:

```yaml
traefik.ingress.kubernetes.io/router.middlewares: <namespace>-basic-auth@kubernetescrd
```

The credential comes from Parameter Store, hashed into htpasswd format by the operator's `htpasswd` template function. The stored value is the password itself; no hash is committed anywhere.

**Traefik cannot reference a Middleware across namespaces**, so every namespace with an exposed `Ingress` carries its own `Middleware` and its own `ExternalSecret`. They read the same parameter, so there is still one password. Adding an exposed app means copying both files, not just writing an `Ingress`.

**`podinfo` is why this is a rule rather than a judgement call.** It is a demo app holding nothing, and it would be easy to argue it needs no protection. But it is an HTTP testing toolkit: `/env` returns the pod's environment, `/panic` crashes the pod, `/delay/{n}` holds connections open on a single-node cluster. The interesting endpoints are rarely on the app you decided was important.

**ArgoCD sits behind basic auth as well**, in front of its own login. Two consequences worth knowing: the `argocd` CLI does not pass the outer credential, and a GitHub webhook to `/api/webhook` would need it too.

## Known limitations

- **`NetworkPolicy` coverage is per-namespace, and incomplete.** `default` and `kube-system` are not protected by anything in this repo. See [The NetworkPolicy layer](#the-networkpolicy-layer) above.
- **Cross-repo values are copied by hand.** The repo URL, the two role ARNs and the hosted zone ID are literal strings here, sourced from `homelab`'s Terraform outputs with nothing gluing the two together. See [Getting started](getting-started.md#forking-this-repo-for-your-own-cluster).
- **One credential fronts every exposed app.** A single basic-auth password, in one Parameter Store entry, guards ArgoCD and `podinfo` alike. Per-app credentials would be a second parameter and a second `ExternalSecret` per namespace; it has not been worth it at two apps.
- **No CI.** Nothing runs `kubectl kustomize --enable-helm` against every app on a pull request; it's done by hand before merging.

---

[← Back to README](../README.md)
