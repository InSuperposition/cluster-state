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
directory names, not in values. `clusters/mgmt/` describes what a *management*
cluster runs, whether that cluster happens to be a single-node OrbStack VM,
an EC2 instance, or bare metal. A role is what a cluster is FOR; it is not an
environment.

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
`postBuild.substitute` (configured in `infra`'s `modules/gitops-flux/flux-aio.cue`).

| Variable | Meaning | Supplied by |
|---|---|---|
| `${CLUSTER_NAME}` | identity of the running cluster | `infra`, from its `cluster_name` output |
| `${K8S_SERVICE_HOST}` | how pods reach the API server | `infra`, per deployment topology |
| `${K8S_SERVICE_PORT}` | the port that address answers on | `infra`, per deployment topology |

The test for whether a value belongs here or in `infra`: *would it differ
between two environments running this same cluster role?* Chart versions,
IPAM mode and Hubble being enabled would not. API address and cluster
identity would.

## Layout

```
modules/                    # component definitions, one directory each
  net-cilium/
    helmrepository-cilium.yaml
    cilium.yaml             # Cilium + Hubble (one release -- Hubble rides Cilium's)
    kustomization.yaml
  security-tetragon/
    tetragon.yaml
    kustomization.yaml
clusters/                   # cluster ROLES -- Flux syncs one of these paths
  mgmt/kustomization.yaml   # the management cluster
```

`modules/` names each component `<role>-<tool>`, matching `infra`'s module
naming taxonomy: what it does first, what implements it second. A role
directory holds nothing but a list of the bases that role runs, so every
component is defined exactly once. When a role first needs to differ, the
difference is a patch in the role directory, never a copy of the base.

**The directory is `modules/`, not `infrastructure/`.** Both repos now use the
same word for the same idea — a composable building block that is never
deployed on its own — so `infra`'s naming taxonomy describes this repo as
written rather than by analogy. `infrastructure/` was also the weaker name on
its own terms: every file in this repo is infrastructure, so it distinguished
nothing, which is the same test that rejected `gitops` as this repo's name and
`k0s` as a module suffix. The rename is directory-only; the manifests, the
component names and the rendered output are unchanged.

There is no `apps/` directory. There will be when there is an application to
put in it; an empty one now would only be a guess about a shape nobody has
needed yet.

## One role directory, for now

There is a single cluster role, `mgmt` -- the management cluster, the one
bootstrapped directly rather than provisioned by another cluster. A second role
is a second directory listing whichever bases it needs; the bases themselves do
not change.

`v0.4.0` briefly carried a second directory, `core`, rendering byte-identical
output. That existed for one window and one reason: `infra` derives the synced
path from its own role output at runtime

```bash
CLUSTER_STATE_PATH="./clusters/$(tofu output -raw cluster_role)"
```

so the role could not be renamed to `mgmt` before `clusters/mgmt/` existed, and
`clusters/mgmt/` could not be the only path while any `infra` revision still
said `core`. `v0.5.0` closes that window. Pin `v0.5.0` or later only from an
`infra` revision whose `cluster_role` is `mgmt`.

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
git tag v0.6.0 && git push --tags          # here
# then in infra: bump the ref in modules/gitops-flux/flux-aio.cue and ./infra flux
```

Rendering a role before tagging is worth the two seconds -- it catches a base
path that moved without its consumers:

```bash
kustomize build clusters/mgmt
```

For a rename that is meant to change nothing, diff the render against the
previous tag rather than eyeballing it. `kustomize build` succeeds on a
role whose base list silently lost an entry, so "it still renders" is not
evidence:

```bash
git stash && kustomize build clusters/mgmt > /tmp/before.yaml && git stash pop
kustomize build clusters/mgmt | diff /tmp/before.yaml -
```

## Bootstrap

Not from here. See `infra`'s `docs/USAGE.md` — "Bootstrapping from zero".
