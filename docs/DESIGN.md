# Design

## Problem

VKS worker nodes are rebuilt from immutable images, so manual host edits do not persist through node replacement, upgrades, or remediation. The same node-immunity constraint applies to both the publicly known CopyFail mitigation (`algif_aead`) and the Dirty Frag module mitigations (`esp4`, `esp6`, and `rxrpc`), so the hardening has to be applied automatically on every node.

## Goals

- Disable the `algif_aead`, `esp4`, `esp6`, and `rxrpc` kernel modules on each Linux worker node.
- Persist the disablement across node churn by reapplying it through Kubernetes.
- Run the Dirty Frag cache-drop cleanup when the DaemonSet first applies or unloads the Dirty Frag modules.
- Keep the implementation small, auditable, and easy to apply.

## Non-goals

- Rebuilding node images or creating a custom TKR.
- Managing broader kernel-hardening policy.
- Handling Windows nodes.

## Chosen approach

The repository ships a privileged DaemonSet. On each node, an init container mounts the host filesystem, writes an idempotent modprobe configuration file into `/etc/modprobe.d`, unloads the CopyFail and Dirty Frag modules if they are currently present, and drops the page cache during Dirty Frag remediation. A second container then continuously monitors the host config and live module state so Kubernetes health reflects whether the mitigation is still in effect.

## Key design decisions

### Host writes are explicit and minimal

The DaemonSet writes only one managed file on the host:

- `/etc/modprobe.d/modblocker.conf`

That file contains:

- `blacklist algif_aead`
- `install algif_aead /bin/false`
- `install esp4 /bin/false`
- `install esp6 /bin/false`
- `install rxrpc /bin/false`

This combination blocks normal autoloading and makes explicit loads fail.

### Reconciliation is idempotent

The init container checks whether each required line already exists before appending it. Re-running the pod does not duplicate entries.

### Module unload uses host tooling

The container uses `chroot /host` to execute the host shell and host `modprobe`, which avoids depending on kernel-module tooling inside the container image.

### Dirty Frag cleanup is automatic on remediation

Dirty Frag guidance recommends dropping the page cache after removing the affected modules. The DaemonSet follows that guidance when it first adds the Dirty Frag rules or unloads `esp4`, `esp6`, or `rxrpc`.

### Ongoing health is explicit

The steady-state container is not just a placeholder. It re-checks the managed host config and `/proc/modules` on a loop, writes a health flag to a shared status volume, and lets readiness fail when the node drifts out of compliance. A separate heartbeat file keeps liveness focused on whether the monitor itself is still running.

### Security hardening is explicit

The pod now disables service account token projection and service links because it does not call the Kubernetes API. The monitor sidecar uses an explicit non-privileged profile: `runAsNonRoot`, `allowPrivilegeEscalation: false`, `capabilities.drop: [ALL]`, `readOnlyRootFilesystem: true`, and `seccompProfile: RuntimeDefault`.

The init container is an intentional exception. It stays privileged, runs as root, and uses `seccompProfile: Unconfined` because unloading host kernel modules and dropping caches requires access that a standard hardened profile would block.

### Compatibility is explicit

Disabling `esp4` and `esp6` prevents kernel ESP/IPsec support from loading on the node, and disabling `rxrpc` blocks RxRPC consumers such as AFS-related workloads. Those impacts are intentional tradeoffs of the mitigation.

### Scope stays namespaced

The implementation uses a namespaced DaemonSet in `kube-system` and does not require cluster-wide RBAC because it does not call the Kubernetes API.
