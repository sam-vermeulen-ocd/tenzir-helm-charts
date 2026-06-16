# Tenzir Helm Charts

| Chart | Purpose |
|---|---|
| [`charts/tenzir-node`](charts/tenzir-node) | Deploy one or more static `tenzir-node` instances. |

## Local validation

Both commands assume [Nix](https://nixos.org/download) is installed. They
fetch `helm` and `trivy` on first run and cache them. No system install of
either tool is required.

```sh
# Render and lint the chart against its values schema.
nix shell nixpkgs#kubernetes-helm \
  --command helm lint charts/tenzir-node

# Validate the rendered manifests against the Kubernetes API schemas
# (catches typos / wrong field nesting that `helm lint` does not).
nix shell nixpkgs#kubernetes-helm nixpkgs#kubeconform \
  --command sh -c '
    helm template tn charts/tenzir-node -f examples/full.yaml \
      | kubeconform -strict -summary -kubernetes-version 1.33.3 -
  '

# Render with a realistic values file, then scan the manifests for
# misconfigurations. examples/full.yaml exercises most chart knobs.
nix shell nixpkgs#trivy \
  --command trivy config charts/tenzir-node \
    --helm-values examples/full.yaml \
    --severity HIGH,CRITICAL,MEDIUM

# Render to stdout for ad-hoc inspection.
nix shell nixpkgs#kubernetes-helm \
  --command helm template tn charts/tenzir-node -f examples/full.yaml
```

See [`charts/tenzir-node/README.md`](charts/tenzir-node/README.md) for the
full configuration reference.
