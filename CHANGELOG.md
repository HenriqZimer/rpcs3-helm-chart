# Changelog

All notable changes to this Helm chart will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

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
