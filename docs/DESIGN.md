# Design

## Problem

VKS worker nodes are rebuilt from immutable images, so manual host edits do not persist through node replacement, upgrades, or remediation. A CopyFail mitigation that depends on disabling `algif_aead` therefore has to be applied automatically on every node.

## Goals

- Disable the `algif_aead` kernel module on each Linux worker node.
- Persist the disablement across node churn by reapplying it through Kubernetes.
- Keep the implementation small, auditable, and easy to apply.

## Non-goals

- Rebuilding node images or creating a custom TKR.
- Managing broader kernel-hardening policy.
- Handling Windows nodes.

## Chosen approach

The repository ships a privileged DaemonSet. On each node, an init container mounts the host filesystem, writes an idempotent modprobe configuration file into `/etc/modprobe.d`, and attempts to unload `algif_aead` if the module is currently present. A pause container keeps the pod scheduled so the DaemonSet remains healthy and gets recreated automatically on new nodes.

## Key design decisions

### Host writes are explicit and minimal

The DaemonSet writes only one managed file on the host:

- `/etc/modprobe.d/disable-copyfail.conf`

That file contains:

- `blacklist algif_aead`
- `install algif_aead /bin/false`

This combination blocks normal autoloading and makes explicit loads fail.

### Reconciliation is idempotent

The init container checks whether each required line already exists before appending it. Re-running the pod does not duplicate entries.

### Module unload uses host tooling

The container uses `chroot /host` to execute the host shell and host `modprobe`, which avoids depending on kernel-module tooling inside the container image.

### Scope stays namespaced

The implementation uses a namespaced DaemonSet in `kube-system` and does not require cluster-wide RBAC because it does not call the Kubernetes API.
