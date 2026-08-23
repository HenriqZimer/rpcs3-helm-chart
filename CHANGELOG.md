# Changelog

All notable changes to this Helm chart will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.3.3] - 2026-08-23

### Changed
- `livenessProbe.initialDelaySeconds` 90s->120s and `failureThreshold` explicitly set to `6` (was the implicit default `3`) - confirmed in production that under heavy I/O contention from a shared NFS-backed PVC (several emulator pods cold-booting simultaneously after a node reboot), the old 150s total liveness tolerance wasn't enough for DOCKER_MODS's ephemeral apt install + KasmVNC desktop startup to finish; kubelet kept killing the container mid-boot, and each kill wiped the ephemeral DOCKER_MODS progress, creating a self-sustaining restart loop that never converged even as node load dropped. New tolerance: 120 + 6*20 = 240s.

## [1.3.2] - 2026-08-23

### Changed
- `readinessProbe.initialDelaySeconds` 10s->30s, `livenessProbe.initialDelaySeconds` 30s->90s, and `resources.requests.cpu` 500m->1 - confirmed in production that under CPU contention on a shared GPU node, the old tight timing killed containers mid-boot (KasmVNC desktop hadn't finished starting) causing a self-sustaining restart storm; the higher CPU request also gives the scheduler an honest signal instead of letting far more pods pack onto a node than it can actually sustain under load.

## [1.2.0] - 2026-08-21

### Added
- `probes.enabled` (default `true`). Set to `false` to disable the readiness/liveness probe and the `http`/`https` `containerPort` declarations. Needed when running multiple `sunshine.hostNetwork` pods on the same node: the Selkies nginx (3000/3001) is hardcoded and can't be moved, so any pod beyond the first would otherwise crashloop or block scheduling entirely even though Sunshine itself works fine. Tradeoff: the web UI (KasmVNC) becomes unreachable on that pod - Moonlight is unaffected.

## [1.1.1] - 2026-08-21

### Fixed
- `sunshine.*` now also mounts `/dev/uhid` - without it, Sunshine falls back to emulating gamepads as a generic Xbox One controller when the client reports a PlayStation-type pad (DualShock/DualSense), and that fallback does not correctly forward face buttons/triggers (confirmed: analog stick axes work, buttons don't). The real virtual DualShock/DualSense device needs kernel `uhid` access to be created.

## [1.1.0] - 2026-08-21

### Added
- `sunshine.*` (POC) - Sunshine/Moonlight low-latency game streaming as an alternative to the
  default Selkies/KasmVNC streaming. Requires a Docker Mod
  (`ghcr.io/henriqzimer/rpcs3-sunshine-mod`) that installs Sunshine and swaps the base image's
  Xvfb for a real Xorg + "dummy" driver - Xvfb in this image isn't linked against `libudev`, so
  it can never hotplug the mouse/keyboard/gamepad devices Sunshine creates via `/dev/uinput`.
  Also requires `sunshine.hostNetwork`/`sunshine.privileged` (Moonlight speaks raw TCP+UDP that
  can't go through the Ingress, and the pod needs the node's network namespace for uinput hotplug
  uevents to reach udev, plus a device cgroup rule Kubernetes has no native knob for) and
  `PIXELFLUX_WAYLAND=false` in `env` (the image defaults to a Wayland compositor, which Sunshine
  can't capture).

## [1.0.5] - 2026-08-17

### Fixed
- README install commands now use a unique repo alias (`rpcs3-helm-chart`) instead of the bare chart name, and pin `--version` explicitly.

## [1.0.4] - 2026-08-17

### Changed
- README now shows the emulator's logo at the top.

## [1.0.3] - 2026-08-17

### Added
- Chart releases are now GPG-signed (`helm package --sign`) - see `artifacthub.io/signKey` in `Chart.yaml` for the public key URL and fingerprint. Powers the "Signed" badge on ArtifactHub.

## [1.0.2] - 2026-08-16

### Added
- `chart/values.schema.json` validating `values.yaml` - powers the "Values schema" feature on ArtifactHub, previously absent since no chart in this project ever had one.

## [1.0.1] - 2026-08-16

### Added
- `icon` in Chart.yaml, pointing at linuxserver.io's own logo image for this app - was missing entirely before, which is why no image ever showed up on ArtifactHub for this chart.

## [1.0.0] - 2026-08-13

### Added
- Initial release of the RPCS3 Helm chart
- Deployment, Service, optional Ingress and PVC for the linuxserver.io RPCS3 KasmVNC webtop image
- Configurable `serviceAccount` (create/name/annotations)
- Readiness and liveness probes (TCP check on the KasmVNC HTTP port)
- GPU passthrough for Intel/AMD (VA-API via `/dev/dri`) and NVIDIA (via the NVIDIA Kubernetes
  device plugin, `gpu.vendor: nvidia`)
- `seccompUnconfined` and a `2Gi` default `shmSize`, carried over from this chart's sibling
  pcsx2-helm-chart after PCSX2's JIT recompiler was found to crash with `SIGBUS` under Docker/
  Kubernetes' default seccomp profile / small `/dev/shm` — not independently confirmed on RPCS3,
  but the same base image/toolchain makes it a likely fix if you hit the same crash
- `extraVolumes`/`extraVolumeMounts` for mounting an existing ROMs/firmware library
- `streaming.enabled`/`streaming.brokerPort` to expose the RomM emulator-streaming broker sidecar
  port. **The `rpcs3-romm-integration` mod has never been merged/released** — only an unreleased
  `:dev` image tag exists (`ghcr.io/loneangelfayt/rpcs3-romm-integration-mod:dev`), published from
  an unmerged branch. Expect instability.
