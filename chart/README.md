# RPCS3 Helm Chart

[![Version: 1.0.3](https://img.shields.io/badge/Version-1.0.3-informational?style=flat-square)](https://github.com/HenriqZimer/rpcs3-helm-chart)
[![AppVersion: latest](https://img.shields.io/badge/AppVersion-latest-informational?style=flat-square)](https://docs.linuxserver.io/images/docker-rpcs3/)

A Helm chart for [RPCS3](https://docs.linuxserver.io/images/docker-rpcs3/) - the linuxserver.io
PlayStation 3 emulator, served as a full desktop over the browser via KasmVNC.

## TL;DR

```bash
# Add the Helm repository
helm repo add rpcs3 https://henriqzimer.github.io/rpcs3-helm-chart
helm repo update

# Install RPCS3
helm install rpcs3 rpcs3/rpcs3
```

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- A default `StorageClass` available in the cluster if you enable persistence (or your own
  `storageClass`/`existingClaim` values)
- Ingress controller (optional, only if `ingress.enabled: true`)
- cert-manager (optional, for automatic TLS certificates)
- A node exposing `/dev/dri` (`gpu.vendor: intel-amd`) or the [NVIDIA device plugin](https://github.com/NVIDIA/k8s-device-plugin)
  (`gpu.vendor: nvidia`) if you enable `gpu.enabled`

## Installing the Chart

### From Helm Repository

```bash
helm repo add rpcs3 https://henriqzimer.github.io/rpcs3-helm-chart
helm repo update

helm install rpcs3 rpcs3/rpcs3
```

### From Source

```bash
git clone https://github.com/HenriqZimer/rpcs3-helm-chart.git
cd rpcs3-helm-chart

helm package chart/
helm install rpcs3 ./rpcs3-1.0.3.tgz
```

## Configuration

### Persistence

By default `persistence.config.enabled` is `false`, so `/config` (app settings, BIOS, save
states) is an ephemeral `emptyDir` — everything is lost when the pod restarts. For anything but a
quick test, enable it:

```yaml
persistence:
  config:
    enabled: true
    type: pvc # pvc | hostPath | emptyDir
    storageClass: "" # leave empty to use the cluster default
    size: 10Gi
```

### ROMs / BIOS library

The chart does not manage a ROMs volume directly. Use `extraVolumes`/`extraVolumeMounts` to
mount an existing PVC, NFS share, or hostPath:

```yaml
extraVolumes:
  - name: roms
    persistentVolumeClaim:
      claimName: my-roms-library
extraVolumeMounts:
  - name: roms
    mountPath: /roms
```

### Ingress

Disabled by default. Reach the UI via `kubectl port-forward` for quick testing, or:

```yaml
ingress:
  enabled: true
  className: "traefik"
  hosts:
    - host: rpcs3.example.com
      paths:
        - path: /
          pathType: Prefix
```

### Access control

The KasmVNC desktop has no authentication by default. Either restrict access at the network
layer (ingress auth, NetworkPolicy, VPN-only), or set:

```yaml
env:
  KASMVNC_ENABLE_BASIC_AUTH: "true"
  CUSTOM_USER: "myuser"
  PASSWORD: "changeme" # prefer envFrom with a Secret instead
```

### RPCS3 crashes when launching a game (SIGBUS)

If the dashboard/UI works but a game crashes on boot (check the container logs for
`Unhandled SIGBUS`), RPCS3's PPU/SPU JIT recompilers may need more `/dev/shm` than the small
default — this is confirmed for this chart's sibling, pcsx2-helm-chart (PCSX2's EE recompiler
uses a shared-memory-backed "fastmem" mmap trick and crashes with SIGBUS right after a game
starts when `/dev/shm` is too small); not independently confirmed on RPCS3, but worth trying
first since the base image/toolchain is the same:

```yaml
shmSize: 2Gi # chart default; raise further if it still crashes
```

If that alone doesn't fix it, the JIT may also be hitting Docker/Kubernetes' default seccomp
profile (blocks syscalls it needs to allocate executable memory on some kernel/libseccomp
combinations):

```yaml
seccompUnconfined: true
```

### GPU / hardware acceleration

#### Intel/AMD (VA-API)

```yaml
gpu:
  enabled: true
  vendor: intel-amd
  devicePath: /dev/dri
  supplementalGroups: [44, 109] # host's video/render group GIDs; check with `getent group video render`
```

#### NVIDIA

Requires the [NVIDIA device plugin](https://github.com/NVIDIA/k8s-device-plugin) already installed on the
cluster (this chart only requests the resource, it doesn't install the plugin):

```yaml
gpu:
  enabled: true
  vendor: nvidia
  nvidia:
    count: 1
```

This sets `resources.limits."nvidia.com/gpu"` and the `NVIDIA_VISIBLE_DEVICES`/`NVIDIA_DRIVER_CAPABILITIES`
env vars — it does **not** need `devicePath`/`supplementalGroups`, the device plugin injects `/dev/nvidia*`
itself.

Two node-level prerequisites this chart cannot set for you (see the
[linuxserver.io GPU Configuration docs](https://docs.linuxserver.io/images/docker-rpcs3/#gpu-configuration)):

- The node needs `nvidia-drm.modeset=1 nvidia_drm.fbdev=1` on its kernel command line (e.g. via GRUB), or the
  compositor's own screen buffer allocation fails with `DRM_IOCTL_MODE_CREATE_DUMB failed: Permission denied`
  and the desktop renders solid black even though the container itself looks healthy.
- On a genuinely headless node (no monitor ever attached to that GPU), DRM may still need a physical HDMI/DP
  dummy plug in the port for the driver to initialize a display at all.

If RPCS3 itself crashes with `Unhandled SIGBUS` in the container logs on boot (i.e. the dashboard works
but launching a game doesn't), see `shmSize` below and `seccompUnconfined` above — try those first, it's
usually not a GPU problem.

### RomM emulator streaming

This chart can also run as a target for RomM's [Emulator Streaming](https://docs.romm.app/latest/using/emulator-streaming/)
feature, which launches ROMs directly inside this container and streams them
back to the browser (as opposed to in-browser emulation). That feature is
provided by a third-party [Docker Mod](https://github.com/LoneAngelFayt/rpcs3-romm-integration)
(not maintained by this chart or by linuxserver.io/RomM) — this chart only
adds the plumbing to expose its broker port.

> [!WARNING]
> Unlike the sibling PCSX2/Dolphin/xemu/Eden charts, the RPCS3 integration has **never been merged
> or released** — it only exists on an unmerged `dev` branch, published as the `:dev` image tag
> (there is no `:latest`). Expect breaking changes and rough edges.

PS3 also needs firmware installed before any game will run: open the container's own desktop
(`service.httpsPort`) once and go to **Configuration → Install Firmware**, using a `PS3UPDAT.PUP`
you obtain yourself from PlayStation's system software page — this chart/mod can't fetch that for
you. ROMs must be **decrypted** content (`.iso`, or a JB/HDD-dump folder, optionally archived as
`.zip`/`.7z`/`.rar`) — a straight disc rip is NPDRM-encrypted and won't boot.

```yaml
env:
  DOCKER_MODS: "ghcr.io/loneangelfayt/rpcs3-romm-integration-mod:dev" # no :latest tag exists yet
  ROM_ROOT: "/romm/library" # must match the mountPath below
  CACHE_DIR: "/config/rpcs3-cache" # extracted .zip/.7z/.rar archives are cached here
  CACHE_MAX_GB: "150" # 0 = unlimited

envFrom:
  - secretRef:
      name: streaming-broker-secret # must provide BROKER_SECRET

streaming:
  enabled: true # exposes containerPort/servicePort "broker" (8000)

# Mount the same ROM library RomM uses, at the same path RomM sees it at.
extraVolumes:
  - name: roms
    persistentVolumeClaim:
      claimName: romm-library-truenas
extraVolumeMounts:
  - name: roms
    mountPath: /romm/library
```

Then in RomM's `config.yml`, `broker_host` and `host` mean different things
and are reached from different places — mixing them up is the most common
way to get this stuck:

- `broker_host` is called **server-to-server**, by RomM's own backend. Point
  it at this Service's ClusterIP on the broker port, e.g.
  `http://rpcs3.<namespace>.svc.cluster.local:8000` — cluster-internal DNS
  is fine, nothing outside the cluster ever needs to reach it. Keep it off
  any externally-reachable Ingress/LoadBalancer.
- `host` is opened **directly by the end user's browser**, so it must be an
  address that specific browser can reach and gets HTTPS from — cluster-
  internal DNS (`*.svc.cluster.local`) will not resolve there, and a bare
  ClusterIP won't either. Two ways to satisfy that:
  - Set `ingress.enabled: true` (pointing at `service.httpPort`, since the
    Ingress controller terminates TLS) and use a real certificate — this
    also sidesteps the next point entirely.
  - Reach the container's own KasmVNC HTTPS port directly (`service.httpsPort`,
    via a LoadBalancer Service or NodePort). Its self-signed cert has
    `CN=*` with **no Subject Alternative Name**, which Chrome/Firefox will
    let you click through but which some browsers (notably Safari) refuse
    outright instead of showing the usual "unsafe site" warning — if `host`
    loads fine in Chrome but fails immediately in Safari with a vague
    "might be down or moved" error, this is why.

See the [Configuration File reference](https://docs.romm.app/latest/reference/configuration-file/#streaming)
for the full `streaming` schema.

## Values

| Key | Default | Description |
|---|---|---|
| `replicaCount` | `1` | Number of replicas. Must stay at 1 — no session clustering. |
| `image.repository` | `lscr.io/linuxserver/rpcs3` | Container image. |
| `image.tag` | `latest` | Image tag. |
| `env.PUID` / `env.PGID` | `1000` / `1000` | User/group the app runs as; match your storage's ownership. |
| `env.TZ` | `Etc/UTC` | Timezone. |
| `env.KASMVNC_ENABLE_BASIC_AUTH` | `false` | Require basic auth on the web UI. |
| `service.httpPort` / `service.httpsPort` | `3000` / `3001` | KasmVNC ports. |
| `streaming.enabled` | `false` | Expose the RomM streaming broker port (see above). |
| `streaming.brokerPort` | `8000` | Broker port to expose, must match the mod's `BROKER_PORT`. |
| `ingress.enabled` | `false` | Expose the HTTP port via Ingress. |
| `persistence.config.enabled` | `false` | Persist `/config`. `false` uses an ephemeral emptyDir. |
| `extraVolumes` / `extraVolumeMounts` | `[]` | Raw volume/volumeMount entries, e.g. a ROMs library. |
| `shmSize` | `2Gi` | Size of the memory-backed `/dev/shm` mount (KasmVNC and RPCS3's JIT both need this). |
| `gpu.enabled` / `gpu.vendor` | `false` / `intel-amd` | Enable GPU passthrough; `intel-amd` (VA-API, `/dev/dri`) or `nvidia` (device plugin, `nvidia.com/gpu`). |
| `seccompUnconfined` | `false` | Disable the default seccomp profile if RPCS3's JIT crashes with SIGBUS. |
| `resources` | `500m/1Gi` requests, `2/4Gi` limits | Container resources. |
| `serviceAccount.create` | `false` | Create a dedicated ServiceAccount. |

See [values.yaml](values.yaml) for the full, commented list.

## Uninstalling the Chart

```bash
helm uninstall rpcs3
```

This does not delete PVCs created by the chart; remove them manually if you no longer need the
data.

## License

MIT — see [LICENSE](../LICENSE).
