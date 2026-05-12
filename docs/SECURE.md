# Security posture

This document explains the hardening and remaining risk areas for each container in the `modblocker` DaemonSet.

## Pod-level controls

These settings apply to the pod as a whole:

- `automountServiceAccountToken: false`: the pod does not receive a Kubernetes API token.
- `enableServiceLinks: false`: Kubernetes service environment variables are not injected.
- `hostPID: true`: both containers can observe the host PID namespace.
- `terminationGracePeriodSeconds: 10`: limits how long shutdown can hang.
- `hostPath` mount of `/`: required so the workload can inspect and modify the host filesystem.

## Init container: `disable-vulnerable-modules`

### Purpose

This container performs the one-time host remediation:

- writes `/host/etc/modprobe.d/modblocker.conf`
- unloads `algif_aead`, `esp4`, `esp6`, and `rxrpc`
- runs the Dirty Frag cache-drop step when needed

### Hardening in place

- `readOnlyRootFilesystem: true`
- explicit resource requests and limits, including `ephemeral-storage`
- no Kubernetes API token
- no service link injection

### Required elevated privileges

This container is intentionally a privileged exception:

- `privileged: true`
- `allowPrivilegeEscalation: true`
- `runAsUser: 0`
- `runAsGroup: 0`
- `runAsNonRoot: false`
- `seccompProfile.type: Unconfined`
- read-write host root mounted at `/host`

### Why the exception exists

The remediation needs host-level access that a standard restricted profile would block:

- writing to host `/etc/modprobe.d`
- unloading host kernel modules
- writing `3` to host `/proc/sys/vm/drop_caches`
- invoking host tools through `chroot /host`

### Risk areas

- **Host compromise blast radius:** if this container is exploited, it effectively has host-root-equivalent access.
- **Host filesystem modification:** it can change files beyond the intended config path if the script or image is compromised.
- **Kernel interaction:** unloading modules and writing to `/proc/sys` directly affects node behavior and stability.
- **Admission-policy sensitivity:** clusters that disallow privileged containers, unconfined seccomp, or `hostPath` mounts will reject it.

## Steady-state container: `monitor-state`

### Purpose

This container continuously checks that:

- `/host/etc/modprobe.d/modblocker.conf` still contains the required rules
- the target modules are absent from `/proc/modules`
- readiness and liveness status files are current

### Hardening in place

- `privileged: false`
- `allowPrivilegeEscalation: false`
- `runAsNonRoot: true`
- `runAsUser: 65532`
- `runAsGroup: 65532`
- `capabilities.drop: [ALL]`
- `readOnlyRootFilesystem: true`
- `seccompProfile.type: RuntimeDefault`
- host root mounted read-only at `/host`
- status written only to an internal `emptyDir`
- explicit resource requests and limits, including `ephemeral-storage`
- no Kubernetes API token
- no service link injection

### Residual risk areas

- **Host visibility:** `hostPID: true` and the read-only host mount still expose host state to the container.
- **Information disclosure:** an attacker with code execution in the sidecar could inspect host files readable through `/host` and host process metadata visible through the PID namespace.
- **Compliance-only role:** it can detect drift and fail readiness, but it cannot repair drift by itself.
- **Image trust:** it still depends on the integrity of the BusyBox image and the monitoring script.

## Shared operational risks

- **Node compatibility:** disabling `esp4` and `esp6` affects kernel ESP/IPsec support; disabling `rxrpc` affects RxRPC consumers.
- **Drift after startup:** the monitor makes the pod unready if drift happens later, but it does not automatically re-run the privileged remediation.
- **Host OS assumptions:** the design assumes the VKS node image allows the required host mount and writes to the relevant host paths.

## Practical interpretation

- The **init container** is the dangerous part by necessity.
- The **monitor sidecar** is intentionally constrained, but not harmless, because it still has host visibility.
- The main security boundary for this design is operational trust in the manifest, the image source, and the cluster policy that allows this pod to exist at all.
