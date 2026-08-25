[← Back to README](../README.md) · [Architecture](architecture.md) · [Getting started](getting-started.md) · [Security model](security.md)

# Operations

## Adding a new app

1. **Create `apps/<name>/`** with at least a `kustomization.yml`. Follow an existing app as a template — `apps/cert-manager/` if it needs a Helm chart, `apps/argocd/` if it's just plain manifests.
2. **Copy `networkpolicy.yml` from an existing app** into the new directory, editing only the `namespace:` field, and add it to the new `kustomization.yml`'s `resources:`. This is easy to forget precisely because nothing fails loudly if you do — the new namespace just stays exposed to the Hetzner metadata service like `default` and `kube-system` already are. See [Security model](security.md) for why that matters.
3. **Add `bootstrap/<name>-app.yml`**, copied from an existing bootstrap file, with:
   - `metadata.name` matching the app
   - `spec.source.path` pointing at `apps/<name>`
   - `spec.destination.namespace` set to wherever it should land
   - `argocd.argoproj.io/sync-wave` set to reflect real dependencies — see [Architecture: sync waves](architecture.md#sync-waves-and-why-this-order) for why this isn't cosmetic
4. **Validate locally before pushing** (see below), then commit and push. `root-app` picks up the new file on its own — nothing needs registering by hand.

## Validating locally before pushing

ArgoCD only ever sees what's committed, so catching a broken manifest before it merges is entirely on local validation:

```bash
kubectl kustomize --enable-helm apps/<name>
```

This must succeed for every app with a `helmCharts:` block, or ArgoCD's repo-server will fail to render it the same way. `helm` needs to be on `PATH` for this — see [Getting started](getting-started.md#tools).

## Reading ArgoCD's state

```bash
kubectl -n argocd get applications
```

`SYNCED`/`Healthy` is the goal state for everything. Two other states worth knowing:

- **`OutOfSync`** means the live cluster state doesn't match this repo — usually resolves itself within the next automated sync, since every `Application` here has `selfHeal: true`.
- **`Progressing` that never ends, with everything healthy underneath.** `argocd-app` sits like this permanently. ArgoCD's built-in health check for an `Ingress` only reports `Healthy` once `status.loadBalancer.ingress` is populated, and nothing here ever populates it: Traefik is a `ClusterIP` service using `hostPort`, so there is no load balancer to write an address. ArgoCD is waiting for something that by design never arrives. Routing works regardless. Check `kubectl -n argocd get application <name> -o json` and look at `status.resources[].health` before chasing it: if every resource is `Healthy` or has no health check, this is what you are looking at.
- **`Missing`** is a *health* status: the resources ArgoCD expects do not exist in the cluster. A failure to render the manifests at all is a different thing, and shows as sync status `Unknown` with a `ComparisonError` condition, so filtering on `Missing` will not surface it. Either way, `kubectl -n argocd get application <name> -o jsonpath='{.status.conditions}'` prints the actual error.

**A real failure mode worth knowing, because it already happened once:** ArgoCD rejects an *entire* `Application` if any single resource inside it references a CRD that doesn't exist in the cluster, not just the broken resource.

What triggered it here was not a leftover. The `sealed-secrets` Helm chart repository started returning 404, so that `Application` never synced, so the `bitnami.com/v1alpha1` CRD was never installed at all. A `SealedSecret` in `apps/traefik` then referenced a kind the cluster had never heard of, and the effect was not "that one secret fails", it was "the whole `traefik-ingress` Application shows `Missing`, including the Helm chart that installs Traefik itself, leaving the cluster with no ingress controller at all."

The order matters for diagnosis. The obvious suspicion is a stale reference left behind by something you deleted; the actual case was a CRD that never arrived because the `Application` owning it failed upstream. So when an `Application` won't sync and the error names a `group version` or kind that "could not be found", check whether the `Application` that provides that CRD is itself healthy before hunting for leftovers.

## Forcing ArgoCD to look at the repository now

ArgoCD polls, and the interval is not a promise. A merged commit can sit
unapplied for longer than `timeout.reconciliation` suggests, with the
`Application` still reporting `Synced` against the older revision it last saw.
That reads like a broken sync and is not one.

Compare what it has against what you pushed:

```bash
kubectl -n argocd get application <name> -o jsonpath='{.status.sync.revision}'
git rev-parse HEAD
```

If they differ, stop waiting:

```bash
kubectl -n argocd annotate application <name> argocd.argoproj.io/refresh=hard --overwrite
```

Reach for this before assuming anything is wrong. Twice now a change that looked
stuck was simply not yet fetched.

## Getting into the ArgoCD UI

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

Username `admin`. Change the password on first login and delete the bootstrap
secret afterwards, which is what upstream recommends:

```bash
kubectl -n argocd delete secret argocd-initial-admin-secret
```

**It does not survive a rebuild.** That secret is generated at install, so a node
replacement mints a new one and any password you set is gone with the old
cluster. Do not store it anywhere expecting it to stay valid.

## Manual changes don't stick

Every `Application` here syncs with `automated: {prune: true, selfHeal: true}`. A `kubectl edit` or `kubectl apply` against anything ArgoCD owns gets reverted on the next reconcile loop (default: every few minutes, or immediately if you trigger a manual sync). If you need to change something running on the cluster, the change belongs in this repo, not on the live objects — that's the entire point of the setup.

---

[← Back to README](../README.md) · [Security model →](security.md)
