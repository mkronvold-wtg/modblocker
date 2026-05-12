# Changelog

## Unreleased

- Bootstrap repository structure for the kernel module mitigation project.
- Add Kubernetes manifests that blacklist and unload `algif_aead` on immutable nodes.
- Extend the DaemonSet with Dirty Frag mitigations for `esp4`, `esp6`, and `rxrpc`, plus cache-drop remediation during enforcement.
- Rename the managed host modprobe file to `/etc/modprobe.d/modblocker.conf` so the on-node state matches the broader mitigation scope.
- Replace the passive second container with a monitoring sidecar that continuously validates the host config and module state, then exposes readiness and liveness health signals.
- Add explicit hardening for the steady-state monitor container and pod defaults, while documenting the privileged init container as a narrow required exception.
- Change the default deployment namespace from `kube-system` to `modblocker`.
- Add `docs/SECURE.md` to document container hardening and residual risk areas.
- Add a `no-monitor` overlay so the compliance sidecar is optional when operators prefer lower steady-state host exposure over ongoing drift visibility.
