# Security Policy

## Supported Versions

We release patches for security vulnerabilities for the following versions:

| Version | Supported          |
| ------- | ------------------ |
| 1.0.x   | :white_check_mark: |

## Reporting a Vulnerability

If you discover a security vulnerability within this Helm chart, please send an email to the maintainer. All security vulnerabilities will be promptly addressed.

**Please do not report security vulnerabilities through public GitHub issues.**

When reporting a vulnerability, please include:

- Description of the vulnerability
- Steps to reproduce
- Potential impact
- Suggested fix (if any)

We will acknowledge your email within 48 hours and send a more detailed response within 5 days indicating the next steps in handling your report.

## Security Best Practices

When deploying RPCS3 using this Helm chart:

1. **Web UI Access**
   - The chart exposes a full KasmVNC desktop with no authentication by default. Either enable
     `env.KASMVNC_ENABLE_BASIC_AUTH` with `env.CUSTOM_USER`/`env.PASSWORD`, or restrict access at
     the network layer (ingress auth, NetworkPolicy, VPN-only).
   - Prefer `extraEnv`/`envFrom` backed by a Kubernetes Secret over plaintext `env.PASSWORD` in
     `values.yaml`.

2. **Network Security**
   - Use TLS/SSL for ingress (enable cert-manager integration) if exposing outside the cluster.
   - Configure network policies to restrict pod communication.

3. **Resource Limits**
   - Set appropriate resource limits and requests.
   - Enable pod security policies/standards where your cluster supports them.

4. **Updates**
   - Keep the chart updated to the latest version.
   - Monitor security advisories for the upstream [linuxserver/docker-rpcs3](https://github.com/linuxserver/docker-rpcs3) image.
   - Regularly update Kubernetes and Helm.

5. **Persistent Data**
   - The `/config` volume can contain BIOS files and save data; back it up and restrict access
     to it like any other sensitive volume.

## Security Features

This chart includes:

- Configurable `securityContext` and `podSecurityContext`
- Optional `serviceAccount` with no default permissions
- Resource limits and requests
- Readiness and liveness probes
