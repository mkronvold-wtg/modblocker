# Changelog

## Unreleased

- Bootstrap repository structure for the kernel module mitigation project.
- Add Kubernetes manifests that blacklist and unload `algif_aead` on immutable nodes.
- Extend the DaemonSet with Dirty Frag mitigations for `esp4`, `esp6`, and `rxrpc`, plus cache-drop remediation during enforcement.
- Rename the managed host modprobe file to `/etc/modprobe.d/modblocker.conf` so the on-node state matches the broader mitigation scope.
