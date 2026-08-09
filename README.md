# cluster-state

Desired in-cluster state (L3), reconciled by Flux. The companion to
[`infra`](https://github.com/InSuperposition/infra), which provisions the
compute and Kubernetes underneath it (L1/L2) and bootstraps Flux.

Named for its role, not its mechanism: "gitops" would describe *how* this is
delivered, which is true of every repo in this setup and therefore
distinguishes nothing — the same reason `k0s` was rejected as a module suffix
in `infra`'s naming taxonomy. `cluster-state` also survives replacing Flux
with something else.

## Layout

```
clusters/
  vm-orbstack-dev/          # one directory per cluster; Flux syncs this path
    helmrepository-cilium.yaml
    cilium.yaml             # Cilium + Hubble (one release -- Hubble rides Cilium's)
    tetragon.yaml
```

Per-cluster directories rather than a shared base with overlays. There is one
cluster today; a base/overlay split with a single consumer would be
generalising from one example. Revisit when a second cluster actually exists
and its real differences are known.

## What owns what

`infra` owns the Flux installation itself and the sync objects that point
here — the `flux-aio` and `flux-git-sync` Timoni instances. This repo owns
everything Flux then reconciles. Neither manages the other's objects; that
boundary is what keeps bootstrap reproducible and prevents Flux from fighting
its own configuration.

## Conventions

- **Versions are pinned exactly.** No `"*"`, no floating ranges. A chart
  bump is a commit here.
- **`storageNamespace` is always explicit.** Flux defaults it to the
  HelmRelease's own namespace (`flux-system`), which does not match releases
  that store their state in `kube-system` — the difference between adopting
  an existing release and failing to find it.
- **Values reflect what actually runs**, captured from `helm get values`
  rather than from whatever flags an install script passed. CLIs inject
  values of their own; rebuilding from the flags alone loses them.

## Bootstrap

Not from here. See `infra`'s `./infra flux`, which installs Flux AIO and
points it at this repo at a pinned tag.
