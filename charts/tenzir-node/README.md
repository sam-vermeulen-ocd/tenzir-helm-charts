# Tenzir Node Helm Chart

This chart deploys one or more static `tenzir-node` instances on Kubernetes. It
does not deploy the Tenzir platform or auxiliary "do not start" containers from
the Docker Compose examples.

Each node gets its own StatefulSet, Service, config file, and persistent state
volume. The chart uses `tenzir.yaml` for node configuration, so you do not have
to model ordinary runtime settings as environment variables.

## Install

Use a values file when configuring the node config map. This is the cleanest way
to express `tenzir.yaml` in Helm:

```yaml
tenzir:
  config:
    tenzir:
      platform-control-endpoint: wss://example.invalid

nodes:
  - name: node-a
    token:
      existingSecret: tenzir-node-a-token
```

```sh
kubectl create secret generic tenzir-node-a-token \
  --from-literal=TENZIR_TOKEN=tnz_replace_me

helm install tenzir-node . -f node-a.yaml
```

For quick testing, you can inline the token in values:

```yaml
nodes:
  - name: node-a
    token:
      value: tnz_replace_me
```

## Multiple Nodes

Configure one entry per pre-provisioned platform node:

```yaml
tenzir:
  config:
    tenzir:
      platform-control-endpoint: wss://example.invalid

nodes:
  - name: node-a
    token:
      existingSecret: tenzir-node-a-token
  - name: node-b
    token:
      existingSecret: tenzir-node-b-token
```

Do not reuse one static `tnz_...` node token across multiple nodes. A static
node token identifies one pre-provisioned node identity.

## Configuration

The chart renders a `tenzir.yaml` per node and passes it to `tenzir-node` via
`--config=/etc/tenzir/tenzir.yaml`.

The generated file starts with these defaults:

```yaml
tenzir:
  endpoint: 0.0.0.0:5158
  file-verbosity: quiet
  console-sink: stderr
```

Then it merges in:
1. `tenzir.config`
2. `nodes[].config`

> Anything you put under `tenzir.config` or `nodes[].config` lands in a
> ConfigMap in plaintext. Do not put secrets there. Use `nodes[].token` for
> the node token, and `extraEnv` with a `valueFrom.secretKeyRef` for other
> sensitive values.

The most important values are:

| Value | Default | Description |
| --- | --- | --- |
| `nodes` | one `default` node | Static Tenzir nodes to deploy. |
| `nodes[].name` | required | Node name used in Kubernetes resource names and labels. |
| `nodes[].token.value` | `""` | Token used to create a per-node Secret for `TENZIR_TOKEN`. Mutually exclusive with `existingSecret`. |
| `nodes[].token.existingSecret` | `""` | Existing Secret that contains the node token. Mutually exclusive with `value`. |
| `nodes[].token.key` | `TENZIR_TOKEN` | Data key inside the Secret (existing or chart-created). |
| `nodes[].config` | `{}` | Per-node `tenzir.yaml` overlay. |
| `nodes[].extraPorts` | `[]` | Extra container ports on this one node. See [Ports](#ports). |
| `sharedServices` | `[]` | Fleet-wide Services that load-balance a port across multiple nodes. See [Ports](#ports). |
| `podDisruptionBudget` | `{}` | When `minAvailable` or `maxUnavailable` is set, renders a single PDB across all node pods. Empty = no PDB. |
| `tenzir.config` | `{}` | Global `tenzir.yaml` overlay. |
| `image.registry` | `docker.io` | Image registry, prepended to `repository`. Set `""` to drop it. |
| `image.repository` | `tenzir/tenzir` | Container image repository. |
| `image.tag` | `latest` | Container image tag. **Production deployments should override this with a pinned version — see [Pinning the image](#pinning-the-image).** |
| `containerSecurityContext` | safe defaults | Container-level `securityContext` block. See [Hardening](#hardening). |
| `networkPolicy.enabled` | `false` | When true, render a single NetworkPolicy across the node pods. |
| `persistence.enabled` | `true` | Persist `/var/lib/tenzir`. |
| `persistence.size` | `10Gi` | Per-node state volume size. |

Example of adding a node-specific configuration block:

```yaml
tenzir:
  config:
    tenzir:
      plugins:
        platform:
          skip-peer-verification: false

nodes:
  - name: node-a
    config:
      tenzir:
        secrets:
          api-key: some-secret-name
```

## Persistence

The chart creates a StatefulSet per node. With persistence enabled, each node
receives its own PersistentVolumeClaim for `/var/lib/tenzir`.

The chart writes logs to stderr (`file-verbosity: quiet`, `console-sink: stderr`)
and relies on the cluster's log pipeline (kubelet → CRI → your log backend) to
collect them. There is intentionally no persistent log volume.

## Ports

The chart always exposes the node port (default `5158`) on each node's main
`Service` and headless `Service`. Additional ports can be opened in two ways
that answer different questions.

### `nodes[].extraPorts` — one node owns the port

Adds a `containerPort` to that node's pod and a port entry on that node's
per-node `Service` (and headless Service for direct pod DNS). Use when one
specific node runs the listener.

```yaml
nodes:
  - name: node-a
    token: { existingSecret: tenzir-node-a-token }
    extraPorts:
      - name: http
        containerPort: 8080
        # servicePort: 8080         # defaults to containerPort
        # protocol: TCP             # default TCP
      - name: syslog
        containerPort: 514
        serviceType: LoadBalancer   # publishes via a dedicated Service
        annotations:
          external-dns.alpha.kubernetes.io/hostname: node-a.example.invalid
```

With `serviceType` set, the chart creates a dedicated `Service` named
`<release>-tenzir-node-<node>-<portName>` whose `type` is whatever you picked.
Without `serviceType`, the port is added to the node's main `Service` (which
inherits the global `service.type`, default `ClusterIP`).

### `sharedServices` — one port, load-balanced across many nodes

Creates **one additional `Service`** whose endpoints span every selected node's
pod. `kube-proxy` round-robins across them, and `type: LoadBalancer` gives one
external IP for the fleet. The chart also opens the corresponding container
port on every selected pod so the traffic has somewhere to land.

```yaml
nodes:
  - name: ingester-a
    token: { existingSecret: tenzir-ingester-a-token }
  - name: ingester-b
    token: { existingSecret: tenzir-ingester-b-token }
  - name: standby
    token: { existingSecret: tenzir-standby-token }

sharedServices:
  - name: http-ingest
    port: 8080
    type: LoadBalancer
    nodes: [ingester-a, ingester-b]   # omit (or "all") to target every node
  - name: syslog
    port: 514
    protocol: UDP
    # type defaults to ClusterIP; no `nodes` ⇒ all nodes
```

Selected pods are labeled `tenzir.io/shared-<svc-name>: "true"` and the
generated `Service` selects on that label. You are still responsible for
running a pipeline on each selected node that actually listens on the port
(for example, `load_tcp "0.0.0.0:8080" { … }` under `tenzir.pipelines`).

### Combining the two

Both modes can coexist on the same node and on the same port number. If a
`nodes[].extraPorts` entry and a `sharedServices` entry resolve to the same
`(containerPort, protocol)` on a pod, the container port is opened once
(named after the `extraPorts` entry).

## Pinning the image

The chart defaults to `image.tag: latest`. That gets you the newest Tenzir
release with zero maintenance, but it has two downsides for production
deployments:

- `helm upgrade` (or even an unrelated rollout that re-pulls the image) can
  silently bring in a newer Tenzir version. Rolling back means finding the
  digest you ran before and pinning to it.
- Trivy flags `latest` as a misconfiguration (KSV-0013).

For anything beyond evaluation, pin the tag to a specific release in your
values file:

```yaml
image:
  tag: v6.2.0   # see https://docs.tenzir.com/changelog/tenzir/ for releases
```

Upgrade by bumping the tag in your values and running `helm upgrade`. That
makes the version change an explicit, reviewable diff and gives you an
obvious target to roll back to.

## Hardening

The chart's default `containerSecurityContext` clears most Trivy
container-hardening findings out of the box:

- `runAsNonRoot: true` + `runAsUser: 999` + `runAsGroup: 999` — matches the
  Tenzir image's `tenzir` user (uid 999). The numeric pin is required for
  the `runAsNonRoot` admission check.
- `seccompProfile: { type: RuntimeDefault }`
- `allowPrivilegeEscalation: false`
- `capabilities.drop: [ALL]` — validated against tenzir-node v6.2.0; does not
  break RPC, platform link, `accept_http` on unprivileged ports, or
  `import`/`export`.

Whether you need `CAP_NET_BIND_SERVICE` for low-numbered listener ports is
a property of the cluster's `net.ipv4.ip_unprivileged_port_start` sysctl,
not the port number alone:

- **0** (Docker Desktop, kind, k3s, most managed-k8s distros) — every port
  is unprivileged from the kernel's perspective. `accept_tcp "0.0.0.0:514"`
  works under `drop: [ALL]` without any capability. Verified live.
- **1024** (kernel legacy default) — binding < 1024 needs the capability.
  Check with `cat /proc/sys/net/ipv4/ip_unprivileged_port_start` from a
  pod. If you hit that case, add it back:
  ```yaml
  containerSecurityContext:
    capabilities:
      add: [NET_BIND_SERVICE]
  ```

The chart also defaults `readOnlyRootFilesystem: true` and mounts an
`emptyDir` at `/tmp` so the root filesystem stays read-only while normal
temp-file operations keep working. `/var/lib/tenzir` is writable via the
PVC. Pipelines that try to `to_file` outside those two trees fail with a
clean `IOError` and do not crash the node. Add `extraVolumes` /
`extraVolumeMounts` if you need another writable path.

## ConfigMap-driven restarts

The pod template carries `checksum/config: <sha256>` derived from the merged
per-node config. A `helm upgrade` that changes `tenzir.config` or
`nodes[i].config` flips the checksum for exactly the affected node(s), which
triggers the StatefulSet controller to roll that pod. Nodes whose config did
not change keep running.

## NetworkPolicy

Set `networkPolicy.enabled: true` to render a single NetworkPolicy that
selects every Tenzir node pod in the release. With no other overrides it
allows ingress from any pod in the same namespace. Use `ingress` to declare
additional rules (namespace selectors, pod selectors, port restrictions). Set
`allowSameNamespace: false` to drop the default rule and rely entirely on
`ingress`.

## Run pipelines as code

Anything under `nodes[].config.tenzir.pipelines:` is rendered into that
node's `tenzir.yaml` and started by `tenzir-node` at boot. The pipeline
survives pod restarts because it lives in the per-node ConfigMap.

To make every node accept HTTP POSTs on port 8080 and write the request
body into local Tenzir storage:

```yaml
sharedServices:
  - name: http-ingest
    port: 8080
    type: ClusterIP   # use LoadBalancer to expose externally

nodes:
  - name: node-a
    token: { existingSecret: tenzir-node-a-token }
    config:
      tenzir:
        pipelines:
          ingest-http:
            name: "HTTP ingest"
            definition: |
              accept_http "0.0.0.0:8080" { read_json }
              this = { received_by: "node-a", received_at: now(), body: this }
              import
            restart-on-error: 1 minute

  - name: node-b
    token: { existingSecret: tenzir-node-b-token }
    config:
      tenzir:
        pipelines:
          ingest-http:
            name: "HTTP ingest"
            definition: |
              accept_http "0.0.0.0:8080" { read_json }
              this = { received_by: "node-b", received_at: now(), body: this }
              import
            restart-on-error: 1 minute
```

`sharedServices` automatically opens containerPort 8080 on both pods and
fronts them with a single `Service`. `kube-proxy` load-balances incoming
requests across the pods. Each node enriches the event with its own name
before `import`, so a later `tenzir 'export | where received_by == ...'`
inside a pod shows what landed locally.

See [Ports](#ports) for one-node-only ingest via `nodes[].extraPorts`.

## Escape Hatches

If you need an unusual setting that is not worth modeling in `tenzir.config`,
use `extraEnv` as a last resort. The chart still prefers `tenzir.yaml` for
ordinary configuration.

## Ephemeral Nodes

Ephemeral workspace-token deployments (`wsk_...`) are intentionally not modeled
in this initial chart. They may be added later as a separate mode for disposable
or autoscaled nodes.
