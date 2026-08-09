# cluster-state

Desired in-cluster state (L3), reconciled by Flux. The companion to
[`infra`](https://github.com/InSuperposition/infra), which provisions the
compute and Kubernetes underneath it (L1/L2) and bootstraps Flux.

Named for its role, not its mechanism: "gitops" would describe *how* this is
delivered, which is true of every repo in this setup and therefore
distinguishes nothing — the same reason `k0s` was rejected as a module suffix
in `infra`'s naming taxonomy. `cluster-state` also survives replacing Flux
with something else.

## This repo knows nothing about environments

No provider, tier or environment name appears anywhere here — not in
directory names, not in values. `clusters/core/` describes what a *core*
cluster runs, whether that cluster happens to be a single-node OrbStack VM,
an EC2 instance, or bare metal.

That is the whole boundary: **`infra` owns environments, `cluster-state` owns
cluster content.** An earlier version of this repo put a provider and a tier
in the cluster directory name and hardcoded a full cluster identity into a
Helm value — both facts this repo has no business knowing.

The rule, precisely: **no provider, tier or environment name in any
directory name, file name or value.** Prose may name one as an illustration
(as above) — that is explanation, not configuration. What must never happen
is a real name reaching a manifest, which is how this repo drifted the first
time.

## Environment-specific values

Anything genuinely environment-specific is a `${VARIABLE}` here, supplied by
whichever environment instantiates the cluster, through Flux's
`postBuild.substitute` (configured in `infra`'s `bootstrap/flux-aio.cue`).

| Variable | Meaning | Supplied by |
|---|---|---|
| `${CLUSTER_NAME}` | identity of the running cluster | `infra`, from its `cluster_name` output |
| `${K8S_SERVICE_HOST}` | how pods reach the API server | `infra`, per deployment topology |

The test for whether a value belongs here or in `infra`: *would it differ
between two environments running this same cluster role?* Chart versions,
IPAM mode and Hubble being enabled would not. API address and cluster
identity would.

## Layout

```
clusters/
  core/                     # the "core" cluster role -- Flux syncs this path
    helmrepository-cilium.yaml
    cilium.yaml             # Cilium + Hubble (one release -- Hubble rides Cilium's)
    tetragon.yaml
```

One directory per cluster *role*, not per cluster instance. There is one role
today; a base/overlay split with a single consumer would be generalising from
one example. Revisit when a second role exists and its real differences are
known.

## What owns what

`infra` owns the Flux installation itself and the sync objects that point
here — the `flux-aio` and `flux-git-sync` Timoni instances. This repo owns
everything Flux then reconciles. Neither manages the other's objects; that
boundary is what keeps bootstrap reproducible and prevents Flux from fighting
its own configuration.

## Conventions

- **No environment names.** See above. If you find yourself wanting one, the
  value belongs in `infra` as a substitution instead.
- **Versions are pinned exactly.** No `"*"`, no floating ranges. A chart
  bump is a commit here, and a tag.
- **`storageNamespace` is always explicit.** Flux defaults it to the
  HelmRelease's own namespace (`flux-system`), which does not match releases
  that store their state in `kube-system` — the difference between adopting
  an existing release and failing to find it.
- **Values reflect what actually runs**, captured from `helm get values`
  rather than from whatever flags an install script passed. CLIs inject
  values of their own; rebuilding from the flags alone loses them.

## Releasing

`infra` bootstraps against an immutable tag, never a branch, so a rebuild
reproduces the same cluster. Promoting a change is two steps:

```bash
git tag v0.2.0 && git push --tags          # here
# then in infra: bump the ref in bootstrap/flux-aio.cue and ./infra flux
```

## Bootstrap

Not from here. See `infra`'s `docs/USAGE.md` — "Bootstrapping from zero".
